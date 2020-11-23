---
title: Go实现HTTPS服务器和客户端
date: 2020-11-20 16:44:38
tags:
- Protocol
- Go
---
HTTPS全称HTTP over TLS，是利用TLS对HTTP数据加密的通信方式。TLS是一种安全协议，在网络多层模型中介于TCP层和Application层之间，在TCP三次握手完成后进行TLS握手，握手目的是获取证书验证身份，双方协商加密算法，生成对称加密密钥，验证加密是否成功。
<!-- more -->

### Wireshark抓包分析
具体TLS握手分为以下几个步骤：
1. Client Hello
2. Server Hello
3. Certificate
4. Server Key Exchange （可选）
5. Server Done
6. Client Key Exchange
7. Change Cipher Spec
8. Encrypted Handshake Message
9. New Session Ticker
10. Change Cipher Sepc
11. Encrypted Handshake Message

通过抓取浏览器与[fancxxy's blog](blog.fancxxy.me)通信的TLS数据包来展示TLS握手过程。

#### ClientHello
<img src="client-hello.png">

ClientHello是TLS握手的第一个包，客户端向服务端请求加密协商，包含自身支持的TLS协议版本，支持的加密套件列表，生成的随机数，用于生成对称加密密钥。

#### ServerHello
<img src="server-hello.png">

服务端根据客户端提供的内容确定加密方式，此处是`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`。

`ECDHE_RSA`表示非对称加密算法是ECDH，此算法需要提供额外参数以及验证用的公钥，会在握手过程多一个步骤`Server key Exchange`，额外参数的签名算法是RSA。

`AES_128_GCM`表示对称加密算法是AES，密钥长度128，分组使用GCM。

`SHA256`表示计算消息流完整性的哈希算法。

同时服务端也生成了随机数，用于后续生成对称加密密钥。

#### Certificate
<img src="certificate.png">

加密算法协商之后，服务端发送自身数字证书以及签发该证书的证书链给客户端。客户端通过证书链找到根CA证书，根CA证书是内置在浏览器或操作系统中的，从根CA证书开始一级一级验证证书的合法性。

验证证书的过程，用证书中的公钥对密文解密，再用证书中的签名算法对证书内容签名，比较两者的结果是否相同。由于中间人无法获取CA机构的密钥，所以无法伪造证书做手脚。

#### Server Key Exchange, Server Hello Done
<img src="server-hello-done.png">

`Server Key Exchange`发送ECDH的参数以及自己的公钥给客户端。

`Server Hello Done`结束。

#### Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message
<img src="client-finished.png">

`Client Key Exchange`发送ECDH的参数以及自己的公钥给服务端。此时服务端和客户端都有了两个随机数、自己的私钥以及对方的公钥，可以各自在本地计算出相同的加密密钥，加密密钥就不用在网络传输了。

`Change Cipher Spec`通知服务端后续数据使用对称加密算法加密。

`Encrypted Handshake Message`客户端结束握手，发送整个流程中第一条密文数据给服务端解密。

#### New Session Ticket, Change Cipher Spec, Encrypted Handshake Message
<img src="server-finished.png">

`New Session Ticket`服务端建立类似Web应用的session机制，适当时机恢复会话用。

`Change Cipher Spec`通知客户端后续数据使用对称加密算法加密。

`Encrypted Handshake Message`服务端结束握手，发送密文数据给客户端解密，解密通过进行数据传输。

### openssl生成证书
自制CA证书。
```sh
# 生成CA证书ca.key
$ openssl genrsa -out ca.key 2048

# 生成根证书签发申请文件ca.csr
$ openssl req -new -key ca.key -subj "/CN=fancxxy" -out ca.csr

# 自签发根证书ca.crt
$ openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

生成服务端私钥并签发服务端证书。
```sh
# 生成服务端私钥server.key
$ openssl genrsa -out server.key 2048

# 生成服务端公钥server.pem
# $ openssl rsa -in server.key -pubout -out server.pem

# 服务端申请文件server.csr，/CN设置为localhost方便测试
$ openssl req -new -key server.key -subj "/CN=localhost" -out server.csr

# 向CA机构申请服务端证书server.crt
$ openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt
```

生成客户端私钥并签发客户端证书。
```sh
# 生成客户端私钥client.key
$ openssl genrsa -out client.key 2048

# 客户端申请文件client.csr
$ openssl req -new -key client.key -subj "/CN=localhost" -out client.csr

# 客户端证书client.crt
$ openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in client.csr -out client.crt
```

目录下文件。
```sh
$ tree certs
certs
├── ca.crt
├── ca.csr
├── ca.key
├── ca.srl
├── client.crt
├── client.csr
├── client.key
├── server.crt
├── server.csr
└── server.key

0 directories, 10 files
```

### Go实现HTTPS认证
#### 单向认证
服务端使用`Server.ListenAndServeTLS`方法导入证书和私钥。
```go
package main

import (
	"net/http"
)

func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("pong"))
	})

	server := &http.Server{
		Addr:    ":443",
		Handler: mux,
	}

	server.ListenAndServeTLS("../certs/server.crt", "../certs/server.key")
}
```

客户端使用`x509.NewCertPool`创建证书容器，`CertPool.AppendCertsFromPEM`导入自建CA根证书。
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"

	"golang.org/x/net/http2"
)

func main() {
	caCert, err := ioutil.ReadFile("../certs/ca.crt")
	if err != nil {
		panic(err)
	}

	caPool := x509.NewCertPool()
	caPool.AppendCertsFromPEM(caCert)

	tlsConfig := &tls.Config{
		RootCAs: caPool,
	}

	client := http.Client{
		Transport: &http2.Transport{
			TLSClientConfig: tlsConfig,
		},
	}

	resp, err := client.Get(os.Args[1])
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	bs, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		panic(err)
	}

	fmt.Printf("%s\n", bs)
}
```

