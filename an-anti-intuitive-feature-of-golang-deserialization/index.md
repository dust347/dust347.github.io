# golang json和gob包反序列化一个反直觉的特点


<!--more-->
## 1.概述
在使用golang对二进制反序列化的时候，会将反序列化的结果写入结构体的实例(当然必须能对应上才行)。一般来说，直觉上会认为如果要反序列话的二进制是空的(或者某些字段的值是空的)，反序列化后，对应的结构体(或对应的字段)应该是空。但是实际上，json包和gob包反序列化会用二进制对应的值去覆盖要赋值的结构体原有的值。也就是说，如果反序列化的时候，要写入的结构体的实例本身不为空，同时二进制为空的时候，反序列化并不会将这个结构体的值清空。

## 2.试验代码
```go
package main_test

import (
	"bytes"
	"encoding/gob"
	"encoding/json"
	"testing"
)

//S 用来试验的结构体定义
type S struct {
	A   int      `json:"a,omitempty"`
	Str string   `json:"str,omitempty"`
	B   []string `json:"b,omitempty"`
}

//测试JSON反序列
func TestJSON(t *testing.T) {
	s1 := S{
		A:   1,
		Str: "test",
		B: []string{
			"a",
			"b",
		},
	}

	s2 := S{
		A: 2,
	}

	//序列化
	raw1, err := genJSONRaw(&s1)
	if err != nil {
		t.Fatal(err)
	}

	t.Logf("raw1=%s", raw1)

	raw2, err := genJSONRaw(&s2)
	if err != nil {
		t.Fatal(err)
	}
	//部分字段有值
	t.Logf("raw2=%s", raw2)

	//反序列化

	//当要反序列化的对象，是空的
	t.Log("s 本身为空的情况")
	var s S
	err = decodeJSONRaw(&s, raw1)
	if err != nil {
		t.Fatal(err)
	}

	t.Logf("decode raw1 to s: %+v", s) //正常反序列化

	s = S{}
	err = decodeJSONRaw(&s, raw2)
	if err != nil {
		t.Fatal(err)
	}
	t.Logf("decode raw2 to s: %+v", s) //raw2已有A字段有值

	t.Log("s 本身有值的情况")

	s = S{
		A:   3,
		Str: "test3",
		B: []string{
			"3",
			"4",
		},
	}

	t.Logf("s=%+v", s)

	err = decodeJSONRaw(&s, raw1)
	if err != nil {
		t.Fatal(err)
	}

	t.Logf("decode raw1 to s: %+v", s)

	s = S{
		A:   3,
		Str: "test3",
		B: []string{
			"3",
			"4",
		},
	}

	t.Logf("s=%+v", s)

	err = decodeJSONRaw(&s, raw2)
	if err != nil {
		t.Fatal(err)
	}
	t.Logf("decode raw2 to s: %+v", s) //raw2已有A字段有值
}

func TestGob(t *testing.T) {
	s1 := S{
		A:   1,
		Str: "test",
		B: []string{
			"a",
			"b",
		},
	}

	s2 := S{
		A: 2,
	} //empty

	//序列化
	raw1, err := genGobRaw(&s1)
	if err != nil {
		t.Fatal(err)
	}

	raw2, err := genGobRaw(&s2)
	if err != nil {
		t.Fatal(err)
	}

	//反序列化

	t.Log("s 本身为空的情况")
	var s S
	err = decodeGobRaw(&s, raw1)
	if err != nil {
		t.Fatal(err)
	}

	t.Logf("decode raw1 to s: %+v", s)

	s = S{}
	err = decodeGobRaw(&s, raw2)
	if err != nil {
		t.Fatal(err)
	}
	t.Logf("decode raw2 to s: %+v", s)

	t.Log("s 本身不为空的情况")

	s = S{
		A:   3,
		Str: "test3",
		B: []string{
			"3",
			"4",
		},
	}

	t.Logf("s=%+v", s)

	err = decodeGobRaw(&s, raw1)
	if err != nil {
		t.Fatal(err)
	}

	t.Logf("decode raw1 to s: %+v", s)

	s = S{
		A:   3,
		Str: "test3",
		B: []string{
			"3",
			"4",
		},
	}
	t.Logf("s=%+v", s)

	err = decodeGobRaw(&s, raw2)
	if err != nil {
		t.Fatal(err)
	}
	t.Logf("decode raw2 to s: %+v", s)

}

func genGobRaw(s *S) ([]byte, error) {
	var buf bytes.Buffer
	enc := gob.NewEncoder(&buf)

	err := enc.Encode(&s)
	if err != nil {
		return []byte{}, err
	}

	return buf.Bytes(), nil
}

func decodeGobRaw(s *S, b []byte) error {
	buf := bytes.NewBuffer(b)
	dec := gob.NewDecoder(buf)

	err := dec.Decode(s)
	if err != nil {
		return err
	}

	return nil
}

func genJSONRaw(s *S) ([]byte, error) {
	return json.Marshal(s)
}

func decodeJSONRaw(s *S, b []byte) error {
	return json.Unmarshal(b, s)
}

```

返回结果如下:
```
=== RUN   TestJSON
--- PASS: TestJSON (0.00s)
    go_test.go:38: raw1={"a":1,"str":"test","b":["a","b"]}
    go_test.go:45: raw2={"a":2}
    go_test.go:50: s 本身为空的情况
    go_test.go:57: decode raw1 to s: {A:1 Str:test B:[a b]}
    go_test.go:64: decode raw2 to s: {A:2 Str: B:[]}
    go_test.go:66: s 本身有值的情况
    go_test.go:77: s={A:3 Str:test3 B:[3 4]}
    go_test.go:84: decode raw1 to s: {A:1 Str:test B:[a b]}
    go_test.go:95: s={A:3 Str:test3 B:[3 4]}
    go_test.go:101: decode raw2 to s: {A:2 Str:test3 B:[3 4]}
=== RUN   TestGob
--- PASS: TestGob (0.00s)
    go_test.go:131: s 本身为空的情况
    go_test.go:138: decode raw1 to s: {A:1 Str:test B:[a b]}
    go_test.go:145: decode raw2 to s: {A:2 Str: B:[]}
    go_test.go:147: s 本身不为空的情况
    go_test.go:158: s={A:3 Str:test3 B:[3 4]}
    go_test.go:165: decode raw1 to s: {A:1 Str:test B:[a b]}
    go_test.go:175: s={A:3 Str:test3 B:[3 4]}
    go_test.go:181: decode raw2 to s: {A:2 Str:test3 B:[3 4]}
PASS
ok  	test	0.400s
```

从结果上看，反序列化的过程应该是将二进制对应的值去覆盖要写入的结构体实例对应的字段。而不会对原有的值进行清空。另外，json包反序列化的时候，如果没有指定omitempty，序列化生成的json串所有的字段都是有值的，只不过值是空。

