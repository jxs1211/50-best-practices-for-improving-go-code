19 不要在函数参数中使用空接口(interface{})

## 不要在函数参数中使用空接口(interface{})

> 空接口不提供任何信息(The empty interface says nothing)。- Rob Pike，Go 语言之父

现今主流编程语言中，第一个正式支持接口（interface）的是 Java 语言。在 Java 中，如果某个类要实现一个接口，它需要显式使用`implements`关键字作出声明，就像下面这样。否则即便该类实现了接口类型的所有方法，这个类也不算是这个接口的一个实现者：

```java
interface MyInterface {
	... ...
}

public class MyInterfaceImpl implements MyInterface {
	... ...
} 
```

如果这个类要实现多个接口，可以在 implements 后面放置多个接口名，接口名之间使用逗号分隔：

```java
public class MyInterfaceImpl implements MyInterface, MyInterface1, MyInterface2 {
	... ...
} 
```

成为某接口类型的实现者之后，我们就可以将该类型的实例传递给该接口类型变量了：

```java
MyInterface i = new MyInterfaceImpl(); 
```

下面是一个完整的示例：

```java
// java_stringer/StringerInterface.java
package stringer;

interface Stringer {
    String String();
}

public class StringerInterface {
	public static String concat(Stringer a, Stringer b) {
		return a.String()+b.String();
	}

	public static void main(String args[]){
		bar b = new bar();
		b.s = "hello";
		foo f = new foo();
		f.i = 5;
		System.out.println(concat(b, f));
	}
}

// java_stringer/bar.java
package stringer;

public class bar implements Stringer {
        String s;
        public String String() {
                return s;
        }
}

// java_stringer/foo.java
package stringer;

public class foo implements Stringer {
        int i;
        public String String() {
                return Integer.toString(i);
        }
} 
```

上面源文件定义了一个 Stringer 接口，两个 Java 类 foo 和 bar 分别实现了该接口。concat 方法接受两个 Stringer 类型参数，并将两个参数的 String 方法的调用结果连接后返回。

运行该示例：

```java
$make
javac foo.java bar.java StringerInterface.java
mv *.class stringer
$java stringer/StringerInterface
hello5 
```

我们看到在 Java 中，要实现一个接口，两个条件必不可少：

- 使用`implements`关键字显式声明要实现的接口；
- 实现接口的所有方法。

Java 编译器会在编译阶段对接口类型变量的赋值进行匹配判定，如果不满足上述两个条件，Java 编译器就会报错，不会让这类错误 **“漏”** 到运行时发生。

动态语言由于无需静态声明变量类型，因此可以将任意数据类型传递给方法。下面是使用 Ruby 实现`Stringer接口`和`concat方法`的例子：

```java
// ruby_stringer/stringer_interface.rb

#!/usr/bin/env ruby
# -*- coding: UTF-8 -*-

def concat(a, b)
  unless a.respond_to?(:string) && b.respond_to?(:string)
    raise ArgumentError, "无效参数"
  end
  a.string() + b.string()
end

class Stringer
  def string
    raise NotImplementedError
  end
end

class Foo
  def initialize(i)
      @i=i
  end
  def string
    @i.to_s
  end
end

class Bar
  def initialize(s)
      @s=s
  end
  def string
    @s
  end
end

f = Foo.new(5)
b = Bar.new("hello")
puts concat(b, f) 
```

Ruby 本身并不支持接口，这里仅是一个**模拟实现**(当然还可以使用 Ruby 的 module mixin 来实现)。我们定义了一个名为 Stringer 的类来模拟接口，我们在它的 string 方法实现中“抛出异常”以暗示该类被用于模拟一个接口。其实该类也可以去掉，这里仅是为了让大家能更直观的“感受”到接口才将其保留。concat 函数接受两个参数，并期望两个参数的实参对象都支持 string 方法。但由于动态语言可以将任意类型传入，因此这里针对参数做了一些安全检查。

运行该示例：

```ruby
$ruby stringer_interface.rb 
hello5 
```

我们看到：和 Java 的严格“约束”和编译期检查不同，动态语言走向另外一个“极端”：“接口”的实现者无需做任何显式的接口实现声明，ruby 解释器也不做任何检查（我们可以手工在实现中增加一些检查以提升安全性，就像上面 concat 代码中做的那样）。

Go 对接口的支持介于这两种语言的中间。在 Go 中你不必像 Java 那样显式声明某个类型实现了某个接口，这在某种意义上它与 Ruby 类似；但是另一方面，你必须声明该接口，这又与接口在 Java 等静态类型语言中的工作方式更加一致。这种不需要类型显式声明实现了某个接口的方式可以使得种类繁多的类型与接口匹配，包括那些存量的、并非由你编写的代码以及你无法编辑的代码(比如：标准库）。

Go 的这种方式兼顾安全性和灵活性，其中安全性是由 Go 编译器来保证的，而为编译器提供输入信息的恰是接口类型的定义。比如下面的接口：

```go
// $GOROOT/src/io/io.go
type Reader interface {
        Read(p []byte) (n int, err error)
} 
```

Go 编译器通过解析该接口定义得到接口的名字信息以及其方法信息，在为此接口类型参数赋值时，编译器就会根据这些信息对实参进行检查。这时，如果函数或方法的参数类型为空接口`interface{}`，会发生什么呢？这恰好就应了本节开篇引用的 Rob Pike 的那句话：“空接口不提供任何信息”。这里“提供”一词的对象不是开发者，而是编译器。在函数或方法参数中使用空接口类型，意味着你没有为编译器提供关于传入实参数据的任何信息，因此，你将失去静态类型语言类型安全检查的 **”保护屏障“**，你需要自己检查类似的错误，并且直到运行时才能发现此类错误。

因此，建议广大 Gopher 尽可能的抽象出带有一定行为契约的接口，并将其作为函数参数类型，**尽量不要使用可以”逃过“编译器类型安全检查的空接口类型(interface{})**。

在这方面，Go 标准库为我们做出了”表率“。全面搜索标准库后，你可以发现以`interface{}`为参数类型的方法和函数少之甚少。使用`interface{}`作为参数类型的函数或方法主要有两类：

- 容器算法类 - 比如：container 下的 heap、list 和 ring 包、sort 包、sync.Map 等
- 格式化/日志类 - 比如：fmt 包、log 包等

这些使用`interface{}`作为参数类型的函数或方法的共同特点就是它们面对的都是未知类型的数据，因此使用`interface{}`也可以理解为在 Go 语言尚未支持泛型这个阶段的一个权宜之计。

最后，我们小结一下本节的内容：

- 仅在处理未知类型数据时使用空接口类型；
- 其他情况下，尽可能将你需要的行为抽象成带有方法的接口，并使用这样的非空接口类型作为函数或方法的参数。