39 慎用reflect包提供的反射能力

## 慎用reflect包提供的反射能力

Go 在标准库中提供的 `reflect` 包让 Go 程序具备运行时的**反射能力 (reflection)**。 **反射**是计算机应用程序在运行时访问、检测和修改它本身状态或行为的一种能力，各种编程语言所实现的反射机制各有不同。我们知道 Go 语言的 `interface{}` 类型变量具有析出任意类型变量的类型信息 (`type`) 和值信息 (`value`) 的能力 (可参考第 26 条 “了解接口类型变量的内部表示”)，Go 的反射本质上就是利用 `interface{}` 的这种能力在运行时对任意变量的**类型和值信息进行检视甚至是对值进行修改**的一种机制。

反射让静态类型语言 Go 在运行时具备了某种基于类型信息的 **“动态特性”**，利用这种特性，`fmt.Println` 在无法提前获知传入参数的真正类型的情况下依旧可以对其进行正确地格式化输出；`json.Marshal` 也是通过这种特性对传入的任意结构体类型进行 “解构” 并正确生成对应的 JSON 文本。下面我们通过一个简单的构建 SQL 查询语句的例子来更为直观地感受 Go 反射的 “魔法”：

```go
// go-reflect/construct_sql_query_stmt.go 
package main

import (
	"bytes"
	"errors"
	"fmt"
	"reflect"
	"time"
)

func ConstructQueryStmt(obj interface{}) (stmt string, err error) {
	// 仅支持struct或struct指针类型
	typ := reflect.TypeOf(obj)
	if typ.Kind() == reflect.Ptr {
		typ = typ.Elem()
	}
	if typ.Kind() != reflect.Struct {
		err = errors.New("only struct is supported")
		return
	}

	buffer := bytes.NewBufferString("")
	buffer.WriteString("SELECT ")

	if typ.NumField() == 0 {
		err = fmt.Errorf("the type[%s] has no fields", typ.Name())
		return
	}

	for i := 0; i < typ.NumField(); i++ {
		field := typ.Field(i)

		if i != 0 {
			buffer.WriteString(", ")
		}
		column := field.Name
		if tag := field.Tag.Get("orm"); tag != "" {
			column = tag
		}
		buffer.WriteString(column)
	}

	stmt = fmt.Sprintf("%s FROM %s", buffer.String(), typ.Name())
	return
}

type Product struct {
	ID        uint32
	Name      string
	Price     uint32
	LeftCount uint32 `orm:"left_count"`
	Batch     string `orm:"batch_number"`
	Updated   time.Time
}

type Person struct {
	ID      string
	Name    string
	Age     uint32
	Gender  string
	Addr    string `orm:"address"`
	Updated time.Time
}

func main() {
	stmt, err := ConstructQueryStmt(&Product{})
	if err != nil {
		fmt.Println("construct query stmt for Product error:", err)
		return
	}
	fmt.Println(stmt)

	stmt, err = ConstructQueryStmt(Person{})
	if err != nil {
		fmt.Println("construct query stmt for Person error:", err)
		return
	}
	fmt.Println(stmt)
} 
```

在这个例子中，我们传给 `ConstructQueryStmt` 函数的参数是结构体实例，得到的是该结构体对应的表的数据查询语句文本，我们采用了一种 ORM (Object Relational Mapping，对象关系映射) 风格的实现。`ConstructQueryStmt` 通过反射获得传入的参数 `obj` 的类型信息，包括 (导出) 字段数量、字段名、字段标签值等，并根据这些类型信息生成 SQL 查询语句文本。如果结构体字段带有 `orm` 标签，该函数会使用标签值替代默认列名 (字段名)。如果将 `ConstructQueryStmt` 包装成一个包的导出 API，那么它可以被放入任何 Go 应用中，并在运行时为传入的任意结构体类型实例生成对应的查询语句。

我们来看一下上述示例的运行结果：

