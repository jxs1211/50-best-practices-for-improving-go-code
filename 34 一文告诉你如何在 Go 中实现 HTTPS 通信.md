34 一文告诉你如何在 Go 中实现 HTTPS 通信

## 一文告诉你如何在 Go 中实现 HTTPS 通信

在 [2019 年 Go 官方开发者调查](https://blog.golang.org/survey2019-results)的 “Go 语言应用领域” 这一调查项的结果中，**Web 开发**以 **66%** 的比例位居第一。Go 在 Web 开发领域的广泛应用得益于 Go 标准库内置了 `net/http` 包，使用该包我们用十几行代码就能快速实现一个 “Hello, World!” 级别的 Web 服务：

```go
// go-https/hello_world_server.go 
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, World!\n")
	})
	http.ListenAndServe("localhost:8080", nil)
} 
```

基于 `net/http` 包较高的开发效率以及 Go 程序优异的执行性能，Go Web 开发为 Go 社区吸引了大量来自 PHP、Python、Ruby 等语言阵营的开发者。

HTTP 协议是目前 Web 服务中使用**最广泛**的应用层协议。不过 HTTP 协议是采用**明文**传输的，在这样一个不安全的互联网世界里，采用 HTTP 协议传输的数据存在如下风险：

- 使用不加密的明文进行通信，内容可能会被窃听；
- 不验证通信方的身份，可能遭遇伪装；
- 无法证明报文的完整性，内容可能遭篡改。

因此 HTTP 协议显然不能满足现在站点或服务日益提高的对安全性的要求。在这一节中，我就和大家一起来看看基于标准库的 `net/http` 包如何实现互联网上的安全通信。

### 1. HTTPS：在安全传输层上运行的 HTTP 协议

我们运行一下上面 `hello_world_server.go` 这个例子：

```go
$go run hello_world_server.go 
```

然后，我们打开 Chrome 浏览器，在地址栏中输入 `http://localhost:8080` 访问我们上面启动的 HTTP 服务，返回结果见下面截图：

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/c298d7ff615bde08ef8d4c29aaa6df0a.png)

图 9-1-1：使用 HTTP 协议访问 Web 服务

我们看到浏览器页面正常显示了 Web 服务返回的 “Hello,World!” 内容。但 Chrome 浏览器在紧邻地址栏左侧的安全提示标识框中使用了ⓘ标识。点击该标识，我们看到浏览器给出了安全提示：**浏览器与 Web 服务之间建立的连接是不安全的**。

我们再使用该浏览器访问一下 `github` 站点的 [Go 项目主页](https://github.com/golang/go)，下面是返回结果的页面截图：

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/f8e0fcd0dd3b5d10474cbdeb10ac0b15.png)

图 9-1-2：使用 HTTPS 协议访问 Web 服务

我们看到和上一幅图不同的是，访问 `github` 站点后，Chrome 浏览器的安全提示标识框中是一把 “小锁头”，这表明浏览器与 `github` 站点之间建立的连接是**安全**的。而这条安全的连接就是建构在 **HTTPS 协议**之上的。

**HTTPS 协议**就是用来解决传统 HTTP 协议明文传输不安全的问题的。和普通 HTTP 协议的不同之处在于 HTTPS 协议在传输层 (TCP) 和应用层 (HTTP) 之间增加了一个 **“安全传输层”**。如下图所示：

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/2fe33f5c988706d693d317ba9ef70464.png)

图 9-1-3：HTTP 和 HTTPS 的对比