启动服务端和客户端程序，客户端执行时需要设置环境变量` GODEBUG=x509ignoreCN=0`。
```sh
$ GODEBUG=x509ignoreCN=0 go run main.go https://localhost:443/ping
pong
```

#### 双向认证
服务端有时候也需要验证客户端身份，客户端身份是CA机构签发，所以服务端需要读取根CA证书。读取方法和单项认证的客户端一样。
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"io/ioutil"
	"net/http"
)

func main() {
	caCert, err := ioutil.ReadFile("../certs/ca.crt")
	if err != nil {
		panic(err)
	}

	caPool := x509.NewCertPool()
	caPool.AppendCertsFromPEM(caCert)

	tlsConfig := &tls.Config{
		ClientCAs:  caPool,
		ClientAuth: tls.RequireAndVerifyClientCert,
	}

	mux := http.NewServeMux()
	mux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("pong"))
	})

	server := &http.Server{
		Addr:      ":443",
		Handler:   mux,
		TLSConfig: tlsConfig,
	}

	server.ListenAndServeTLS("../certs/server.crt", "../certs/server.key")
}
```

使用单项认证的客户端代码直接执行可以看到客户端返回错误`tls: bad certificate`，服务端返回错误`tls: client didn't provide a certificate`。
```sh
$ GODEBUG=x509ignoreCN=0 go run main.go https://localhost:443/ping
panic: Get "https://localhost:443/ping": remote error: tls: bad certificate

goroutine 1 [running]:
main.main()
	/Users/fancxxy/Works/test/http/client/main.go:33 +0x427
exit status 2

$ go run main.go
2020/11/20 16:25:22 http: TLS handshake error from [::1]:49995: tls: client didn't provide a certificate
```

客户端提供认证证书，需要`tls.LoadX509KeyPair`方法加载自己的私钥和公钥并设置到`tls.Config`。
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func main() {
	caCert, err := ioutil.ReadFile("../certs/ca.crt")
	if err != nil {
		panic(err)
	}

	caPool := x509.NewCertPool()
	caPool.AppendCertsFromPEM(caCert)

	cliCert, err := tls.LoadX509KeyPair("../certs/client.crt", "../certs/client.key")
	if err != nil {
		panic(err)
	}

	tlsConfig := &tls.Config{
		RootCAs:      caPool,
		Certificates: []tls.Certificate{cliCert},
	}

	client := http.Client{
		Transport: &http.Transport{
			TLSClientConfig: tlsConfig,
		},
	}

	resp, err := client.Get(os.Args[1])
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	bs, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		panic(err)
	}

	fmt.Printf("%s\n", bs)
}
```

重新直接客户端代码能正确返回结果。
```sh
$ GODEBUG=x509ignoreCN=0 go run main.go https://localhost:443/ping
pong
```