```SQL
$go run construct_sql_query_stmt.go 
SELECT ID, Name, Price, left_count, batch_number, Updated FROM Product
SELECT ID, Name, Age, Gender, address, Updated FROM Person 
```

Go 反射十分适合处理这一类问题，它们的典型特点包括：

- 输入参数的类型无法提前确定；
- 函数或方法的处理结果因传入的参数 (的类型信息和值信息) 的不同而异。

如果没有反射机制，要想在 Go 中优雅地解决此类问题几乎是不可能的。但 Go 语言之父 Rob Pike 在 2015 年的 [Gopherfest](http://www.gopherfest.org/) 技术大会上却 [告诫大家](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=15m22s)：**反射并不是 Go 推荐的惯用法，建议大家谨慎使用**。绝大多数情况下，反射都不是为你提供的。反射在带来强大功能的同时，也是很多困扰你的问题的来源，比如：

- 反射让你的代码逻辑看起来不再那么清晰，难于理解；
- 反射让你的代码运行的更慢；
- 在编译阶段，编译器无法检测到使用反射的代码中的问题，这种问题只能在 Go 程序运行时暴露出来；并且一旦暴露，很大可能会导致运行时的 panic。

Rob Pike 还为 Go 反射的规范使用定义了[三大法则](https://blog.golang.org/laws-of-reflection)，如果经过评估，你必须使用反射才能实现你要的功能特性，那么你在使用反射时需要牢记这三条法则：

1. 反射世界的入口：经由接口 (`interface{}`) 类型变量值进入到反射的世界并获得对应的反射对象 (`reflect.Value` 或 `reflect.Type`)；
2. 反射世界的出口：反射对象 (`reflect.Value`) 通过化身为一个接口 (`interface{}`) 类型变量值的形式走出反射世界；
3. 修改反射对象的前提：其对应的 `reflect.Value` 必须是可设置的 (`Settable`)。

对于前两条法则，我们可以用下面的图来表示：

![39 慎用reflect包提供的反射能力](https://img-hello-world.oss-cn-beijing.aliyuncs.com/c1340a6ad3b6c06bfd8c633b62441033.png)

图 9-6-1：Go 变量与反射对象之间的转换关系

- 进入到反射世界的入口：`reflect.TypeOf` 和 `reflect.ValueOf`

`reflect.TypeOf` 和 `reflect.ValueOf` 是进入反射世界的**仅有的两扇 “大门”**。通过 `reflect.TypeOf` 这扇门进入到反射世界，你将得到一个 `reflect.Type` 对象，该对象中包含了被反射的 Go 变量实例的所有类型信息；而通过 `reflect.ValueOf` 这扇门进入到反射世界，你将得到一个 `reflect.Value` 对象，`Value` 对象是反射世界的核心，该对象中不仅包含了被反射的 Go 变量实例的值信息，通过调用该对象的 `Type` 方法，我们还可以得到 Go 变量实例的类型信息，这与通过 `reflect.TypeOf` 获得类型信息是等价的：

```go
// reflect.ValueOf().Type() 等价于 reflect.TypeOf()
var i int = 5
val := reflect.ValueOf(i)
typ := reflect.TypeOf(i)
fmt.Println(reflect.DeepEqual(typ, val.Type())) // true 
```

反射世界入口可以获取 Go 变量实例的类型信息和值信息的关键在于它们利用了 `interface{}` 类型形式参数对传入的实际参数 (Go 变量实例) 的析构能力 (可参考第 26 条 “了解接口类型变量的内部表示”)，两个入口函数分别将得到的值信息和类型信息存储在 `reflect.Value` 对象和 `reflect.Type` 对象中。

进入反射世界后，我们就可以通过 `reflect.Value` 实例和 `reflect.Type` 实例进行**值信息和类型信息的检视**。我们先来看看对 Go 的一些简单原生类型的检视结果：

```go
// go-reflect/examine_value_and_type.go 

// 简单原生类型
var b = true // 布尔类型
val := reflect.ValueOf(b)
typ := reflect.TypeOf(b)
fmt.Println(typ.Name(), val.Bool()) // bool true

var i = 23 // 整型
val = reflect.ValueOf(i)
typ = reflect.TypeOf(i)
fmt.Println(typ.Name(), val.Int()) // int 23

var f = 3.14 // 浮点型
val = reflect.ValueOf(f)
typ = reflect.TypeOf(f)
fmt.Println(typ.Name(), val.Float()) // float64 3.14

var s = "hello, reflection" // 字符串
val = reflect.ValueOf(s)
typ = reflect.TypeOf(s)
fmt.Println(typ.Name(), val.String()) //string hello, reflection

var fn = func(a, b int) int { // 函数(一等公民)
	return a + b
}
val = reflect.ValueOf(fn)
typ = reflect.TypeOf(fn)
fmt.Println(typ.Kind(), typ.String()) // func func(int, int) int 
```

`reflect.Value` 类型拥有很多方便我们进行值检视的方法，比如：`Bool`、`Int`、`String` 等，但显然这些方法不能对所有的变量类型都适用，比如：`Bool` 方法仅适用于对布尔类型变量进行反射后得到的 `Value` 对象。一旦应用的方法与 `Value` 对象的值类型不匹配，我们将收到运行时 `panic`：

```go
var i = 17
val := reflect.ValueOf(i)
fmt.Println(val.Bool()) // panic: reflect: call of reflect.Value.Bool on int Value 
```

`reflect.Type` 是一个接口类型，它包含了很多用于检视类型信息的方法，而对于简单原生类型来说，通过 `Name`、`String` 或 `Kind` 方法就可以得到我们想要的类型名称或类型类别等信息。`Name` 方法返回有确定定义的类型的名字 (不包括包名前缀)，比如：`int`、`string`，对于上面的函数类型变量，`Name` 方法将返回空；我们可以通过 `String` 方法得到**类型的描述字符串**，比如上面的 `func(int, int) int`。`String` 方法返回的类型描述可能包含包名 (一般使用短包名，即仅使用包导入路径的最后一段)，比如：`main.Person`；`Type` 接口的 `Kind` 方法则返回**类型的特定类别**，比如下面的两个变量 `pi` 和 `ps` 虽然是不同类型的指针，但是它们的 `Kind` 都是 `ptr`：

```go
var pi = (*int)(nil)
var ps = (*string)(nil)
typ := reflect.TypeOf(pi)
fmt.Println(typ.Kind(), typ.String()) // ptr *int
					     
typ = reflect.TypeOf(ps)                     
fmt.Println(typ.Kind(), typ.String()) // ptr *string 
```

接下来我们再来看看对原生复合类型以及其他自定义类型的检视结果：

```go
// go-reflect/examine_value_and_type.go 

// 原生复合类型
var sl = []int{5, 6} // 切片
val = reflect.ValueOf(sl)
typ = reflect.TypeOf(sl)
fmt.Printf("[%d %d]\n", val.Index(0).Int(),
	val.Index(1).Int()) // [5, 6]
fmt.Println(typ.Kind(), typ.String()) // slice []int

var arr = [3]int{5, 6} // 数组
val = reflect.ValueOf(arr)
typ = reflect.TypeOf(arr)
fmt.Printf("[%d %d %d]\n", val.Index(0).Int(),
	val.Index(1).Int(), val.Index(2).Int()) // [5 6 0]
fmt.Println(typ.Kind(), typ.String()) // array [3]int

var m = map[string]int{ // map
	"tony": 1,
	"jim":  2,
	"john": 3,
}
val = reflect.ValueOf(m)
typ = reflect.TypeOf(m)
iter := val.MapRange()
fmt.Printf("{")
for iter.Next() {
	k := iter.Key()
	v := iter.Value()
	fmt.Printf("%s:%d,", k.String(), v.Int())
}
fmt.Printf("}\n")                     // {tony:1,jim:2,john:3,}
fmt.Println(typ.Kind(), typ.String()) // map map[string]int

type Person struct {
	Name string
	Age  int
}

var p = Person{"tony", 23} // 结构体
val = reflect.ValueOf(p)
typ = reflect.TypeOf(p)
fmt.Printf("{%s, %d}\n", val.Field(0).String(),
	val.Field(1).Int()) // {"tony", 23}

fmt.Println(typ.Kind(), typ.Name(), typ.String()) // struct Person main.Person

var ch = make(chan int, 1) // channel
val = reflect.ValueOf(ch)
typ = reflect.TypeOf(ch)
ch <- 17
v, ok := val.TryRecv()
if ok {
	fmt.Println(v.Int()) // 17
}
fmt.Println(typ.Kind(), typ.String()) // chan chan int

// 其他自定义类型
type MyInt int

var mi MyInt = 19
val = reflect.ValueOf(mi)
typ = reflect.TypeOf(mi)
fmt.Println(typ.Name(), typ.Kind(), typ.String(), val.Int()) // MyInt int main.MyInt 19 
```

通过 `Value` 提供的 `Index` 方法，我们可以获取到切片以及数组类型的元素所对应的 `Value` 对象值，通过后者我们可以得到其值信息；通过 `Value` 的 `MapRange`、`MapIndex` 等方法我们可以获取到 map 中的 `key` 和 `value` 对象所对应的 `Value` 对象值，有了 `Value` 对象，我们就可以像上面获取简单原生类型的值信息那样获得这些元素的值信息；对于结构体类型，`Value` 提供了 `Field` 系列方法，上面示例中，我们通过下标的方式 (`Field` 方法) 获取结构体字段所对应的 `Value` 对象，从而获取字段的值信息。

通过反射对象，我们还可以调用函数或对象的方法：

```go
// go-reflect/call_func_and_method.go 
... ...
func Add(i, j int) int {
	return i + j
}

type Calculator struct{}

func (c Calculator) Add(i, j int) int {
	return i + j
}

func main() {
	// 函数调用
	f := reflect.ValueOf(Add)
	var i = 5
	var j = 6
	vals := []reflect.Value{reflect.ValueOf(i), reflect.ValueOf(j)}
	ret := f.Call(vals)
	fmt.Println(ret[0].Int()) // 11

	// 方法调用
	c := reflect.ValueOf(Calculator{})
	m := c.MethodByName("Add")
	ret = m.Call(vals)
	fmt.Println(ret[0].Int()) // 11
} 
```

我们看到通过函数类型变量或包含有方法的类型实例反射出的 `Value` 对象，我们可以通过其 `Call` 方法调用该函数或类型的方法。函数或方法的参数以 `reflect.Value` 类型切片的形式提供，函数 / 方法的返回值也以 `reflect.Value` 类型切片的形式返回。不过务必保证 `Value` 参数的类型信息与原函数 / 方法的参数的类型相匹配，否则也会导致运行时 `panic`：

```go
// go-reflect/call_func_and_method.go 
var k float64 = 3.14                                                
ret = m.Call([]reflect.Value{reflect.ValueOf(i), 
        reflect.ValueOf(k)}) // panic: reflect: Call using float64 as type int 
```

- 出口：通过 `reflect.Value.Interface()` 将 `reflect.Value` 对象恢复成一个 `interface{}` 类型变量值

`reflect.Value.Interface()` 是 `reflect.ValueOf()` 的逆过程，通过 `Interface` 方法我们可以将 `reflect.Value` 对象恢复成一个 `interface{}` 类型变量值。这个离开反射世界的过程实质是将 `reflect.Value` 中的类型信息和值信息重新打包成一个 `interface{}` 的内部表示。之后，我们就可以像上图中展示的那样，通过类型断言 (`type assertion`) 得到一个反射前的类型变量值：

```go
// go-reflect/reflect_value_to_interface.go 
... ...
func main() {
	var i = 5
	val := reflect.ValueOf(i) // enter the reflection world
	r := val.Interface().(int) // exit the reflection world
	fmt.Println(r) // 5
	r = 6
	fmt.Println(i, r) // 5 6

	val = reflect.ValueOf(&i)
	q := val.Interface().(*int)
	fmt.Printf("%p, %p, %d\n", &i, q, *q) // 0xc0000b4008, 0xc0000b4008, 5
	*q = 7
	fmt.Println(i) // 7
} 
```

通过上述例子，我们看到通过 `reflect.Value.Interface()` 重建后的 `interface{}` 类型变量而得到的新变量 (比如例子中的 `r`) 与原变量 (比如例子中的 `i`) 是两个不同的变量，它们的唯一 “联系” 就是值相同。这样，如果我们反射的对象是一个指针 (就像例子中的 `&i`)，那么我们通过 `reflect.Value.Interface()` 得到的新变量 (如例子中的 `q`) 也是一个指针，且它所指的内存地址与原指针变量相同。通过新指针变量对所指内存值的修改会反映到原变量上 (变量 `i` 的值由 5 变为 7)。

- 输出参数、`interface{}` 类型变量以及反射对象的可设置性 (`Settable`)

在学习传统编程语言 (比如 C 语言) 的函数概念的时候，通常会有输入参数、输出参数的概念，当然 Go 语言也是支持这些概念的，比如下面例子：

```go
func myFunc(in int, out *int) {
	in = 1
	*out = in + 10
}

func main() {
	var n = 17
	var m = 23
	fmt.Printf("n=%d, m=%d\n", n, m) // n=17, m=23
	myFunc(n, &m)
	fmt.Printf("n=%d, m=%d\n", n, m) // n=17, m=11
} 
```

例子中 `in` 是一个输入参数，函数体内对 `in` 的修改不会影响到作为实参传入 `myFunc` 的变量 `n`，因为 Go 函数参数的传递是传值，即值拷贝；`out` 是输出参数，它的传递也是值拷贝，但是这里拷贝的却是指针值，即作为实参参数 `myFunc` 的变量 `m` 的地址，这样函数体内通过解引用对 `out` 所指内存地址上的值的修改就会同步修改变量 `m` 的值。

对于以 `interface{}` 类型变量 `i` 作为形式参数的 `reflect.ValueOf` 和 `reflect.TypeOf` 函数来说，`i` 自身是被反射对象的 “复制品”，就像上面函数的输入参数那样。而新创建的反射对象又拷贝了 `i` 中所包含的值信息，因此当被反射的对象以值类型 (`T`) 传递给 `reflect.ValueOf` 时，在反射世界中对反射对象的值信息的修改不会对被反射对象产生影响，于是 Go 的设计者们认为这种修改毫无意义，并且禁止了这种行为，一旦发生这种行为，将会导致运行时 `panic`：

```
var i = 17
val := reflect.ValueOf(i)
val.SetInt(27) // panic: reflect: reflect.flag.mustBeAssignable using unaddressable value 
```

`reflect.Value` 提供了 `CanSet`、`CanAddr` 以及 `CanInterface` 等方法可以帮助我们判断反射对象是否是**可设置的**、**可寻址的**以及**可恢复**为一个 `interface{}` 类型变量。我们来看一个具体的例子：

```go
// go-reflect/reflect_value_settable.go
... ...
type Person struct {
	Name string
	age  int
}

func main() {
	var n = 17
	fmt.Println("int:")
	val := reflect.ValueOf(n)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true

	fmt.Println("\n*int:")
	val = reflect.ValueOf(&n)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val = reflect.ValueOf(&n).Elem()
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // true true true

	fmt.Println("\nslice:")
	var sl = []int{5, 6, 7}
	val = reflect.ValueOf(sl)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val = val.Index(0)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // true true true

	fmt.Println("\narray:")
	var arr = [3]int{5, 6, 7}
	val = reflect.ValueOf(arr)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val = val.Index(0)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true

	fmt.Println("\nptr to array:")
	var pArr = &[3]int{5, 6, 7}
	val = reflect.ValueOf(pArr)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val = val.Elem()
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // true true true
	val = val.Index(0)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // true true true

	fmt.Println("\nstruct:")
	p := Person{"tony", 33}
	val = reflect.ValueOf(p)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val1 := val.Field(0) // Name
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val1.CanSet(), val1.CanAddr(), val1.CanInterface()) // false false true
	val2 := val.Field(1) // age
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val2.CanSet(), val2.CanAddr(), val2.CanInterface()) // false false false

	fmt.Println("\nptr to struct:")
	pp := &Person{"tony", 33}
	val = reflect.ValueOf(pp)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val = val.Elem()
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // true true true
	val1 = val.Field(0) // Name
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val1.CanSet(), val1.CanAddr(), val1.CanInterface()) // true true true
	val2 = val.Field(1) // age
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val2.CanSet(), val2.CanAddr(), val2.CanInterface()) // false true false

	fmt.Println("\ninterface:")
	var i interface{} = &Person{"tony", 33}
	val = reflect.ValueOf(i)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true
	val = val.Elem()
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // true true true

	fmt.Println("\nmap:")
	var m = map[string]int{
		"tony": 23,
		"jim":  34,
	}
	val = reflect.ValueOf(m)
	fmt.Printf("Settable = %v, CanAddr = %v, CanInterface = %v\n",
		val.CanSet(), val.CanAddr(), val.CanInterface()) // false false true

	val.SetMapIndex(reflect.ValueOf("tony"), reflect.ValueOf(12))
	fmt.Println(m) // map[jim:34 tony:12]
} 
```

通过上述例子，我们看到当被反射的对象以值类型 (`T`) 传递给 `reflect.ValueOf` 时，所得到的反射对象 (`Value`) 是不可设置的和不可寻址的；当被反射的对象以指针类型 (`*T或&T`) 传递给 `reflect.ValueOf` 时，我们通过 `reflect.Value` 的 `Elem` 方法可以得到代表着该指针所指内存对象的 `Value` 反射对象，而这个反射对象则是**可设置和可寻址的**，对其进行修改 (比如利用 `Value` 的 `SetInt` 方法) 将会像函数的输出参数那样直接修改被反射对象所指向的内存空间的值；同时，当传入结构体或数组指针时，我们通过 `Field` 或 `Index` 方法得到的代表结构体字段或数组元素的 `Value` 反射对象也是**可设置和可寻址的**；如果结构体中某个字段是非导出字段，则该字段是**可寻址但不是可设置的** (比如上面例子中的 `age` 字段)；当被反射的对象的静态类型是接口类型时 (就像上面的 `interface{}` 类型变量 `i`)，该被反射对象的动态类型决定了其进入反射世界后的可设置性。如果动态类型为 `*T或&T` 时，就像上面传给变量 `i` 的是 `&Person{}`，那么通过 `Elem` 方法获得的反射对象就是**可设置和可寻址的**；`map` 类型被反射对象是一个特殊的存在，它的 `key` 和 `value` 都是不可寻址和不可设置的，但我们可以通过 `Value` 提供的 `SetMapIndex` 方法对 `map` 反射对象进行修改，这种修改会同步到被反射的 `map` 变量中。

综上，`reflect` 包所提供的 Go 反射能力是一把 “双刃剑”，它既可以被用于优雅地解决一类特定的问题，但同时也带来了逻辑不清晰、性能问题以及难于发现问题和调试等困惑。因此，我们应**谨慎使用这种能力**，在做出使用的决定之前，认真评估反射是否是问题的唯一解决方案；在确定要使用反射能力后，我们也要遵循上述的三个反射法则的要求。

