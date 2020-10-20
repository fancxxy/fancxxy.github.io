---
title: ICMP监控
date: 2020-09-09 14:37:37
tags: 
- Protocol
- Go
---

`ping`命令可以用来探测对端机器的网络连通状况以及网络延迟时间。当只有单台机器时可以用`ping`命令来探测，但如果有几千台机器就不合适了，所以我们利用`ping`命令的原理写了一个支持同时探测上千台机器网络连通性的程序。
<!--more-->

### ICMP Protocol
首先得了解ICMP协议，ICMP中文名是互联网控制报文协议，全称Internet Control Message Protocol，主要作用就是传输网络诊断信息。

ICMP报文分两类，分别是差错报文和询问报文，监控网络连通性用的是询问报文的Echo Request和Echo Reply，报文结构如下，传输时被封装在IP数据报中，但通常还是认为ICMP是网络层协议，目的机器收到Echo Request后会回应一个Echo Reply，内容和请求一模一样。
```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |     Type      |     Code      |          Checksum             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Identifier          |        Sequence Number        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |     Data ...
   +-+-+-+-+-
```

* Type: 报文的类型，占用一个字节，ICMPv4的Echo Request和Echo Reply对应的Type分别是8和0，ICMPv6的对应值是128和129。

* Code: 占用一个字节，请求和应答的Code都是0。

* Checksum: 占用两个字节，校验和用来校验数据包的完整性。

* Identifier: 占用两个字节，在ping命令中用的本进程的PID来赋值的，用来区分是哪个ping进程发出的请求。因为如果有多个ping进程同时运行，会从底层原始套接字读到对方进程的请求回包，根据这个字段判断如果不是自己发出的就丢弃。

* Sequence Number: 占用两个字节，给每个发出的请求打上序列号，由于响应内容和请求内容一样所以也会包含相同序列号，用来关联请求和响应。

### ICMP package
`golang.org/x/net/icmp`包已经实现了ICMPv4和ICMPv6的基础功能，可以使用Echo相关的代码。

`Message`是ICMP消息的结构体，其中`MessageBody`是一个接口类型，`Echo`实现了该接口。
```go
type Message struct {
    Type     Type        // type, either ipv4.ICMPType or ipv6.ICMPType
    Code     int         // code
    Checksum int         // checksum
    Body     MessageBody // body
}

type Echo struct {
    ID   int    // identifier
    Seq  int    // sequence number
    Data []byte // data
}
```

`ListenPacket`方法返回一个监听原始套接字的连接，通过这个连接可以发送和读取ICMP数据包（监听原始套接字需要root权限）。
```go
func ListenPacket(network, address string) (*PacketConn, error)
```

如果想要同时监控ICMPv4和ICMPv6，就需要初始化两个连接。
```go
ListenPacket("ip4:icmp", "0.0.0.0")
ListenPacket("ip6:ipv6-icmp", "::")
```

利用上面提到的连接和结构体实现一个最简单的发送Echo Request和接收Echo Reply的程序。
```go
package main

import (
	"errors"
	"fmt"
	"net"
	"os"

	"golang.org/x/net/icmp"
	"golang.org/x/net/ipv4"
)

func send(conn *icmp.PacketConn, ip string) error {
	dest, err := net.ResolveIPAddr("ip", ip)
	if err != nil {
		return err
	}

	msg := &icmp.Message{
		Code: 0,
		Type: ipv4.ICMPTypeEcho,
		Body: &icmp.Echo{
			ID:   os.Getpid(),
			Seq:  2,
			Data: []byte("hello icmp"),
		},
	}

	wb, err := msg.Marshal(nil)
	if err != nil {
		return err
	}

	_, err = conn.WriteTo(wb, dest)
	if err != nil {
		return err
	}

	return nil
}

func receive(conn *icmp.PacketConn) (*icmp.Message, *icmp.Echo, error) {
	buf := make([]byte, 64)
	n, _, err := conn.ReadFrom(buf)
	if err != nil {
		return nil, nil, err
	}

	msg, err := icmp.ParseMessage(1, buf[:n])
	if err != nil {
		return nil, nil, err
	}
	body, ok := msg.Body.(*icmp.Echo)
	if !ok {
		return nil, nil, errors.New("cannot convert Message.Body to *icmp.Echo")
	}

	return msg, body, nil
}

func main() {
	conn, err := icmp.ListenPacket("ip4:icmp", "0.0.0.0")
	if err != nil {
		panic(err)
	}

	if err := send(conn, "www.baidu.com"); err != nil {
		panic(err)
	}

	if msg, echo, err := receive(conn); err != nil {
		panic(err)
	} else {
		fmt.Printf("Type: %v, Code: %v, Checksum: %x\nIdentifier: %v, Sequence Number: %v\nData: %s\n",
			msg.Type, msg.Code, msg.Checksum, echo.ID, echo.Seq, echo.Data)
	}
}
```

在root权限下运行的结果如下。
```
Type: echo reply, Code: 0, Checksum: d779
Identifier: 3518, Sequence Number: 2
Data: hello icmp
```

### Checksum
Checksum的计算在`msg.Marshal`方法中调用。对每个16bit进行二进制反码求和，把最终结果的低16bit和高16bit继续求和，直到高16bit为0，返回结果。
```go
func (m *Message) Marshal(psh []byte) ([]byte, error) {
    // ...
    b := []byte{mtype, byte(m.Code), 0, 0}
    // ...
    s := checksum(b)
    // Place checksum back in header; using ^= avoids the
    // assumption the checksum bytes are zero.
    b[len(psh)+2] ^= byte(s)
    b[len(psh)+3] ^= byte(s >> 8)
    return b[len(psh):], nil
}

func checksum(b []byte) uint16 {
	csumcv := len(b) - 1 // checksum coverage
	s := uint32(0)
	for i := 0; i < csumcv; i += 2 {
		s += uint32(b[i+1])<<8 | uint32(b[i])
	}
	if csumcv&1 == 0 {
		s += uint32(b[csumcv])
	}
	s = s>>16 + s&0xffff
	s = s + s>>16
	return ^uint16(s)
}
```