从图中我们看到：采用 HTTPS 协议后，原先网络协议栈上的应用层 (HTTP) 直接和传输层 (TCP) 通信变成了应用层 (HTTP) 与**安全传输层**通信。**安全传输层**通常采用 SSL (Secure Socket Layer) 或 TLS (Transport Layer Security) 协议实现 (Go 标准库支持 TLS[1.3 版本协议](https://tip.golang.org/doc/go1.13#tls_1_3))，这一层负责 HTTP 协议传输的内容加密、通信双方身份验证等。有了这一层后，HTTP 协议就摇身一变，成为了拥有加密、证书身份验证和内容完整性保护功能的 HTTPS 协议了。或者反过来说：**HTTPS 协议就是在安全传输层上运行的 HTTP 协议**。

Go 标准库 `net/http` 包同样提供了对采用 HTTPS 协议的 Web 服务的支持。我们修改一行代码就能将上面示例中的那个基于 HTTP 协议的 Web 服务改为一个采用 HTTPS 协议的 Web 服务:

```go
// go-https/https_hello_world_server.go 
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, World!\n")
	})
	fmt.Println(http.ListenAndServeTLS("localhost:8081", "server.crt", "server.key", nil))
} 
```

在这个示例中，我们仅是用 `http.ListenAndServeTLS` 函数替换掉了 `http.ListenAndServe` 就实现了将一个 HTTP Web 服务转换为一个 HTTPS Web 服务的目的。不过与 `ListenAndServe` 不同的是，`ListenAndServeTLS` 这个函数新增了两个参数 `certFile` 和 `keyFile`：

```go
// $GOROOT/src/net/http/server.go
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error 
```



`certFile` 和 `keyFile` 两个参数是 `http` 包针对 HTTPS 协议进行内容加密、身份验证和内容完整性验证的前提。我们利用 `[openssl]()https://github.com/openssl/openssl` 工具可以生成该示例中 HTTPS Web 服务所需的 `certFile` 和 `keyFile` 并让这个示例中的服务运行起来。关于这两个参数的功能原理在后续会有详细说明。

```shell
$openssl genrsa -out server.key 2048
Generating RSA private key, 2048 bit long modulus
...........................................+++
...+++
e is 65537 (0x10001)

$openssl req -new -x509 -key server.key -out server.crt -days 365
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Organizational Unit Name (eg, section) []:
Common Name (eg, fully qualified host name) []:localhost
Email Address []: 
```

执行该示例程序：

```go
$go run https_hello_world_server.go 
```

通过 Chrome 浏览器访问 `https://localhost:8081`，浏览器会显示如下⚠️警告页面：

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/6640310b01ec854bf73128229e876c82.png)

图 9-1-4：访问示例 HTTPS Web 服务，浏览器给出的警告页面

点击页面上的 “高级”，选择 “继续前往不安全页面” 后，我们期望的 **“Hello,World!” 字样** 的结果才会显示在结果页面上。

我们也可以使用 `curl` 工具验证这个 HTTPS Web 服务：

```shell
$curl -k https://localhost:8081
Hello, World! 
```

注意：如果不加 `-k` 参数，`curl` 会报如下错误：

```shell
$curl https://localhost:8081 
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
HTTPS-proxy has similar options --proxy-cacert and --proxy-insecure. 
```

从 `curl` 的错误输出来看，报错的主要原因是因为示例中 HTTPS Web 服务所使用证书 (`server.crt`) 是我们自己生成的自签名证书 (`self signed certificate`)，`curl` 使用测试环境系统中内置的各种数字证书授权机构 (Certificate Authority，CA) 的公钥证书无法对其进行验证。而 `-k` 选型则是表示忽略对示例中 HTTPS Web 服务的证书的校验。

### 2. HTTPS 安全传输层的工作机制

HTTPS 是建构在基于 SSL/TLS 协议实现的传输安全层之上的 HTTP 协议，也就是说一旦通信双方在传输安全层上将连接成功建立起来，那么后续的通信就和普通 HTTP 协议的一样了，只不过所有的 HTTP 协议数据经过安全层传输时都会被自动加密 / 解密罢了。

安全传输层是整个 HTTPS 协议的核心，了解安全传输层的工作机制对理解 HTTPS 协议至关重要。下面我们就来看看 HTTPS 的安全传输层是如何建立连接并进行数据通信的。

为了探究安全传输层连接的建立过程，我们通过 `curl` 命令再次访问上面的 HTTPS Web 示例服务，不过这次我们加上了 `-v` 参数，让 `curl` 输出更为详细的日志：

```shell
$curl -v -k https://127.0.0.1:8081
* Rebuilt URL to: https://127.0.0.1:8081/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8081 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
... ... 
```

我们根据上述 `curl` 命令 (作为客户端) 与 `https_hello_world_server`(作为服务端) 的通信过程归纳到一幅示意图中：

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/f0cf81d8655f240c1cddb2fb54d81e81.png)

图 9-1-5：HTTPS 安全传输层建立连接的过程

