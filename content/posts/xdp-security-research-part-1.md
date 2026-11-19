---
title: "XDP在安全上的一些研究 I"
date: "2024-06-26T10:26:17+08:00"
previewImage: "/images/covers/security-research.svg"
draft: false
categories:
  - "安全研究"
tags:
  - "XDP"
  - "eBPF"
  - "Linux"
  - "网络安全"
---
## 1. 网络数据包是如何进入进计算机的

众所周知，网络数据包通常需要在TCP/IP协议栈中进行处理，但网络数据包并不直接进入TCP/IP协议栈；相反，他们直接进入网络接口。因此，在数据包进入 TCP/IP 堆栈之前，它们已经到达计算机内部。到目前为止，大多数应用程序都是在 TCP/IP 堆栈之后处理的。

![image-20240118150012600](/images/legacy/image-20240118150012600.png)

## 2. 什么是XDP

XDP，全称eXpress Data Path，是Linux内核提供的一个高可用性和可编程性的网络数据包处理框架，XDP允许在网络数据包到达Linux网络栈之前，在网络驱动程序级别对数据包进行处理。 XDP 使内核有能力在数据包到达网络层时快速处理数据包，它有高性能、高灵活度、低开销等优点。

## 3. 开始一个简单的XDP项目

**挂载XDP程序请谨慎，一条错误的xdp规则是极有可能导致服务器失联的！**

以下是一个很简单的XDP示例程序

~~~C
#include <linux/bpf.h>

SEC("xdp")
//XDP 处理逻辑
int xdp_main(struct xdp_md *ctx)
{
    static int count = 1;
    count++;
    if (count%2)
    {
        return XDP_DROP;
    }
    else
    {
        return XDP_PASS;
    }
}

char __license[] __section("license") = "GPL";
~~~

工作时，当数据包进入接口时，计数将加1。当计数为偶数时，数据包被丢弃。

我这里采用了clang来编译XDP程序，不过clang有一定的版本限制，需要10.0以上，同样对于内核版本来说xdp需要4.15以上。

**编译命令**

~~~bash
clang -O2 -g -Wall -target bpf -c main.c -o xdp.o
~~~

当命令执行时，它会生成一个名为 xdp.o 的文件。

![image-20240119104600652](/images/legacy/image-20240119104600652.png)

当所有准备工作完成后，在网络接口上挂载xdp.o。

~~~bash
ip link set dev ens33 xdp obj xdp.o
~~~

如果您不需要此xdp程序，可以将其卸载。

~~~bash
ip link set dev ens33 xdp off
~~~

当这个XDP程序处于运行状态时，如果去ping该主机，每两组数据包中，就会有一组无响应，就像如下这样。![image-20240119111654161](/images/legacy/image-20240119111654161.png)

## 4. 逐步升级XDP应用功能

### 4.1 Level 1 使用XDP记录源地址IP

由于XDP是内核应用，而将源地址IP记录到本地又是一个用户态行为，那么我们就需要设法让内核态跟用户态进行交互，在这里我们使用了自带的bpf系统当中的MAP来进行数据交换。Map的本质是结构，它允许用户在内核空间和用户空间之间存储和共享数据。

#### 4.1.1 XDP

~~~C
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

#define MAX_ENTRIES 1024

// 定义一个xdp_map结构，其中要包含map的类型、kv的数据类型以及map的最大条数
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __type(value, __u32);
    __uint(max_entries, MAX_ENTRIES);
} xdp_map SEC(".maps");

struct ip_event {
    __u32 src_ip;
};

