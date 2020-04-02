# 设计模式笔记-策略模式

<!--more-->

## 什么是策略模式
定义了算法族，分别封装起来，让他们直接可以互相替换。让算法的变换独立于算法的客户。


uml图大概是这个样子
![](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/dust347/blog_image/master/design-pattern-notes-strategy_pattern/strategy-patterm.puml )
<!--
``` plantuml
 @startuml
interface StrategyInterface {
  exe()
} 
StrategyA : exe()
StrategyB : exe()

StrategyInterface <|.. StrategyA
StrategyInterface <|.. StrategyB

StrategyContext *-- StrategyInterface

@enduml
```
-->

StrategyInterface 定义了算法族  
StrategyA, StrategyB 实现了具体的算法(分别封装起来)  
对 StrategyContext 来说(StrategyContext 是使用 StrategyInterface这个算法族的客户)，不同的算法实现与自身独立，具体算法可以互相替换。

## 为什么使用策略模式
以上面的类图为例，为什么必须要使用策略模式？  

既然一个 StrategyContext 基本上与一个具体的 StrategyInterface 具体实现绑定，用普通的继承可以吗？比如这种：
![](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/dust347/blog_image/master/design-pattern-notes-strategy_pattern/other.puml )
<!--
``` plantuml
@startuml
interface StrategyInterface {
  exe()
} 

StrategyA : exe()
StrategyB : exe()

StrategyInterface <|.. StrategyA
StrategyInterface <|.. StrategyB


StrategyA <|-- StrategyContextA
StrategyB <|-- StrategyContextB

@enduml

```
-->

> 我们约定Strategy 是具体的算法实现, StrategyContext是具体的客户代码。可能StrategyContext还有别的什么逻辑。不单单只是调用算法。

对比策略模式的uml图，上面的这个uml采用的是**继承**，而策略模式的类图采用的是**组合**。

使用继承的方式，一个具体的 StrategyContent 与具体的 StrategyInterface 实现强行绑定到一起。如果要新增一个新的 StrategyC ，必须在新加一个对应的 StrategyContentC 。达不到我们希望的“算法与客户独立”的期望。

同时，面向对象使用继承的方式实质上是通过将公共逻辑抽象为基类，供不同的派生类使用，达到代码复用的目的。所以基类本身应该是不变的。



在这样的场景下，算法族的具体实现本身是可能变化的。**将可能变化的部分独立出来，不要与不需要变化的部分混在一起**是设计模式的一个设计原则。
而使用组合（has-a）比使用继承（is-a）更容易应对这种场景。
> 考虑下，如果 StrategyContent 依赖2种不同的算法族会怎么样，如果使用继承的方式 uml图会是什么样子的。。。2种以上又是什么样子?
> 
> 另外，由于 StrategyInterface 只是 StrategyContent 的成员，我们在使用的过程中随时可以改变。


## 代码示例
使用 go 语言实现
``` go
package designpattern

import "fmt"

//StrategyInterface 策略接口
type StrategyInterface interface {
	Exe()
}

//StrategyA 策略实现A
type StrategyA struct{}

//Exe 执行策略
func (s *StrategyA) Exe() {
	fmt.Println("exe StrategyA")
}

//StrategyB 策略实现A
type StrategyB struct{}

//Exe 执行策略
func (s *StrategyB) Exe() {
	fmt.Println("exe StrategyB")
}

//StrategyContent 使用策略的类
type StrategyContent struct {
	s StrategyInterface
}

//SetStrategy 设定策略
func (c *StrategyContent) SetStrategy(s StrategyInterface) {
	c.s = s
}

//Run 执行具体逻辑
func (c *StrategyContent) Run() {
	fmt.Println("begin run")
	c.s.Exe()
	fmt.Println("end run")
}
```


测试代码如下：
```go
package designpattern_test

import (
	"designpattern"
	"fmt"
	"testing"
)

func TestStrategyPattern(t *testing.T) {
	var c designpattern.StrategyContent

	c.SetStrategy(new(designpattern.StrategyA))
	c.Run()

	fmt.Println("===")

	c.SetStrategy(new(designpattern.StrategyB))
	c.Run()

}
```

运行结果：
```
begin run
exe StrategyA
end run
===
begin run
exe StrategyB
end run
PASS
```