安全传输层连接建立的过程也称为 **“握手阶段 (handshake)”**。从示意图中可以看出，这个握手阶段也涉及四轮通信，我们逐一简单说明一下：

- ClientHello (客户端 -> 服务端)

客户端向服务端发出建立安全层传输连接，构建加密通信通道的请求。在这个请求中，客户端会向服务端提供本地最新 TLS 版本、支持的加密算法组合的集合（比如上面 `curl` 示例所建立的安全层传输会话最终选择的 `ECDHE-RSA-AES128-GCM-SHA256` 组合) 以及随机数等。

- ServerHello & Server certificate &ServerKeyExchange (服务端 -> 客户端)

这一轮通信也分为三个重要步骤：

首先，服务端收到客户端发过来的 `ClientHello` 请求后，用客户端发来的信息与自己本地支持的 TLS 版本、加密算法组合集合作比较，选出一个 TLS 版本和一个合适的加密算法组合，然后同样也生成一个随机数，一起打包到 `ServerHello` 中返回给客户端。

接下来，服务器会将自己的**服务端公钥证书**发送给客户端 (`Server certificate`)，这个服务端公钥证书身兼两大职责：客户端对服务端身份的验证以及后续双方会话密钥的协商和生成。

如果服务端要验证客户端身份 (可选的)，那么这里服务端还会发送一个 `CertificateRequest` 的请求给客户端，要求对客户端的公钥证书进行验证。

第三个步骤则是发送开启双方会话密钥协商的请求 (`ServerKeyExchange`)。相对于非对称加密算法，对称加密算法性能要高出几个数量级，因此 HTTPS 在开始真正传输应用层的用户数据之前，选择了在非对称加密算法的帮助下协商一个基于对称加密算法的密钥。在密钥协商环节，通常会使用到 `Diffie-Hellman（DH）密钥交换算法`，这是一种密钥协商的协议，它支持通信双方在不安全的通道上生成对称加密算法所需的共享密钥。因此，在这个步骤的请求中，服务端会向客户端发送密钥交换算法的相关参数信息给客户端。

最后，服务端以 `Server Finished(又称为ServerDone)` 作为此轮通信的结束标志。

- ClientKeyExchange & ClientChangeCipher & Finished (客户端 -> 服务端)

客户端在收到服务端的公钥证书后会对服务端的身份进行验证（当然也可以选择不验证），如果验证失败，则此次安全传输层连接建立就会以失败告终。如果验证通过，那么客户端将从证书中提取出服务端的公钥，用于加密后续协商密钥时发送给服务端的信息。

如果服务端要求对客户端进行身份验证 (即接到服务端发送的 `CertificateRequest` 请求)，那么客户端还需通过 `ClientCertificate` 将自己的公钥证书发给服务端进行验证。

收到服务端对称加密共享密钥协商的请求后，客户端根据之前的随机数、确定的加密算法组合以及服务端发来的参数计算出最终的会话密钥，然后将服务端单独计算出会话密钥所需的信息用服务端的公钥加密后以 `ClientKeyExchange` 请求发给服务端。

随后客户端用 `ClientChangeCipher` 通知服务端从现在开始发送的消息都是加密过的。

最后，伴随着 `ClientChangeCipher` 消息总会有一个 `Finished` 消息来验证双方的对称加密共享密钥协商是否成功。其验证的方法就是通过协商好的新共享密钥和对称加密算法对一段特定内容进行加密，并以服务端是否能够正确解密该请求报文作为密钥协商成功与否的判定标准。而被加密的这段特定内容包含的是连接至今的全部报文内容。`Finished` 报文也作为此轮通信的结束标志，也是客户端发出的第一条使用协商密钥加密的信息。

- ServerChangeCipher & Finished (服务端 -> 客户端)

服务端收到客户端发过来的 `ClientKeyExchange` 中的参数后，也将单独计算出会话密钥。之后和客户端一样，服务端用 `ServerChangeCipher` 通知客户端从现在开始发送的消息都是加密过的。

最后，服务端用一个 `Finished` 消息跟在 `ServerChangeCipher` 后面，既用于标识此轮握手结束，也用于来验证对方计算出来的共享密钥是否有效。这也是服务端发出的第一条使用协商密钥加密的信息。

一旦 HTTPS 安全传输层的连接成功建立起来，后续双方通信的内容 (即应用层的 HTTP 协议) 就会在一个经过加密处理的**安全通道**中得以传输。