SEC("xdp")
int xdp_main(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    struct iphdr *ip;
    // 对一些畸形包以及非ipv4的包放行
    if (eth + 1 > data_end) {
        return XDP_PASS;
    }
    if (eth->h_proto != __constant_htons(ETH_P_IP)) {
        return XDP_PASS;
    }

    ip = data + sizeof(*eth);
    if (ip + 1 > data_end) {
        return XDP_PASS;
    }
    struct ip_event evt = {
        .src_ip = ip->saddr,
    };
    __u32 key = 0; 
    __u32 value = evt.src_ip;
    bpf_map_update_elem(&xdp_map, &key, &value, BPF_ANY);
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
~~~

在上述代码当中使用了**bpf_map_update_elem**接口，这个接口是bpf系统提供用来更新映射Map的。

#### 4.1.2 用户态

~~~go
package main

import (
	"fmt"
	"log"
	"net"
	"time"

	"github.com/cilium/ebpf"
	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/rlimit"
)

var (
	XDP_PATH = "xdp.o"
	IF       = "ens33"
	XDP_Func = "xdp_main"
	XDP_Map  = "xdp_map"
)

func main() {
	if err := rlimit.RemoveMemlock(); err != nil {
		log.Fatalf("关闭内存锁失败: %v", err)
	}
	// 加载xdp程序规则
	spec, err := ebpf.LoadCollectionSpec(XDP_PATH)
	if err != nil {
		log.Fatalf("XDP 程序加载失败: %v", err)
	}
	coll, err := ebpf.NewCollection(spec)
	if err != nil {
		log.Fatalf("创建新的程序集失败: %v", err)
	}
	defer coll.Close()
	// 在当前的程序集当中寻找名为 xdp_map 的bpfmap对象
	cmdMap := coll.DetachMap(XDP_Map)
	if cmdMap == nil {
		log.Fatalf("在%s当中未找到%s对象", XDP_PATH, XDP_Map)
	}
	// 将网卡名称转换为网卡index
	ifIndex, err := getInterfaceIndex(IF)
	if err != nil {
		log.Fatalf("获取%s索引失败: %v", IF, err)
	}
	// 寻找
	prog := coll.Programs[XDP_Func]
	if prog == nil {
		log.Fatalf("未找到%s方法",XDP_Func)
	}
	link, err := link.AttachXDP(link.XDPOptions{
		Program:   prog,
		Interface: ifIndex,
	})
	defer link.Close()
	// 读取map当中内容
	var key uint32 = 0
	value := make([]byte, 512)

	for {
		err = cmdMap.Lookup(&key, &value)
		if err != nil {
			log.Printf("查询map失败: %v", err)
			time.Sleep(1 * time.Second)
			continue
		}
		fmt.Println(value)
		time.Sleep(1 * time.Second)
	}
}
func getInterfaceIndex(name string) (int, error) {
	ifIndex, err := net.InterfaceByName(name)
	if err != nil {
		return 0, err
	}
	return ifIndex.Index, nil
}

~~~

在一通折腾下，达成效果如下，这同时也意味着，成功的将网卡收到的数据经过xdp程序传递到了用户态。

![image-20240623174709784](/images/legacy/image-20240623174709784.png)

当然对于想要进行使用某一个特定源ip的访问来触发某个事件的场景，也可以将打印的逻辑改掉，改成自己想要的样子。

### 4.2 Level 2 使用XDP做一个简易的防火墙

第二阶段开始尝试着使用XDP的行为指令来处理数据包，XDP 定义了数据包的五种处理行为

~~~C
enum xdp_action {
	XDP_ABORTED = 0, // 将数据包丢弃，并抛出异常
	XDP_DROP, // 丢弃数据包
	XDP_PASS, // 放行至内核协议栈
	XDP_TX, // 从收到的这张往卡上再将包发出
	XDP_REDIRECT, // 从其他网卡将包发出，
};
~~~

#### 4.2.1 XDP程序

在这个xdp程序当中，使用了bpf_map_lookup_elem来获取被禁止的ip，如果被禁止的ip存在在map当中，那么该ip就会被XDP_DROP行为丢掉

~~~c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

#define MAX_ENTRIES 1024

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __type(value, __u32);
    __uint(max_entries, MAX_ENTRIES);
} xdp_map SEC(".maps");

