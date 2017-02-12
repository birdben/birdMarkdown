---
title: "Go学习（四）Go复合数据类型总结"
date: 2017-01-10 14:02:41
tags: [Go]
categories: [Go]
---

### Map

map是一种key-value的关系，一般都会使用make来初始化内存，有助于减少后续新增操作的内存分配次数。假如一开始定义了话，但没有用make来初始化，会报错的。

```
package main
import (
    "fmt"
)

func main(){
    var test =  map[string]string{"姓名":"李四","性别":"男"}
    name, ok := test["姓名"]
    // 假如key存在，则name = 李四，ok = true，否则，ok = false
    if ok {
        fmt.Println(name)
    }
    delete(test,"姓名")//删除为姓名为key的值，不存在没关系
    fmt.Println(test)
    var a map[string]string
    a["b"] = "c"//这样会报错的，要先初始化内存
    a = make(map[string]string)
    a["b"] = "c"//这样才不会错
}
```

### interface

接口的转换遵循以下规则：

- 普通类型向接口类型的转换是隐式的。
- 接口类型向普通类型转换需要类型断言。

```
package main

import (
    "fmt"
)

func main() {
    var val interface{} = "hello"
    fmt.Println(val)
    val = []byte{'a', 'b', 'c'}
    fmt.Println(val)
}
```

```
package main

import "fmt"

func main() {
	var i32 interface{}
	i32 = int32(1)
	
	var i int
	//i = i32.(int)  // won't work, you need to assert against the exact type in i32
	
	i32_tmp := i32.(int32) // this is called a type assertion
	i = int(i32_tmp)
	
	fmt.Println(i)
}
```

正如您所预料的，"hello"作为string类型存储在interface{}类型的变量val中，[]byte{'a', 'b', 'c'}作为slice存储在interface{}类型的变量val中。这个过程是隐式的，是编译期确定的。

接口类型向普通类型转换有两种方式：Comma-ok断言和switch测试。任何实现了接口I的类型都可以赋值给这个接口类型变量。由于interface{}包含了0个方法，所以任何类型都实现了interface{}接口，这就是为什么可以将任意类型值赋值给interface{}类型的变量，包括nil。还有一个要注意的就是接口的实现问题，*T 包含了定义在 T 和 *T 上的所有方法，而T只包含定义在T上的方法。我们来看一个例子：

```
package main

import (
    "fmt"
)

// 演讲者接口
type Speaker interface {
    // 说
    Say(string)
    // 听
    Listen(string) string
    // 打断、插嘴
    Interrupt(string)
}

// 王兰讲师
type WangLan struct {
    msg string
}

func (this *WangLan) Say(msg string) {
    fmt.Printf("王兰说：%s\n", msg)
}

func (this *WangLan) Listen(msg string) string {
    this.msg = msg
    return msg
}

func (this *WangLan) Interrupt(msg string) {
    this.Say(msg)
}

// 江娄讲师
type JiangLou struct {
    msg string
}

func (this *JiangLou) Say(msg string) {
    fmt.Printf("江娄说：%s\n", msg)
}

func (this *JiangLou) Listen(msg string) string {
    this.msg = msg
    return msg
}

func (this *JiangLou) Interrupt(msg string) {
    this.Say(msg)
}

func main() {
    wl := &WangLan{}
    jl := &JiangLou{}

    var person Speaker
    person = wl
    person.Say("Hello World!")
    person = jl
    person.Say("Good Luck!")
}
```

Speaker接口有两个实现WangLan类型和JiangLou类型。但是具体到实例来说，变量wl和变量jl只有是对应实例的指针类型才真正能被Speaker接口变量所持有。这是因为WangLan类型和JiangLou类型所有对Speaker接口的实现都是在*T上。这就是上例中person能够持有wl和jl的原因。

想象一下Java的泛型(很可惜golang不支持泛型)，java在支持泛型之前需要手动装箱和拆箱。由于golang能将不同的类型存入到接口类型的变量中，使得问题变得更加复杂。所以有时候我们不得不面临这样一个问题：我们究竟往接口存入的是什么样的类型？有没有办法反向查询？答案是肯定的。

Comma-ok断言的语法是：value, ok := element.(T)。element必须是接口类型的变量，T是普通类型。如果断言失败，ok为false，否则ok为true并且value为变量的值。来看个例子：

```
package main

import (
    "fmt"
)

type Html []interface{}

func main() {
    html := make(Html, 5)
    html[0] = "div"
    html[1] = "span"
    html[2] = []byte("script")
    html[3] = "style"
    html[4] = "head"
    for index, element := range html {
        if value, ok := element.(string); ok {
            fmt.Printf("html[%d] is a string and its value is %s\n", index, value)
        } else if value, ok := element.([]byte); ok {
            fmt.Printf("html[%d] is a []byte and its value is %s\n", index, string(value))
        }
    }
}
```

其实Comma-ok断言还支持另一种简化使用的方式：value := element.(T)。但这种方式不建议使用，因为一旦element.(T)断言失败，则会产生运行时错误。如：

```
package main

import (
    "fmt"
)

func main() {
    var val interface{} = "good"
    fmt.Println(val.(string))
    // fmt.Println(val.(int))
}
以上的代码中被注释的那一行会运行时错误。这是因为val实际存储的是string类型，因此断言失败。

还有一种转换方式是switch测试。既然称之为switch测试，也就是说这种转换方式只能出现在switch语句中。可以很轻松的将刚才用Comma-ok断言的例子换成由switch测试来实现：

package main

import (
    "fmt"
)

type Html []interface{}

func main() {
    html := make(Html, 5)
    html[0] = "div"
    html[1] = "span"
    html[2] = []byte("script")
    html[3] = "style"
    html[4] = "head"
    for index, element := range html {
        switch value := element.(type) {
        case string:
            fmt.Printf("html[%d] is a string and its value is %s\n", index, value)
        case []byte:
            fmt.Printf("html[%d] is a []byte and its value is %s\n", index, string(value))
        case int:
            fmt.Printf("error type\n")
        default:
            fmt.Printf("unknown type\n")
        }
    }
}
```

```
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var val interface{} = int64(58)
    fmt.Println(reflect.TypeOf(val))
    val = 50
    fmt.Println(reflect.TypeOf(val))
}
```

我们已经知道接口类型的变量底层是作为两个成员来实现，一个是type，一个是data。type用于存储变量的动态类型，data用于存储变量的具体数据。在上面的例子中，第一条打印语句输出的是：int64。这是因为已经显示的将类型为int64的数据58赋值给了interface类型的变量val，所以val的底层结构应该是：(int64, 58)。我们暂且用这种二元组的方式来描述，二元组的第一个成员为type，第二个成员为data。第二条打印语句输出的是：int。这是因为字面量的整数在golang中默认的类型是int，所以这个时候val的底层结构就变成了：(int, 50)。

理解上就是，感觉比静态的类型逼格高一点,总之我的interface就写在那里，你要是想继承我，使用我的方法,就把这些方法实现了就行，并不要求你显示的声明，继承自我，怎样怎样。有些动态类型的语言在运行期才能进行类型的语法检测，go语言在编译期间就可以检测。


参考文章：

- http://blog.csdn.net/newjueqi/article/details/36388665
- http://blog.csdn.net/kjfcpua/article/details/18665041
- http://blog.csdn.net/kjfcpua/article/details/18667255
- https://play.golang.org/p/EVyMNehzbq


编程规范
- https://segmentfault.com/a/1190000000464394
- http://studygolang.com/articles/2059