### 3. 非对称加密和公钥证书

在上面 HTTPS 安全传输层连接的建立过程中，服务端的公钥证书在验证服务端身份以及辅助对称加密算法共享密钥的协商和生成方面起到了关键的作用。

公钥证书 (public-key certificate) 是非对称加密体系的重要内容。所谓**非对称加密体系**，又称公钥加密体系，是和我们熟知的对称加密体系所对应的。

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/4eeae6ebde6cca6bd4fd5a907c474b8f.png)

图 9-1-6：对称加密与非对称加密的对比

如上图所示我们看到：**对称加密**指的是通信双方使用一个**共享密钥 (key)**，该密钥既用于传输数据的加密（发送方），同样也用于数据的解密（接收方）；而**非对称加密**则指通信的每一方都有两个密钥，一个公钥 (public key)，一个私钥 (private key)。通信的发送方 (如图中的 A) 使用对方 (如图中 B) 的公钥对数据进行加密，数据接收方 (如图中 B) 使用自己的私钥对数据进行解密。

对称加密和非对称加密**各有优缺点**。对称加密性能好，但密钥的保存、管理和分发存在较大安全风险；而非对称加密就是为了解决对称加密的密钥分发安全隐患而设计的。在非对称加密体系中，任何参与通信的一方都有两把密钥，一把是需要自己保存好的私钥，一把则是**对外公开的**、用于加密的公钥。以上图中的 A 为例，任何想与 A 通信的另一方都可以获取到 A 的公钥，并将通过这把公钥加密的消息发给 A。A 使用自己的私钥可以对收到的消息进行解密。由于公钥是公开的，因此其分发的安全风险显然要比对称加密低很多。不过，非对称加密的性能相较于对称加密要差上很多，这也是为何在实际应用中（比如 HTTPS 的传输安全层）会将两种加密方式结合使用的原因。

非对称加密体系的公钥是对外公开的，这大大降低了密钥分发的复杂性。但直接分发公钥信息仍然可能存在安全隐患。比如上面 HTTPS 协议安全传输层连接建立的过程中，如何保证 HTTPS 服务端发送给客户端的公钥信息没有被篡改呢？我们也看到了 HTTPS 建立连接的过程并非直接传输公钥信息，而是使用携带公钥信息的**数字证书**来保证公钥信息的正确性和完整性。

**数字证书**，被称为互联网上的 “身份证”，用于唯一标识网络上的一个域名地址或一个服务器主机的，这就好比我们日常生活中使用的 “居民身份证”，用于唯一标识一个公民。服务端将包含公钥信息的数字证书传输给客户端，客户端如何校验这个证书的真伪呢？我们知道居民身份证是由国家统一制作和颁发的，个体公民向户口所在地公安机关申请办理。只有国家颁发的身份证才是具有法律 效力的，在中国大陆任何地方这个身份证都是有效和可被接纳的。大悦城的会员卡也是一种身份标识，但如果你用大悦城的会员卡去买机票，对不起！不卖。航空公司可不认大悦城的会员卡，只认居民身份证，网站的证书也是同样的道理。一般来说数字证书是从受信的权威证书授权机构 (`Certification Authority`，证书授权机构) 买来的（当然也有免费的，不过免费的很少）。一般浏览器或操作系统在出厂时就内置了诸多知名 CA（如 `Verisign`、`GoDaddy`、`CNNIC` 等）的公钥数字证书，这些 CA 公钥证书可以用于验证这些 CA 机构为网站颁发的公钥证书。对于这些内置 CA 公钥证书无法识别的证书，浏览器就会报错，就像上面 Chrome 浏览器针对我们的那个自签名证书报错的截图那样。

那么 **使用 CA 的公钥证书是如何校验服务端公钥证书的有效性的呢？** 这就涉及到数字公钥证书到底是什么了！

我们可以通过浏览器中的 “https/ssl 证书管理” 来查看证书的内容。一般公钥证书都会包含诸如站点的名称、主机名、公钥、证书签发机构 (CA) 名称和来自签发机构的签名等。我们重点关注这个来自签发机构的签名，因为对于公钥证书的校验方法就是使用本地 CA 公钥证书来验证来自通信对端的公钥证书中的签名是否是这个 CA 签的。

