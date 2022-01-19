30 Go 惯例：将测试依赖的外部数据文件放在 testdata 下面

## Go 惯例：将测试依赖的外部数据文件放在 testdata 下面

**测试固件是 Go 测试执行所需的上下文环境**，其中测试依赖的**外部数据文件**就是一种常见的测试固件（可以理解为静态测试固件，即无需在测试代码中为其单独编写固件的创建和清理辅助函数）。在一些包含文件 I/O 的包的测试中，我们经常需要从外部数据文件中加载数据或向外部文件写入结果数据以满足测试固件的需求。

在其他主流编程语言中，测试依赖的外部数据文件如何管理往往是由程序员们自行决定的。但 Go 语言是一门面向软件工程的语言。从工程化的角度出发，Go 的设计者们将一些在传统语言中由程序员自身习惯决定的事情都一一**规范化**了，这样可以最大程度地提升程序员间的协作效率。而对测试依赖的外部数据文件的管理就是 Go 语言在这方面的一个典型例子。本节我们就来看看 Go 管理测试依赖的外部数据文件所采用的一些惯例和最佳实践。

## 1. testdata 目录

Go 语言规定：Go 工具链将忽略名为`testdata`的目录。这样开发者在编写测试时，就可以在名为`testdata`的目录下存放和管理测试代码依赖的数据文件。而`go test`命令执行时会将被测试程序包源码所在目录设置为其工作目录，这样如果要使用`testdata`目录下的某数据文件，我们无需再处理各种恼人的路径问题，可以直接在测试代码中像下面这样定位到充当测试固件的数据文件：

```go
f, err := os.Open("testdata/data-001.txt") 
```

如果考虑到不同操作系统对路径分隔符定义的差别(Windows 使用反斜线“\”，Linux/MacOS 使用斜线“/”)，使用下面的方式可以使测试代码更具可移植性：

```go
f, err := os.Open(filepath.Join("testdata", "data-001.txt")) 
```

在`testdata`目录中管理测试依赖的外部数据文件的方式在标准库中有着广泛应用：

```shell
在$GOROOT/src路径下(Go 1.14)：

$find . -name "testdata" -print
./cmd/vet/testdata
./cmd/objdump/testdata
./cmd/asm/internal/asm/testdata
... ...
./image/testdata
./image/png/testdata
./mime/testdata
./mime/multipart/testdata
./text/template/testdata
./debug/pe/testdata
./debug/macho/testdata
./debug/dwarf/testdata
./debug/gosym/testdata
./debug/plan9obj/testdata
./debug/elf/testdata 
```

以`image/png/testdata`为例，这里存储着`png包`测试代码用作静态测试固件的外部依赖数据文件：

```shell
$ls
benchGray.png			benchRGB.png			invalid-palette.png
benchNRGBA-gradient.png		gray-gradient.interlaced.png	invalid-trunc.png
benchNRGBA-opaque.png		gray-gradient.png		invalid-zlib.png
benchPaletted.png		invalid-crc32.png		pngsuite/
benchRGB-interlace.png		invalid-noend.png

$ls testdata/pngsuite 
README			basn2c08.png		basn4a16.png		ftbgn3p08.png
README.original		basn2c08.sng		basn4a16.sng		ftbgn3p08.sng
... ...
basn0g16.sng		basn4a08.sng		ftbgn2c16.sng		ftp1n3p08.sng 
```

`png`包的测试代码将这些数据文件作为输入，并将经过被测函数(比如：`png.Decode`等)处理后得到的结果数据与预期数据对比：

```go
// $GOROOT/src/image/png/reader_test.go

var filenames = []string{
        "basn0g01",
        "basn0g01-30",
        "basn0g02",
	... ...
}

func TestReader(t *testing.T) {
        names := filenames
        if testing.Short() {
                names = filenamesShort
        }
        for _, fn := range names {
                // 读取.png文件
                img, err := readPNG("testdata/pngsuite/" + fn + ".png")
                if err != nil {
                        t.Error(fn, err)
                        continue
                }
                ... ...
                // 比较读取的数据img与预期数据
        }
        ... ...
} 
```

我们还经常将预期结果数据保存在文件中并放置在`testdata`下面，然后在测试代码中我们将被测对象输出的数据与这些预置在文件中的数据进行比较，一致则测试通过；反之，测试失败。我们来看一个例子：