SEC("xdp")
int xdp_main(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    struct iphdr *ip;

    if (eth + 1 > data_end) {
        return XDP_PASS;
    }

    if (eth->h_proto != __constant_htons(ETH_P_IP)) {
        return XDP_PASS;
    }

    ip = data + sizeof(*eth);
    if (ip + 1 > data_end) {
        return XDP_PASS;
    }

    __u32 src_ip = ip->saddr;
    __u32 *value;

    // 检查源 IP 是否在 BPF map 中
    value = bpf_map_lookup_elem(&xdp_map, &src_ip);
    if (value) {
        // 如果存在，则丢弃包
        return XDP_DROP;
    }

    // 否则放行包
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
~~~

#### 4.2.2 用户态程序

在研究过程中，使用golang来写用户态程序会在将ip的uint32作为key出现问题，可能是遇到了C跟go之间的奇妙羁绊了，所以无奈使用c来写这个用户态程序，值得注意的是在C当中挂载函数**bpf_set_link_xdp_fd**在较高内核版本当中变为了**bpf_xdp_attach**

~~~C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <errno.h>
#include <net/if.h>
#include <arpa/inet.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <linux/bpf.h>

static int xdp_map_fd;
static int ifindex;

// cleanup用来清除xdp程序，若不写善后程序则在用户态程序关闭之后xdp程序还挂载在网卡上
static void cleanup(int sig) {
    if (ifindex) {
        bpf_set_link_xdp_fd(ifindex, -1, 0);
    }
    exit(0);
}

int main(int argc, char **argv) {
    struct bpf_object *obj;
    int prog_fd;
    __u32 key, value;


    signal(SIGINT, cleanup);
    signal(SIGTERM, cleanup);

    obj = bpf_object__open_file(argv[1], NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "XDP 程序打开失败\n");
        return 1;
    }

    if (bpf_object__load(obj)) {
        fprintf(stderr, "XDP 程序加载失败\n");
        return 1;
    }

    prog_fd = bpf_program__fd(bpf_object__find_program_by_title(obj, "xdp"));
    if (prog_fd < 0) {
        fprintf(stderr, "未查询到xdp程序的fd\n");
        return 1;
    }

    xdp_map_fd = bpf_object__find_map_fd_by_name(obj, "xdp_map");
    if (xdp_ctrl_map_fd < 0) {
        fprintf(stderr, "未查询到map的fd\n");
        return 1;
    }

    ifindex = if_nametoindex(argv[2]);
    if (ifindex == 0) {
        return 1;
    }

    if (bpf_set_link_xdp_fd(ifindex, prog_fd, 0) < 0) {
        return 1;
    }

    char command[256];
    char ip_str[INET_ADDRSTRLEN];
    char action[10];

    while (1) {
        printf("Enter command (e.g., '192.168.11.102 block' or '10.102.11.192 accept'): ");
        fgets(command, sizeof(command), stdin);

        if (sscanf(command, "%s %s", ip_str, action) != 2) {
            fprintf(stderr, "非法输入\n");
            continue;
        }

        key = inet_addr(ip_str);

        if (strcmp(action, "block") == 0) {
            value = 1;
            if (bpf_map_update_elem(xdp_ctrl_map_fd, &key, &value, BPF_ANY) != 0) {
                perror("数据压入失败");
            } else {
                printf("Blocked IP address: %s\n", ip_str);
            }
        } else if (strcmp(action, "accept") == 0) {
            if (bpf_map_delete_elem(xdp_map_fd, &key) != 0) {
                perror("数据删除失败");
            } else {
                printf("Accepted IP address: %s\n", ip_str);
            }
        } else {
            fprintf(stderr, "未知操作: %s\n", action);
        }
    }

    return 0;
}

// 编译命令gcc -o user user.c -l:libbpf.a -lelf -lz
~~~

下面则是正常情况下golang对map进行更新的操作

~~~go
// 使用golang对map进行更新
xdpMap := coll.Maps[XDP_Map]
xdpMap.Update(unsafe.Pointer(&key), unsafe.Pointer(&value), ebpf.UpdateAny)
~~~

在这个测试当中，我对11.1这个ip先进行了封禁，又开了放行，这样用户态和xdp程序联动的防火墙就做好了

![image-20240620153349576](/images/legacy/image-20240620153349576.png)

![image-20240620153333031](/images/legacy/image-20240620153333031.png)

### 4.3 Level 3 使用 XDP 拦截特定的请求包，并发送给用户态

第三阶段就是设法将XDP的数据包传递给用户态程序，并让用户态根据收到的数据进行一些操作。这里参考**leveryd**师傅的项目，使用连接起来最简单的udp进行信息的传递，使用map将获取到的信息储存在value当中。

#### 4.3.1 XDP

~~~C
#include <arpa/inet.h>
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/udp.h>

#define SIZE1 200
#define SIZE2 180

typedef char PAYLOAD[SIZE1];

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __type(value, PAYLOAD);
    __uint(max_entries, 1);
} xdp_map SEC(".maps");