下面我们就来看看公钥证书的申请与验证的过程，请看下面这幅示意图：

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/b7158eef0f4eae1ed3b4f9ba81bf38b0.png)

图 9-1-7：公钥证书申请与验证的过程

图中展示了 “公钥证书申请” 与 “公钥证书验证” 两个过程，我们分别来说一下。

图的右上半部分展示的是 “公钥证书申请” 过程。如果某个互联网服务站点要启用 HTTPS 访问，那么这个站点就需要向数字证书授权签发机构 (CA) 提交数字证书申请请求。这个请求以证书签名请求 (Certificate Signing Request, csr) 文件的形式提供。我们通过 `openssl` 命令可以基于申请者的私钥生成证书签名请求文件。我们可以使用上面例子中的 `server.key` 来生成一个 `server.csr` 并查看其内容：

```shell
$openssl req -new -key server.key -subj "/CN=localhost" -out server.csr

$openssl req -in server.csr -noout -text
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: CN=localhost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b5:84:83:d3:10:48:fa:da:cd:dd:b4:5e:c8:47:
                    ... ...
                    48:c0:8f:e7:99:3b:1a:05:db:61:79:7e:7f:4b:33:
                    e9:f5
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    Signature Algorithm: sha256WithRSAEncryption
         29:70:cc:aa:0f:3d:88:55:88:73:d6:03:07:e1:6d:18:f8:ba:
         ... ...
         ae:b1:34:b7:dc:7d:5b:1c:d1:1e:12:71:9f:ab:ff:aa:62:56:
         b4:bf:b2:29 
```

我们看到 `server.csr` 中内包含了证书申请人的信息，比如：国家、邮件、域名 (这里仅有域名，是 `localhost`) 等，另外一个重要信息就是申请者的公钥信息。

证书授权签发机构 (CA) 收到客户的证书申请后，会按照标准数字证书规范生成该申请人的数字公钥证书。我们用示例来演示一下 CA 机构签发证书的过程。首先我们来创建一个模拟 CA 机构。CA 机构的核心就是一个**私钥**以及由该私钥自签名的 **CA 公钥证书** (内置到操作系统和浏览器中分发)。这里我们通过 `openssl` 命令创建 CA 私钥 (`ca.key`) 以及其公钥证书 (`ca.crt`)：

```shell
$openssl genrsa -out ca.key 2048
Generating RSA private key, 2048 bit long modulus
.....................+++
....................+++
e is 65537 (0x10001)

$openssl req -x509 -new -nodes -key ca.key -subj "/CN=myca.com" -days 5000 -out ca.crt 
```

接下来，我们就可以用 `ca.key` 和 `ca.crt` 处理前面提交的数字证书申请请求 (`server.csr`) 了：

```shell
$openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server-signed-by-ca.crt -days 5000 
Signature ok
subject=/CN=localhost
Getting CA Private Key 
```

生成的 `server-signed-by-ca.crt` 就是我们的 CA 为上面示例中的服务端 (localhost) 创建的公钥证书。这个证书格式符合 X509 数字证书基本规范。X509 证书由公钥和用户标识符组成，此外还包括版本号、证书序列号、CA 标识符、签名算法标识、签发者名称、证书有效期等信息。我们可以通过 `openssl` 命令查看生成的公钥证书文件：

```shell
$openssl x509 -in server-signed-by-ca.crt -noout -text 
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 12186770843341339017 (0xa9201b838472b989)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: CN=myca.com
        Validity
            Not Before: Jun 10 14:24:39 2020 GMT
            Not After : Feb 17 14:24:39 2034 GMT
        Subject: CN=localhost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b5:84:83:d3:10:48:fa:da:cd:dd:b4:5e:c8:47:
                    ... ...
                    48:c0:8f:e7:99:3b:1a:05:db:61:79:7e:7f:4b:33:
                    e9:f5
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha1WithRSAEncryption
         05:e6:58:ff:94:89:f6:ea:05:ee:1a:2a:55:d8:0c:0c:2e:66:
         ... ...
         d7:b9:43:af:78:b0:2b:be:30:00:a0:49:a3:db:bb:c7:48:a5:
         1f:f5:a6:89 
```