```go
// testdata-demo1/attendee.go
package attendee

import (
	"encoding/xml"
	"strconv"
)

type Attendee struct {
	Name  string
	Age   int
	Phone string
}

func (a *Attendee) MarshalXML(e *xml.Encoder, start xml.StartElement) error {
	tokens := []xml.Token{}

	tokens = append(tokens, xml.StartElement{
		Name: xml.Name{"", "attendee"}})

	tokens = append(tokens, xml.StartElement{Name: xml.Name{"", "name"}})
	tokens = append(tokens, xml.CharData(a.Name))
	tokens = append(tokens, xml.EndElement{Name: xml.Name{"", "name"}})

	tokens = append(tokens, xml.StartElement{Name: xml.Name{"", "age"}})
	tokens = append(tokens, xml.CharData(strconv.Itoa(a.Age)))
	tokens = append(tokens, xml.EndElement{Name: xml.Name{"", "age"}})

	tokens = append(tokens, xml.StartElement{Name: xml.Name{"", "phone"}})
	tokens = append(tokens, xml.CharData(a.Phone))
	tokens = append(tokens, xml.EndElement{Name: xml.Name{"", "phone"}})

	tokens = append(tokens, xml.StartElement{Name: xml.Name{"", "website"}})
	tokens = append(tokens, xml.CharData("https://www.gophercon.com/speaker/"+a.Name))
	tokens = append(tokens, xml.EndElement{Name: xml.Name{"", "website"}})

	tokens = append(tokens, xml.EndElement{Name: xml.Name{"", "attendee"}})

	for _, t := range tokens {
		err := e.EncodeToken(t)
		if err != nil {
			return err
		}
	}

	err := e.Flush()
	if err != nil {
		return err
	}

	return nil
} 
```

在`attendee`包中，我们为`Attendee`类型实现了`MarshalXML`方法，进而实现了 xml 包的`Marshaler`接口。这样，当我们调用 xml 包的`Marshal`或`MarshalIndent`方法序列化上面`Attendee`实例时，我们实现的`MarshalXML`方法会被调用对`Attendee`实例进行 xml 编码。和默认的 xml 编码不同的是，在我们实现的`MarshalXML`方法中，我们会根据 Attendee 的 name 字段，自动在输出的 xml 格式数据中增加一个元素（element）: **website**。

下面我们就来为`Attendee`的`MarshalXML`方法编写测试：

```go
// testdata-demo1/attendee_test.go 
package attendee

import (
	"bytes"
	"encoding/xml"
	"io/ioutil"
	"path/filepath"
	"testing"
)

func TestAttendeeMarshal(t *testing.T) {
	tests := []struct {
		fileName string
		a        Attendee
	}{
		{
			fileName: "attendee1.xml",
			a: Attendee{
				Name:  "robpike",
				Age:   60,
				Phone: "13912345678",
			},
		},
	}

	for _, tt := range tests {
		got, err := xml.MarshalIndent(&tt.a, "", "  ")
		if err != nil {
			t.Fatalf("want nil, got %v", err)
		}

		want, err := ioutil.ReadFile(filepath.Join("testdata", tt.fileName))
		if err != nil {
			t.Fatalf("open file %s failed: %v", tt.fileName, err)
		}

		if !bytes.Equal(got, want) {
			t.Errorf("want %s, got %s", string(want), string(got))
		}
	}
} 
```

接下来，我们将预期结果放入`testdata/attendee1.xml`中：

```xml
//testdata/attendee1.xml 
<attendee>
  <name>robpike</name>
  <age>60</age>
  <phone>13912345678</phone>
  <website>https://www.gophercon.com/speaker/robpike</website>
</attendee> 
```

执行该测试：

```shell
$go test -v .
=== RUN   TestAttendeeMarshal
--- PASS: TestAttendeeMarshal (0.00s)
PASS
ok  	sources/testdata-demo1	0.007s 
```

测试通过是预料之中的事情。

## 2. golden 文件惯用法

在为上面例子准备预期结果数据文件：`attendee1.xml`时，你可能会有这样的问题：`attendee1.xml`中的数据从哪儿得到？