SEC("xdp")
int xdp_main(struct xdp_md *ctx)
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    char match_pattern[] = "xdp";
    unsigned int payload_size, i;
    struct ethhdr *eth = data;
    unsigned char *payload;
    struct udphdr *udp;
    struct iphdr *ip;

    __u32 key = 0;
    PAYLOAD value;

    if ((void *)eth + sizeof(*eth) > data_end) {
        return XDP_PASS;
    }

    ip = data + sizeof(*eth);
    if ((void *)ip + sizeof(*ip) > data_end) {
        return XDP_PASS;
    }

    if (ip->protocol != IPPROTO_UDP){
        return XDP_PASS;
    }

    udp = (void *)ip + sizeof(*ip);
    if ((void *)udp + sizeof(*udp) > data_end){
        return XDP_PASS;
    }

    payload_size = ntohs(udp->len) - sizeof(*udp);
    if (payload_size != SIZE1) {
        return XDP_PASS;
    }

    payload = (unsigned char *)udp + sizeof(*udp);
    if ((void *)payload + payload_size > data_end) {
        return XDP_PASS;
    }

    for (i = 0; i < payload_size && payload_size <= SIZE1; i++){
        if (i == sizeof(match_pattern) - 1) {
            break;
        }
        if (payload[i] != match_pattern[i]){
            return XDP_PASS;
        }
    }
        bpf_map_update_elem(&xdp_map, &key, (char *)payload, BPF_ANY);
    return XDP_DROP;
}

char _license[] SEC("license") = "GPL";
~~~

#### 4.3.2 用户态

~~~go
package main

import (
	"fmt"
	"log"
	"net"
	"time"

	"github.com/cilium/ebpf"
	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/rlimit"
)

const PayloadSize = 200

var (	XDP_PATH = "xdp.o"
	IF       = "ens33"
	XDP_Func = "xdp_main"
	XDP_Map  = "xdp_map") 

func main() {
	if err := rlimit.RemoveMemlock(); err != nil {
		log.Fatalf("移除内存锁失败: %v", err)
	}
	spec, err := ebpf.LoadCollectionSpec(XDP_PATH)
	if err != nil {
		log.Fatalf("加载 XDP 程序失败: %v", err)
	}
	
	coll, err := ebpf.NewCollection(spec)
	if err != nil {
		log.Fatalf("创建新的集合失败: %v", err)
	}
	defer coll.Close()
	xdpMap := coll.DetachMap(XDP_Map)

	if xdpMap == nil {
		log.Fatalf("在 XDP 程序中找不到 map")
	}
	fmt.Println(xdpMap.FD())
	
	ifIndex, err := getInterfaceIndex(IF)
	if err != nil {
		log.Fatalf("获取接口索引失败: %v", err)
	}
	prog := coll.Programs[XDP_Func]
	if prog == nil {
		log.Fatalf("找不到 % 程序")
	}
	link.AttachXDP(link.XDPOptions{
		Program:   prog,
		Interface: ifIndex,
	})
	
	var key uint32 = 0
	value := make([]byte, 180)
	for {
		value, err = xdpMap.LookupBytes(&key)
		fmt.Println(xdpMap.String())
		if err != nil {
			log.Printf("查找 map 失败: %v", err)
			time.Sleep(1 * time.Second)
			continue
		}
		fmt.Println(value)
		if value[0] != '\x00' {
			fmt.Printf("接收到的值: %s\n", string(value))
			// 重置 map 中的值
			err = xdpMap.Update(key, make([]byte, PayloadSize), ebpf.UpdateAny)
			if err != nil {
				log.Printf("更新 map 失败: %v", err)
			}
		}
		time.Sleep(1 * time.Second)
	}
}
func getInterfaceIndex(name string) (int, error) {
	ifIndex, err := net.InterfaceByName(name)
	if err != nil {
		return 0, err
	}
	return ifIndex.Index, nil
}

~~~

由于xdp是在二层处理数据，而端口是四层才有的概念，所以对发往任意端口的包都会被接收（前提是不超过65535），因为记录包内容以后采取的是XDP_DROP行为，所以在流量上看并不能发现这个包未到达，

![image-20240624111925272](/images/legacy/image-20240624111925272.png)

