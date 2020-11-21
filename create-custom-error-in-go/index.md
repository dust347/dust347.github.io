# 在 Go 中创建自定义错误

Go 语言使用 error 来表示错误。在开发过程中，错误处理基本上是判断所调用的函数所返回的 error 是否为空，如果不为空进行必要的处理。

但是在一些复杂的业务场景中，仅知道是否有错误是不够的，还需要知道具体的错误类型。这种时候标准库提供的 error 功能就太过简单。这时可以实现自定义的 error 解决。
<!--more-->

error 不是一个具体类型，而是一个接口：

```go
type error interface {
  Error() string
}
```

标准库提供两个函数用于创建 error：`errors.New` 和 `fmt.Errorf`。

在 go 1.13 之前，fmt.Errorf 也仅仅调用了 errors.New。

```go
func New(text string) error {
	return &errorString{text}
}

type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

返回的 error 基本等同于字符串，无法细分错误类型。

如果想要判断细分类型，很容易想到的一个办法是自定义错误类型，在判断错误处理的时候，通过类型断言判断：

```go
type Err1 struct {  // 自定义 error 类型
  s string
}


func handleErr() error {
  e := function()
  if e == nil {
    return nil
  }
  if e1, ok := e.(Err1); ok { //类型断言判断是否是自定义 error
    // 错误处理
  }
}
```

但是存在两个问题：
- 需要多少个错误类型就需要实现多少个 error。调用方判断错误类型的时候需要大量的 if 语句。不好维护。
- 函数调用链路中，中间任何一个函数使用 fmt.Error “包装” error，底层的 error 类型就会丢失；

第一个问题比较好解决。可以在自定义 error 中增加错误类型标识：

```go
type Err2 struct {  // 自定义 error 类型
  s string
  Type int32
}

func (e *Err2) Error() string {
  return e.s
}

func NewErr(typ int32, text string) *Err2 {
  return &Err2 {
    s: text,
    Type: typ,
  }
}


func handleErr() error {
  e := function()
  if e == nil {
    return nil
  }
  if e2, ok := e.(Err1); ok { //类型断言判断是否是自定义 error
    // 错误处理
    switch e2.Type {
      // ……
    }
  }

```

## wrapError 和 errors.As
在 go 1.13 开始，标准库对错误处理进行了小的升级。

 `fmt.Error` 函数新增 `fmt.Errorf("... %w ...", ..., err, ...)` 的写法。当使用 `%w` 作为 verb 时，生成的 error 是这样的：

```go
type wrapError struct {
	msg string
	err error
}

func (e *wrapError) Error() string {
	return e.msg
}

func (e *wrapError) Unwrap() error {
	return e.err
}
```

也就是说，fmt.Errorf 生成的 error 可以嵌套其他的 error 避免在错误处理中，底层 error 的丢失。

而在 errors 包，新增了 `errors.Unwrap`，`errors.Is` 和 `errors.As 函数`。

Unwrap 函数比较简单，如果传入的 error 嵌套了其他 error，返回嵌套的 error，否则返回 nil。
```go
func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	})
	if !ok {
		return nil
	}
	return u.Unwrap()
}
```

`errors.Is` 也很简单，判断传入的 err 和 target（我们想要的 error）是否一致，如果是返回 true，否则不断对 err 调用 Unwrap，直到有明确结果。

```go
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	for {
		if isComparable && err == target {
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true
		}

		if err = Unwrap(err); err == nil {
			return false
		}
	}
}
```

通过这几个函数，基本可以解决 error 在函数调用过程中，底层函数类型丢失的问题。

但是，还是需要实现多个 error 类型满足细分错误类型的需要。这个问题可以使用 `errors.As` 解决：

```go
func As(err error, target interface{}) bool {
	if target == nil {
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
		panic("errors: target must be a non-nil pointer")
	}
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
		if reflectlite.TypeOf(err).AssignableTo(targetType) {
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
			return true
		}
		err = Unwrap(err)
	}
	return false
}
```

`errors.As` 看起来和 `errors.Is` 类似，不同点在于，如果 `errors.As` 返回 true，target 会被修改为匹配的具体 error。

相对的，`errors.As` 要求传入的 target 必须是指针类型，非空，包含 error 方法的接口（接口，不是结构体），来实现修改。

> [pkg/errors](https://github.com/pkg/errors) 是一个很好用的 errors 实现，这个包出现在 go 1.13 以前。代码中自己调用 Uwrap 匹配 error 的。

## 实现自定义的 error
假设在业务场景中，需要给具体的错误定义类型，自定义的 errors 代码大概是这样的：

```go
// 保留errType 定义
const (
	// NoErr 无err
	NoErr = 0
	// UnknowErr 未知err
	UnknowErr = -1
)

// Typer 带有业务错误类型的 error
type Typer interface {
	error
	Type() int32
}

// Error 业务错误
type Error struct {
	typ int32  //错误类型
	msg string //错误信息

	cause error //要包装的原因
}

// New 创建一个新的Error
func New(typ int32, msg string) error {
	return &Error{
		typ: typ,
		msg: msg,
	}
}

// Type 返回错误类型
func (e *Error) Type() int32 {
	if e == nil {
		return NoErr
	}
	return e.typ
}

func (e *Error) Unwrap() error {
	return e.cause
}

// Type 返回error 的业务类型type
func Type(err error) int32 {
	if err == nil { //error 为空认为无错误
		return NoErr
	}

	if e := Typer(nil); errors.As(err, &e) {
		return e.Type()
	}

	return UnknowErr
}
```

完整代码地址：[errors](https://github.com/dust347/errors)

最后，Go 标准库提供的 error 功能还是比较简单，对于复杂的场景，需要自定义 error 满足自己的业务需求。在实现过程中可以使用标准库的函数较少开发量。



