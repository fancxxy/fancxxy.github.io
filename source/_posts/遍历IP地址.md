---
title: 遍历IP地址
date: 2019-08-29 11:35:00
tags: golang
---

有个需求是定时探测公司所有网段可用的IP地址，记录探测结果到数据库。网段的信息也存储在数据库中，我看了下有两种存储格式，一种是以CIDR格式存储的，比如：`168.61.45.4/30`，还有一种是给定了IP区间范围，比如：`[168.60.73.250 - 168.60.74.5]`。
<!--more-->

主要的任务就是解析这两种存储格式，得到它们表示范围内的所有IP地址，再对获取到的地址进行ping探测。

### 区间格式
IP地址其实是一个32位的二进制数，为了方便记忆一般写成点分十进制，所以只需要把起始IP和终止IP都转换成uint32，每次加1循环遍历，之后再把uint32的数字转成点分十进制格式。Go的binary包提供了字节序列和数字的互转。

```go
package main

import (
	"encoding/binary"
	"fmt"
	"net"
)

func main() {
	section := []string{"168.60.73.250", "168.60.74.5"}

	beginIP := net.ParseIP(section[0])
	endIP := net.ParseIP(section[1])

	beginVal := binary.BigEndian.Uint32(beginIP.To4())
	endVal := binary.BigEndian.Uint32(endIP.To4())

	var ips []string
	for i := beginVal; i <= endVal; i++ {
		ip := make(net.IP, 4)
		binary.BigEndian.PutUint32(ip, i)
		ips = append(ips, ip.String())
	}
	fmt.Printf("%#v\n", ips)
}
```

执行`$ go run main.go`的输出结果
```
[]string{"168.60.73.250", "168.60.73.251", "168.60.73.252", "168.60.73.253", "168.60.73.254", "168.60.73.255", "168.60.74.0", "168.60.74.1", "168.60.74.2", "168.60.74.3", "168.60.74.4", "168.60.74.5"}
```

### CIDR格式
CIDR中文名无分类域间路由，是用来给用户分配IP地址以及对IP地址进行归类的方法，基于可变长子网掩码进行任意长度的前缀分配，相比于之前的五类IP，分配更加灵活。

`168.61.45.4/30`，表示IP的前30位是网络前缀，后2位是主机号，所以它表示的网段里有4个IP地址。使用Go的`net.ParseCIDR`函数来解析CIDR字符串。

```go
package main

import (
	"fmt"
	"log"
	"net"
)

func main() {
	cidr := "168.61.45.4/30"
	ipAddr, ipNet, err := net.ParseCIDR(cidr)
	if err != nil {
		log.Fatalf("net.ParseCIDR error: %v", err)
	}

	var ips []string
	for ip := ipAddr.Mask(ipNet.Mask); ipNet.Contains(ip); inc(ip) {
		ips = append(ips, ip.String())
	}

	fmt.Printf("%#v\n", ips)

}

func inc(ip net.IP) {
	for j := len(ip) - 1; j >= 0; j-- {
		ip[j]++
		if ip[j] > 0 {
			break
		}
	}
}
```

执行`$go run main.go`的输出结果是
```
[]string{"168.61.45.4", "168.61.45.5", "168.61.45.6", "168.61.45.7"}
```