![image-20240624021100965](/images/legacy/image-20240624021100965.png)

### 4.4 Level 4 使用XDP将处理过的请求包发回

#### 4.4.1 XDP_TX

在这个demo当中，我们尝试了对udp包的源地址目的地址进行了对调，并通过xdp挂载的网卡发出，其中用到了XDP_TX行为，XDP_TX默认会将包原封不动的从自己的网卡丢出。

~~~c
 if (ip->protocol != IPPROTO_UDP)
        return XDP_PASS;

    struct udphdr *udp = (void *)(ip + 1);
    if ((void *)(udp + 1) > data_end)
        return XDP_PASS;
    // 对换mac
    unsigned char tmp_mac[ETH_ALEN];
    __builtin_memcpy(tmp_mac, eth->h_source, ETH_ALEN);
    __builtin_memcpy(eth->h_source, eth->h_dest, ETH_ALEN);
    __builtin_memcpy(eth->h_dest, tmp_mac, ETH_ALEN);
    
    // 对换地址
    __u32 tmp_ip = ip->saddr;
    ip->saddr = ip->daddr;
    ip->daddr = tmp_ip;

    // 对换端口
    __u16 tmp_port = udp->source;
    udp->source = udp->dest;
    udp->dest = tmp_port;
    

    return XDP_TX;
~~~

下面是使用起来的效果

![image-20240623184432678](/images/legacy/image-20240623184432678.png)

#### 4.4.2 修改发出的包

那么利用level2当中的向手段与XDP_TX行为结合，就可以修改包内数据并发出了，以下是实现所用到的三个代码块。

~~~c
// 定义一个map
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __uint(value_size, 128);
    __uint(max_entries, 1);
} xdp_map SEC(".maps");

// 从map当中获取收到的包
__u32 key = 0;
char *payload = bpf_map_lookup_elem(&xdp_map, &key);  

//将数据包进行修改
char *udp_payload = (char *)(udp + 1);
    if ((void *)udp_payload + 128 > data_end)  
        return XDP_PASS;
 __builtin_memcpy(udp_payload, payload, 128);
~~~

在之前golang加载器的基础上加入向map当中压数据的操作，新的加载器就做好了


~~~go
// golang loader
	key := uint32(0)
	payload := [128]byte{69, 120, 112, 101, 108, 108, 105, 97, 114, 109, 117, 115}
	err = xdpMap.Update(unsafe.Pointer(&key), unsafe.Pointer(&payload), ebpf.UpdateAny)
	if err != nil {
		log.Fatalf("写入map失败: %v", err)
	}

~~~

用起来的效果是这个样子的

![image-20240623193707516](/images/legacy/image-20240623193707516.png)

利用xdp将发包收包的功能进行结合，就可以达到不通过协议栈来进行数据传递了，需要注意的是，如过想要使发回的包正常进入协议栈，还需要对增加ip包与udp包的校验和重新计算，只有校验和正确的包才能进入到传输层端口上。

## 5. 恶意XDP如何处置

当一台主机发生一些邪门的行为的时候，尤其是网络行为，有几率是被下了恶意的XDP程序，恶意的xdp可以使用bpftool工具进行排查

~~~bash
bpftool prog
~~~

![image-20240623204808594](/images/legacy/image-20240623204808594.png)

prog参数会将所有的prog以及挂载时间都列出来，根据prog可以寻找可疑的xdp运行程序，再使用ip命令可以找出是哪张网卡有xdp的挂载

~~~bash
ip link show
~~~

![image-20240623205223795](/images/legacy/image-20240623205223795.png)

如果发现了恶意程序再使用ip命令进行卸载

~~~bash
ip link set dev <interface> xdp off
~~~















参考文献

https://developers.redhat.com/blog/2021/04/01/get-started-with-xdp

https://www.cnblogs.com/bakari/p/10966303.html

https://rexrock.github.io/post/xdp1/

https://www.leveryd.top/2022-08-14-%E8%81%8A%E4%B8%80%E8%81%8A%E5%9F%BA%E4%BA%8E%22ebpf%20xdp%22%E7%9A%84rootkit/

https://developers.redhat.com/blog/2021/04/01/get-started-with-xdp

https://github.com/xdp-project/xdp-tutorial