下面的代码是模拟校验checksum的过程，x是待发送的ICMP数据包，先把checksum字段置0，也就是x[2]和x[3]置0。调用`checksum`方法计算结果，结果是16位整数，低位字节赋值给x[2]，高位字节赋值给x[3]。接收方收到数据后对完整的ICMP数据包调用`checksum`方法，最终计算结果是0，表示校验通过。
```go
func main() {
	x := []byte{0x01, 0x02, 0x00, 0x00, 0x03, 0x04, 0x05, 0x06, 0x08, 0x09}
	s := checksum(x)
	x[2] ^= byte(s)
	x[3] ^= byte(s >> 8)
	fmt.Println(checksum(x))
}
```

### Multiple ICMP
支持同时对多个机器发送和接收ICMP数据包，发送和接收的方法肯定要分离，连接绑定原始套接字后，发送协程把Echo Request写入连接，接收协程从连接中读取并解析Echo Reply，接收的时间戳和发送的时间戳之差就是RTT值。

问题是接收之后如何通知发送协程，`Ping`是发送Echo Request的方法，每个Echo Request都有唯一序列号Sequence，可以给每个请求分配一个通道channel，和序列号关联。`receive`是接收Echo Reply的方法，根据解析出来的Echo Reply中的序列号找到通道，通知正在等待的发送协程。
```go
func (p *Ping) Ping(ip string, timeout time.Duration) (time.Duration, error) {
	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(timeout))
	defer cancel()

	dest, err := net.ResolveIPAddr("ip", ip)
	if err != nil {
		return 0, err
	}

	var (
		seq = int(atomic.AddUint32(&sequence, 1))
		msg = icmp.Message{
			Code: 0,
			Body: &icmp.Echo{
				ID:   p.id,
				Seq:  seq,
				Data: p.payload,
			},
		}
		conn *icmp.PacketConn
		lock *sync.Mutex
	)

	if dest.IP.To4() != nil {
		msg.Type = ipv4.ICMPTypeEcho
		conn = p.conn4
		lock = &p.lock4
	} else {
		msg.Type = ipv6.ICMPTypeEchoRequest
		conn = p.conn6
		lock = &p.lock6
	}

	bs, err := msg.Marshal(nil)
	if err != nil {
		return 0, err
	}

	req := newRequest()
	p.addReq(seq, req)

	lock.Lock()
	_, err = conn.WriteTo(bs, dest)
	lock.Unlock()

	if err != nil {
		req.close()
		p.delReq(seq)
		return 0, err
	}

	select {
	case <-req.wait:
	case <-ctx.Done():
		p.delReq(seq)
		return 0, errors.New("wait echo reply timeout")
	}

	return req.rtt()
}

func (p *Ping) receive(proto int, conn *icmp.PacketConn) {
	buf := make([]byte, 1500)
	defer p.wg.Done()

	for {
		// conn.IPv4PacketConn().ReadFrom(buf)
		// conn.IPv6PacketConn().ReadFrom(buf)
		n, _, err := conn.ReadFrom(buf)
		if err != nil {
			if netErr, ok := err.(net.Error); !ok || !netErr.Temporary() {
				break
			}
		}

		msg, err := icmp.ParseMessage(proto, buf[:n])
		if err != nil {
			continue
		}

		switch msg.Type {
		case ipv4.ICMPTypeEchoReply, ipv6.ICMPTypeEchoReply:
			echo, ok := msg.Body.(*icmp.Echo)
			if !ok || echo == nil {
				continue
			}

			if echo.ID != p.id {
				continue
			}

			req := p.delReq(echo.Seq)
			if req != nil {
				req.close()
			}

		default:
		}
	}
}
```

完整代码可以在[GitHub](https://github.com/fancxxy/ping)查看。

下面的例子演示同时ping多个地址的结果。
```go
package main

import (
	"fmt"
	"sync"
	"time"

	"github.com/fancxxy/ping"
)

func rtt(p *ping.Ping, dest string, timeout time.Duration, wg *sync.WaitGroup) {
	defer wg.Done()
	rtt, err := p.Ping(dest, timeout)
	if err != nil {
		fmt.Printf("ping %s, error %s\n", dest, err)
	} else {
		fmt.Printf("ping %s, rtt %.3f ms\n", dest, float64(rtt/time.Microsecond)/1000.0)
	}
}

func main() {
	p, err := ping.New("0.0.0.0", "")
	if err != nil {
		panic(err)
	}

	var (
		dests = []string{
			"www.baidu.com",
			"www.weibo.com",
			"www.qq.com",
			"www.bilibili.com",
			"www.zhihu.com",
			"www.sina.com.cn",
			"www.google.com",
		}
		timeout = time.Second * 2
		wg      sync.WaitGroup
	)

	wg.Add(len(dests))
	for _, dest := range dests {
		go rtt(p, dest, timeout, &wg)
	}

	wg.Wait()
}
```

输出结果如下。
```
ping www.baidu.com, rtt 25.514 ms
ping www.sina.com.cn, rtt 26.012 ms
ping www.weibo.com, rtt 53.461 ms
ping www.zhihu.com, rtt 27.881 ms
ping www.bilibili.com, rtt 29.017 ms
ping www.qq.com, rtt 38.193 ms
ping www.google.com, error wait echo reply timeout
```