我们的确可以根据`Attendee`的`MarshalXML`方法的逻辑手工“造”出结果数据，但更快捷的方法是通过代码来得到预期结果。我们可以通过标准格式化函数输出对`Attendee`实例进行序列化后的结果。如果这个结果与我们的期望相符，那么这个结果就可以作为预期结果数据写入到`attendee1.xml`文件中：

```go
got, err := xml.MarshalIndent(&tt.a, "", "  ")
if err != nil {
	... ...
}
println(string(got)) // 这里输出xml编码后的结果数据 
```

如果仅是将标准输出中符合要求的预期结果数据手工拷贝到`attendee1.xml`文件中，那么标准输出中的不可见控制字符很可能会对最终拷贝的数据造成影响，从而导致测试失败。更有一些被测目标输出的是纯二进制数据，通过手工复制是无法实现预期>结果数据文件的制作的。因此，我们还是需要通过代码来实现`attendee1.xml`文件内容的填充，比如：

```go
got, err := xml.MarshalIndent(&tt.a, "", "  ")        
if err != nil {
	... ...
}  
ioutil.WriteFile("testdata/attendee1.xml", got, 0644) 
```

问题出现了！难道我们还要为每个`testdata`下面的预期结果文件单独编写一个小程序用于测试前写入预期数据？我们能否将采集预期数据到文件的过程与测试代码融合到一起呢？Go 标准库为我们提供了一种惯用法：**golden 文件**。

我们将上面的例子改造为采用**golden 文件**模式(将`attendee1.xml`重命名为`attendee1.golden`以显式告诉大家该测试用例采用了 golden 文件惯用法)：

```go
// testdata-demo2/attendee_test.go
... ...

var update = flag.Bool("update", false, "update .golden files")

func TestAttendeeMarshal(t *testing.T) {
	tests := []struct {
		fileName string
		a        Attendee
	}{
		{
			fileName: "attendee1.golden",
			a: Attendee{
				Name:  "robpike",
				Age:   60,
				Phone: "13912345678",
			},
		},
	}

	for _, tt := range tests {
		got, err := xml.MarshalIndent(&tt.a, "", "  ")
		if err != nil {
			t.Fatalf("want nil, got %v", err)
		}

		golden := filepath.Join("testdata", tt.fileName)
		if *update {
			ioutil.WriteFile(golden, got, 0644)
		}

		want, err := ioutil.ReadFile(golden)
		if err != nil {
			t.Fatalf("open file %s failed: %v", tt.fileName, err)
		}

		if !bytes.Equal(got, want) {
			t.Errorf("want %s, got %s", string(want), string(got))
		}
	}
} 
```

在改造后的测试代码中，我们看到新增了一个名为`update`的变量以及它所控制的 golden 文件的预期结果数据采集过程：

```go
if *update {
	ioutil.WriteFile(golden, got, 0644)
} 
```

这样，当我们执行下面命令时，测试代码会先将最新的预期结果写入`testdata`目录下的`golden文件`中，然后再将该结果与从`golden文件`中读出的结果做比较：

```shell
$go test -v . -update
=== RUN   TestAttendeeMarshal
--- PASS: TestAttendeeMarshal (0.00s)
PASS
ok  	sources/testdata-demo2	0.006s 
```

显然这样执行的测试是一定会通过的，因为在此次执行中，预期结果数据文件的内容就是通过被测函数刚刚生成的。

但带有`-update`命令参数的`go test`命令仅在需要进行预期结果数据采集时才会执行，尤其是在因数据生成逻辑发生变化或类型结构定义发生变化需要重新采集预期结果数据时。比如我们给上面的`Attendee`结构体类型增加一个新字段`topic`，如果不重新采集预期结果数据，那么测试一定是无法通过的。

采用**golden**文件惯用法后，要格外注意在每次重新采集预期结果后，对**golden 文件**中的数据进行正确性检查，否则很容易出现预期结果数据不正确，但测试依然通过的情况。

## 3. 小结

本节要点回顾：

- 面向工程的 Go 语言对测试依赖的外部数据文件的存放位置做了规范化，统一使用`testdata`目录；
- 开发人员可以采用将预期数据文件放在`testdata`下的方式为测试提供静态测试固件；
- golden 文件惯用法实现了`testdata`目录下测试依赖的预期结果数据文件的数据采集与测试代码的融合。