接下来我们来说一下：客户端是如何通过内置的 `ca.crt` 对服务端下发的 “server.crt” 进行验证的。从上面 `server-signed-by-ca.crt` 的内容来看，其主要包含三个部分：

- 服务端公钥 (server.pub)
- 证书相关属性信息：域名、有效期等
- 证书颁发机构的签名信息

在这三部分信息中，**证书颁发机构的签名信息**在验证证书环节起到至关重要的作用。我们知道在非对称加密体系中，我们使用公钥对数据进行加密，使用私钥对数据进行解密。私钥和公钥还有另外一种重要功能，那就是**验证信息来源与保证数据完整性**。而这一功能也恰**被用在对通信对方的公钥证书的验证上了**。当一对私钥和公钥被用于验证信息来源以及保证数据完整性时，我们**使用私钥对数据 (或数据摘要) 进行签名 (加密)**，然后**用公钥对签过名的数据进行校验 (解密)**。这样一个签名和校验的过程，我们同样可以用 `openssl` 工具来演示一下：

我们创建一个待签名的数据文件 `hello.txt`，然后使用 `ca.key` 对其进行签名，生成签名后的数据文件 `hello.signed`，然后再使用 `ca.crt` 对 `hello.signed` 进行验证 (解密)，验证后的结果文件为 `hello.verify`。如果一切顺利，`hello.verify` 应该与 `hello.txt` 一模一样：

```shell
$echo "hello,world" > hello.txt

$openssl rsautl -sign -in hello.txt -inkey ca.key -out hello.signed 
$openssl rsautl -verify -in hello.signed -inkey ca.crt -certin -out hello.verify 

$cat hello.verify 
hello,world 
```

我们也可以直接使用 `openssl` 命令来查看 `ca.crt` 对 `server-signed-by-ca.crt` 的验证结果：

```shell
$openssl verify -CAfile ca.crt server-signed-by-ca.crt 
server-signed-by-ca.crt: OK 
```

`server-signed-by-ca.crt` 公钥证书中的签名信息的由来与 `hello.signed` 大同小异。公钥证书中的签名信息就是使用 `ca.key` 对证书中公钥信息与证书属性信息的摘要信息 (这里使用 sha1 算法制作摘要) 进行加密而得来的：

```shell
d = digest(server.pub, certificate info) // server.pub为公钥信息
sign = encrypt_with_ca_key(d)  // 使用ca.key对d进行加密(即签名) 
```

这样当客户端使用 `ca.crt` 对服务端发来的公钥证书进行验证时，客户端会直接使用 `ca.crt` 中的公钥对公钥证书中的签名信息进行解密：

```shell
d' = decrypt_with_ca_crt(sign) 
```

然后用解密得到的 `d'` 与使用相同摘要算法对证书中公钥信息与证书属性信息进行摘要计算后的结果 `d` 进行比较，如果一致，则说明证书验证通过；否则，证书验证失败。一旦签名验证通过，我们因为信任这个 CA (公钥证书)，从而信任这个服务端证书。由此也可以看出，CA 机构的最大资本就是其信用度。我们看到通过对证书签名信息的验证可以保证证书内容未被中途篡改，同时也确定了证书归属（可与我们访问的站点对比，一致说明是安全的）。

### 4. 对服务端公钥证书的验证

现在我们有了 CA 的公钥证书 `ca.crt`，也有了 CA 签发的服务端证书 `server-signed-by-ca.crt`，我们就来用 Go 实现一下客户端对服务端公钥证书的验证。

我们创建一个新的 HTTPS Web 服务，该服务使用我们通过 CA 新签发的证书：`server-signed-by-ca.crt`：

```go
// go-https/verify-server-cert/hello_world_server.go
... ...
func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, World!\n")
	})
	fmt.Println(http.ListenAndServeTLS("localhost:8081",
		"../server-signed-by-ca.crt",
		"../server.key", nil))
} 
```

启动该服务端：

```shell
$ cd go-https/verify-server-cert
$go run hello_world_server.go 
```

现在我们创建一个客户端，尝试访问上述的服务端：

```go
// go-https/verify-server-cert/client_without_cacert.go 
... ...
func main() {
	resp, err := http.Get("https://localhost:8081")
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	fmt.Println(string(body))
} 
```

在这一版客户端里，我们没有定制客户端，而是直接使用了 `net/http` 包的 `Get` 函数尝试去访问上面的服务端。这样实现的客户端默认情况下会对服务端的证书进行验证。我们运行该客户端：

```go
$go run client_without_cacert.go 
error: Get "https://localhost:8081": x509: certificate signed by unknown authority 
```

我们看到：客户端输出了一段错误日志。这段日志的意思是**服务端发过来的服务端公钥证书的签发者是一个未知的 CA**。也就是说客户端在对服务端证书进行校验时，在本地环境中没有找到签发该证书的那个 CA。

如果客户端信任这个服务端，可以忽略对服务端证书的校验：

```go
// go-https/verify-server-cert/client_skip_verify.go
... ...
func main() {
	tr := &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}
	client := &http.Client{Transport: tr}
	resp, err := client.Get("https://localhost:8081")

	if err != nil {
		fmt.Println("error:", err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	fmt.Println(string(body))
} 
```

在这第二版的客户端代码中，我们没有再直接使用 `http.Get`，而是定义了一个 `http.Client` 类型的变量 `client` 用于代表客户端。在初始化该 `client` 的结构时，我们将其内部的 `Transport` 字段设置为一个忽略证书检查的 `http.Transport` 实例。这样我们运行这个客户端就可以成功得到服务端的应答数据：

```shell
$go run client_skip_verify.go 
Hello, World! 
```

不过大多数时候，我们是要对服务端证书进行验证的，这时我们就需要让客户端知晓并加载 CA 的公钥证书，下面的代码演示了如何在客户端加载 CA 公钥证书：

```go
// go-https/verify-server-cert/client_verify_by_cacert.go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	pool := x509.NewCertPool()
	caCertPath := "../ca.crt"

	caCrt, err := ioutil.ReadFile(caCertPath)
	if err != nil {
		fmt.Println("ReadFile err:", err)
		return
	}
	pool.AppendCertsFromPEM(caCrt)

	tr := &http.Transport{
		TLSClientConfig: &tls.Config{RootCAs: pool},
	}
	client := &http.Client{Transport: tr}
	resp, err := client.Get("https://localhost:8081")
	if err != nil {
		fmt.Println("Get error:", err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	fmt.Println(string(body))
} 
```

上面这版客户端代码创建了一个 `x509.CertPool` 实例 (公钥证书池)，并将我们的 CA 公钥证书添加到该 `pool` 中，然后将这个公钥证书池实例赋值给我们新建的 `http.Client` 实例的 `TLSClientConfig` 属性的 `RootCAs` 字段。

我们运行这个版本的客户端：

```go
$go run client_verify_by_cacert.go 
Hello, World! 
```

我们看到客户端使用 CA 的公钥证书成功对服务端发送的公钥证书进行了验证并输出了服务端的应答结果。

我们也可以将自签名的 CA 公钥证书导入到系统 CA 证书存储目录下。在 MacOS 下，我们可以使用 “钥匙串访问” 将 `ca.crt` 导入。导入后，在 “信任” -> “使用此证书时” 的下拉选项中选择：“始终信任”。

![34 一文告诉你如何在 Go 中实现 HTTPS 通信](https://img-hello-world.oss-cn-beijing.aliyuncs.com/fe3968fe524d5a279e6452a971735a7d.png)

图 9-1-8：MacOS 下导入的 CA 公钥证书

这样，我们就无需再显式地给客户端传入 `ca.crt` 了。我们再运行一下使用默认 `http.Get` 函数访问服务端的那一版客户端实现：

```go
$cd go-https/verify-server-cert
$go run client_without_cacert.go
Hello, World! 
```

这次，`client_without_cacert` 使用了上面导入的、位于系统默认位置的 `ca.crt` 对服务端的证书做了成功的验证，并顺利打印出服务端的返回应答。

### 5. 小结

在本小节中，我们了解了如何利用 Go 标准库提供的 `net/http`、`crypto/tls` 以及 `crypto/x509` 等包建立一条安全的 HTTPS 协议通信通道。下面是本节要点：

- 了解 HTTP 协议的优点与不足；
- 了解 HTTPS 协议安全传输层的建立过程；
- 理解非对称加密体系以及数字证书的组成与功用；
- 数字证书就是使用 CA 私钥 (key) 对证书申请者的公钥和证书相关信息进行签名 (加密) 后的满足标准证书格式的信息。