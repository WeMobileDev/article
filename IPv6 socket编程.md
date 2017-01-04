#IPv6 socket编程 

##背景
研究IPv6 socket编程原因：
> [Supporting IPv6 in iOS 9](https://developer.apple.com/news/?id=08282015a)</br>
> WWDC2015苹果宣布在ios9支持纯IPv6的网络服务，并且要求2016年提交到app store的应用必须兼容纯IPv6的网络，要求适配的系统版本是ios9以上（包括ios9）。

写这篇文章虽然是来源于iOS的需求，但是下面的内容除了特别说明外，大部分都适用于其他平台。</br>
IPv6的复杂度之一，在于和IPv4的兼容和相互访问。本文会提及其他的互相访问技术，但是重点是`NAT64`，也是一般手机用户最有可能遇到的纯IPv6环境。</br>
本文重点在**不同IP stack组合的处理方式**和**判断客户端支持的IP stack**。

##问题复杂性
为了降低问题的复杂性，我们先把v4 socket排除掉，统一使用v6 socket。
v6 socket的区别是使用`AF_INET6`来创建。</br>
IPv6转换机制有很多种，苹果期望iOS app能兼容NAT64/DNS64的方式，因此其他方式我们先不考虑。

1.	socket api支持 [RFC 4038 - Application Aspects of IPv6 Transition]
	-	v4 socket接口只能支持IPv4 stack
	-	v6 socket能支持IPv4 stack和IPv6 stack
2.	服务器IP
	-	返回v4 IP
	-	返回v6 IP
3.	用户本地IP stack
	- IPv4-only
	- IPv6-only
	- IPv4-IPv6 Dual stack
4.	各种IPv6转换机制
	-	NAT64/DNS64 `64:ff9b::/96`用于v6的本地网络通过NAT访问v4的资源。[RFC 6146](https://tools.ietf.org/html/rfc6146)       	、[RFC 6147](https://tools.ietf.org/html/rfc6147)
	-	6to4 `2002::/16`用于两个拥有v4公网地址的IPv6 only子网的互相访问。[RFC 6343](https://tools.ietf.org/html/rfc6343)
	-	Teredo tunneling `2001::/32`通过隧道的方式让两个IPv6 only子网互相访问，没有NAT问题。[RFC 4380](https://tools.ietf.org/html/rfc4380)
	-	464XLAT 用于程序只有v4地址(使用v4 socket)，但是本地网络是ipv6网络，程序需要访问v4资源，类似NAT64，不过区别在于服务器是运营商提供，手机上需要安装`CLAT服务` 。[RFC 6877](https://tools.ietf.org/html/rfc6877)
	-	还有很多兼容方案，复杂程度都很高，这里不介绍了



##不同IP stack组合的处理方式
###v4 ip + IPv4-only or IPv4-IPv6 Dual stack
在这样的情况下我们虽然用的是v6的socket，但是必须要让socket走的是v4的协议。
这里，让我们先了解下IPv6的保留地址（类似IPv4，192.168.*.*, 127.*.*.*这种）这里假设读者已经对IPv6地址组成和书写方式有一定了解的了解。

>  ::ffff:0:0/96 — This prefix is designated as an IPv4-mapped IPv6 address. With a few exceptions, this address type allows the transparent use of the Transport Layer protocols over IPv4 through the IPv6 networking application programming interface. Server applications only need to open a single listening socket to handle connections from clients using IPv6 or IPv4 protocols. IPv6 clients will be handled natively by default, and IPv4 clients appear as IPv6 clients at their IPv4-mapped IPv6 address. Transmission is handled similarly; established sockets may be used to transmit IPv4 or IPv6 datagram, based on the binding to an IPv6 address, or an IPv4-mapped address. (See also Transition mechanisms.) [^1]

从上文可以看到如果服务器地址为`128.0.0.128`，我们转换成IPv4-mapped IPv6 address`::ffff:128.0.0.128`或者纯16进制`::ffff：ff00：00ff`, 然后赋值给`sockaddr_in6.sin6_addr = "::ffff:128.0.0.128";`（注意这里是伪代码，真正代码还要用inet_pton进行转换）。这个socket虽然用了IPv6的`sockaddr_in6`，但实际上走的是IPv4 stack。</br></br>

IPv4-mapped IPv6 address是让用户能够使用一致的socket api，来访问IPv4和IPv6网络。

上文提及[RFC 4038 - Application Aspects of IPv6 Transition](https://www.ietf.org/rfc/rfc4038)对这种情况进行说明。

	//IPv4-mapped IPv6 address sample
	//address init 
	const char* ipv4mapped_str ="::FFFF:14.17.32.211";
    in6_addr ipv4mapped_addr = {0};
    int v4mapped_r = inet_pton(AF_INET6, ipv4mapped_str, &ipv4mapped_addr);
   
    sockaddr_in6 v4mapped_addr = {0};
    v4mapped_addr.sin6_family = AF_INET6;
    v4mapped_addr.sin6_port = htons(80);
    v4mapped_addr.sin6_addr = ipv4mapped_addr;

   	//socket connect
 	int v4mapped_sock = socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP); 
    std::string v4mapped_error;
    if (0 != connect(v4mapped_sock, (sockaddr*)&v4mapped_addr, 28))
    {
        v4mapped_error = strerror(errno);
    }
    
	//get local ip
    sockaddr_in6 v4mapped_local_addr = {0};
    socklen_t v4mapped_local_addr_len = 28;
    char v4mapped_str_local_addr[64] = {0};
    getsockname(v4mapped_sock, (sockaddr*)&v4mapped_local_addr, &v4mapped_local_addr_len);
    inet_ntop(v4mapped_local_addr.sin6_family, &v4mapped_local_addr.sin6_addr, v4mapped_str_local_addr, 64);

    close(v4mapped_sock);


###v4 ip + IPv6-only
这里是重点，也是苹果要求支持的主要场景。这里会涉及到NAT64/DNS64，关于这个环境的搭建请参考[Supporting IPv6 DNS64/NAT64 Networks]（废弃了的SIIT技术我们就不讨论了）

这里我们先看看wikipedia对NAT64/DNS64的描述。
>NAT64 is a mechanism to allow IPv6 hosts to communicate with IPv4 servers. The NAT64 server is the endpoint for at least one IPv4 address and an IPv6 network segment of 32-bits, e.g., 64:ff9b::/96 (RFC 6052, RFC 6146). The IPv6 client embeds the IPv4 address with which it wishes to communicate using these bits, and sends its packets to the resulting address. The NAT64 server then creates a NAT-mapping between the IPv6 and the IPv4 address, allowing them to communicate.[^2]

>DNS64 describes a DNS server that when asked for a domain's AAAA records, but only finds A records, synthesizes the AAAA records from the A records. The first part of the synthesized IPv6 address points to an IPv6/IPv4 translator and the second part embeds the IPv4 address from the A record. The translator in question is usually a NAT64 server. The standard-track specification of DNS64 is in RFC 6147.
> 
> There are two noticeable issues with this transition mechanism:
> 
> -  It only works for cases where DNS is used to find the remote host address, if IPv4 literals are used the DNS64 server will never be involved.
> -  Because the DNS64 server needs to return records not specified by the domain owner, DNSSEC validation against the root will fail in cases where the DNS server doing the translation is not the domain owner's server.[^3]

这里大概描述一下NAT64的工作流程，首先局域网内有一个NAT64的路由设备并且有DNS64的服务。

1.	客户端进行getaddrinfo的域名解析.
2.	DNS返回结果，如果返回的IP里面只有v4地址，并且当前网络是IPv6-only网络，DNS64服务器会把v4地址加上`64:ff9b::/96`的前缀，例如`64:ff9b::14.17.32.211`。如果当前网络是IPv4-only或IPv4-IPv6，DNS64不会做任何事情。
3.	客户端拿到IPv6的地址进行connect
4.	路由器发现地址的前缀为`64:ff9b::/96`，知道这个是NAT64的映射，是需要访问`14.17.32.211`。这个时候进行需要NAT64映射，因为到外网需要转换成IPv4 stack。
5.	当数据返回的时候，按照NAT映射，IPv4回包重新加上前缀`64:ff9b::/96`，然后返回给客户端。

apple的文档里面也有很详细的描述：
![img](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/art/NAT64-DNS64-ResolutionOfIPv4_2x.png)
![img](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/art/NAT64-DNS64-Workflow_2x.png)

	//NAT64 address sample
	//address init
    const char* ipv6_str ="64:ff9b::14.17.32.211";
    in6_addr ipv6_addr = {0};
    int v6_r = inet_pton(AF_INET6, ipv6_str, &ipv6_addr);
    sockaddr_in6 v6_addr = {0};
    v6_addr.sin6_family = AF_INET6;
    v6_addr.sin6_port = htons(80);
    v6_addr.sin6_addr = ipv6_addr;

	//socket connect
	int v6_sock = socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP)；
    std::string v6_error;
    if (0 != connect(v6_sock, (sockaddr*)&v6_addr, 28))
    {
        v6_error = strerror(errno);
    }
    
	//get local ip
    sockaddr_in6 v6_local_addr = {0};
    socklen_t v6_local_addr_len = 28;
    char v6_str_local_addr[64] = {0};
    getpeername(v6_sock, (sockaddr*)&v6_local_addr, &v6_local_addr_len);
    inet_ntop(v6_local_addr.sin6_family, &v6_local_addr.sin6_addr, v6_str_local_addr, 64);
    
    close(v6_sock);
这里讨论下比较坑的地方，按照NAT64的规则，客户端如果没有做DNS域名解析的话（微信依赖的是自己实现的NEWDNS），客户端就需要完成DNS64的工作。这里的关键点是，发现网络是IPv6-only的NAT64网络的情况下，我们可以自己补充上前缀`64:ff9b::/96`，然后进行正常的访问。然而这里客户端能获取的信息量一般都是很有限的，怎么样处理这个问题，后面有专门的章节来处理这个问题(**判断客户端支持的IP stack**)。
###v6 ip + IPv4-only
这里一般connect的时候会返回错误码`network is unreachable`,因为根本没有v6的协议栈，就像没有硬件设备一样，但是不排除会有系统会返回`no route to host `。
当然，如果服务器的地址是 `Teredo tunneling 2001::/32`，可以客户端直接做隧道。如果是`6to4 2002::/16`，并且客户端有RAW socket权限加上非NAT网络，这种情况下可以客户端自己做6to4的路由。(这里的结论不一定百分百正确，还需要继续研读RFC)。

###v6 ip + IPv6-only or IPv4-IPv6
这里只要没有配置上，是可以直接通讯的。
当然这里会涉及到一个问题，如果DNS返回上文说的`6to4`或`Teredo tunneling`或`pure native IPv6 addresses`，这样的情况下我们怎么样做IP的选择呢，这个可以参照[RFC 3484 - Default Address Selection for Internet Protocol version 6 (IPv6)](https://tools.ietf.org/html/rfc3484)。

##判断客户端可用的IP stack(IPv4-only、 IPv6-only、IPv4-IPv6 Dual stack)
原理大家都明白了，但是客户端做不同的处理的前提是需要知道**客户端可用的IP协议栈**。</br>
我们先定义**客户端可用的IP协议栈**的意思是，获取客户端当前能使用的IP协议栈。例如iOS在NAT64 WIFI连接上的情况下，Mobile的网卡虽然存在IPv4的协议栈，但是系统是不允许使用的。IOS只能使用WIFI的协议栈，在NAT64 WIFI的情况下就是IPv6-only网络了。</br>
这里还有一个问题需要讨论，如果遇到IPv6-only网络，需要把它当作NAT64来处理，在v4 IP前添加前缀`64:ff9b::/96`。</br>
但是这里NAT64和IPv6-only不是等价的。IPv6-only网络可能支持NAT64，能访问v4的互联网资源，但是IPv6-only能访问v6的互联网资源，不支持NAT64。这里假设IPv6-only的网络都是支持NAT64的，对v4 IP进行`64:ff9b::/96`的处理。因为不支持NAT64的话，微信服务器v4地址根本就不可访问（当然如果手机系统有`464XLAT`服务，并且运营商支持，也是可以访问v4资源的，但是不在讨论范围了）。

###获取本地IP和网关方案（iOS）
-	IOS通过sysctl获取当前网关或路由

	    如果只能获取IPv6网关，那当前是IPv6-only
	    如果只能获取IPv4网关，那当前是IPv4-only
	    如果同时能获取IPv6/IPv4路由，那情况就比复杂，分析如下
	    IOS在WIFI连接上的情况下，并不会关闭Mobile的网卡。
	    在WIFI是IPv6-only网络，Mobile是IPv4-only网络，下v4 socket或者v4-mapped都无法出去。
		证明apple应该对TCP connect函数进行过改造，在WIFI和Mobile共存的情况下，只能走WIFI网络，和Android不一样，iOS不是通过去掉Mobile网卡的方式来做。
		这样导致的一个有趣的特性:网络切换时候如果Mobile 下建立的socket不关闭可以继续使用Mobile网络。
		如果程序使用bind接口绑定到Mobile的网卡下，这个时候是可以使用Mobile网络进行访问的。（这里算不算偷流量呢，当然这里是特性，具体怎么样应用是程序的问题了）。
	    因此我们可以考虑WIFI连接了的情况下，我们只要知道网关是对应那张网卡，就可以知道当前是不是当前支持的IP协议栈？
	    然而事情没有那么简单，我们先按照刚刚说的思路走下去


-	通过getifaddr接口，可以拿到当前全部网络的IP地址（排除掉非活跃和loopback的网卡）

		如果IPv4、IPv6网关都属于WIFI网卡，那当前是IPv4-IPv6 Dual stack
		如果IPv4、IPv6网关都属于Mobile网卡，那当前是IPv4-IPv6 Dual stack
		到这里都没有问题，但是下面的情况呢：
		如果IPv4网关属于Mobile网卡，IPv6网关属于WIFI?
		如果IPv4网关属于WIFI网卡，IPv6网关属于Mobile?
		这里的情况还要分开，如果是正常情况下IOS在WIFI连接后是不允许使用Mobile网卡的，但是iOS又有一个特性是3G热点。
		在这样的情况下IOS手机本身是走Mobile网络的，WIFI只是做桥接。
		
		
这个方案非常复杂，而且跟iOS平台的系统实现强耦合，其他平台必须重新实现，后续如果iOS进行网络逻辑的更新，这里还必须修改。因此这个的方案不太建议大家用。

###DNS方案
这里的方案是直接做DNS解析，然后判断返回的IP有没有带上`64:ff9b`前缀来确定当前的IP协议栈。这也是唯一能够判断IPv6-only网络是否支持NAT64的方案。

	//gateway
	in6_addr addr6_gateway = {0};
    if (0 != getdefaultgateway6(&addr6_gateway))
        return EIPv4;
    
    if (IN6_IS_ADDR_UNSPECIFIED(&addr6_gateway))
        return EIPv4;
    
    in_addr addr_gateway = {0};
    if (0 != getdefaultgateway(&addr_gateway))
        return EIPv6;
    
    if (INADDR_NONE == addr_gateway.s_addr || INADDR_ANY == addr_gateway.s_addr )
        return EIPv6;
    
	//getaddrinfo
	struct addrinfo hints, *res, *res0;
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = PF_INET6;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_ADDRCONFIG|AI_V4MAPPED;
    int error = getaddrinfo("dns.weixin.qq.com", "http", &hints, &res0);
	
	if (0 != error) {
		return EIPv4; 
	}
	
 	for (res = res0; res; res = res->ai_next) {
		if (AF_INET6 == res->ai_addr.sa_family) {
			if (is_nat64_address(((sockaddr_in6&)res->ai_addr).sin6_addr)) {
				return EIPv6;
			}
		}
    }

	return EIPv4;


我们分析下上面的sample，前面gateway的代码是为了加速判断过程，我们知道DNS是一个网络过程，耗时很有可能是非常久的。
`dns.weixin.qq.com`必须保证解析的域名只有v4 ip地址。`hints.ai_family = PF_INET6`利用了DNS64的特性，如果在纯IPv6环境下会返回NAT64映射地址的方式。`AI_V4MAPPED`为了在非DNS64网络下，返回v4-mapped ipv6 address，不会返回EAI_NONAME失败，导致判断不准确。`AI_ADDRCONFIG`返回的地址是本地能够使用的（具体可以看文档下面的介绍）。如果有NAT64前缀的v6地址返回，证明当前网络是IPv6-only NAT64网络。</br>
不过这个方案有很多缺点，就是耗时不确定，可能因为网络失败导致错误的结果，需要网络流量，会对运营商的DNS服务器造成压力，网络切换需要立刻进行重试重连。</br>
结论，这个方案不太合适。

###socket connect的方式(支持iOS9和Android)
这里的方案是直接使用v4 IP地址和v6 IP地址进行连接，通过结果来确认当前客户端可用IP stack。

	_test_connect(int pf, struct sockaddr *addr, socklen_t addrlen) {
	    int s = socket(pf, SOCK_STREAM, IPPROTO_TCP);
	    if (s < 0)
	        return 0;
	    int ret;
	    do {
	        ret = connect(s, addr, addrlen);
	    } while (ret < 0 && errno == EINTR);
	    int success = errno;
	    do {
	        ret = close(s);
	    } while (ret < 0 && errno == EINTR);
	    return success;
	}
	
	static int
	_have_ipv6() {
	    static const struct sockaddr_in6 sin6_test = {
	        .sin6_len = sizeof(sockaddr_in6),
	        .sin6_family = AF_INET6,
	        .sin6_port = htons(0xFFFF),
	        .sin6_addr.s6_addr = {
	            0, 0x64, 0xff, 0x9b, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}
	    };
	    sockaddr_union addr = { .in6 = sin6_test };
	    return _test_connect(PF_INET6, &addr.generic, sizeof(addr.in6));
	}
	
	static int
	_have_ipv4() {
	    static const struct sockaddr_in sin_test = {
	        .sin_len = sizeof(sockaddr_in),
	        .sin_family = AF_INET,
	        .sin_port = htons(0xFFFF),
	        .sin_addr.s_addr = htonl(0x08080808L),  // 8.8.8.8
	    };
	    sockaddr_union addr = { .in = sin_test };
	    return _test_connect(PF_INET, &addr.generic, sizeof(addr.in));
	}

	enum TLocalIPStack {
    	ELocalIPStack_None = 0,
    	ELocalIPStack_IPv4 = 1,
    	ELocalIPStack_IPv6 = 2,
    	ELocalIPStack_Dual = 3,
	};

	void test() {
		TLocal IPlocal_stack = 0;
		int errno_ipv4 = _have_ipv4();
	    int errno_ipv6 = _have_ipv6();
	    int local_stack = 0;
	    if ( errno_ipv4 != EHOSTUNREACH && errno_ipv4 != ENETUNREACH) {
	        local_stack |= ELocalIPStack_IPv4;
	    }
	    if (errno_ipv6 != EHOSTUNREACH && errno_ipv6 != ENETUNREACH) {
	        local_stack |= ELocalIPStack_IPv6;
	    }
	}

这个方案是利用外网IP进行连接，如果返回`EHOSTUNREACH`的时候说明本地没有对应的路由到达目标地址，如果`ENETUNREACH`的时候说明本地没有相应的协议栈，这两种情况都是说明相应的协议栈不可用。</br>
分析下这个方案的缺点，和getaddrinfo一样，耗时不确定，因为有调用connect动作，进行tcp连接。如果connect遇到`EHOSTUNREACH ENETUNREACH`错误是不会耗费流量和立刻返回的，因为这些都是本地网络判断。但是，如果相应网络可用，这个是要花费网络流量的，耗时也不能确定。如果我们连接一个存在的IP，这样在网络好的时候很快返回（这样会对服务器造成连接的压力），网络差的时候很久才返回。如果连接一个不存在的IP，需要很久时间才会返回(75s的连接超时)。

这样看来，这三个方案都不完美，根本不能在真实场景中使用, 有没有更加可用的方案呢？iOS 9.0 上层Objc Framework可以无缝支持，但是用bsd socket需要代码完成对应的工作。但是iOS Framework的最新源码也没有开源出来，无法知道其实现原理。</br>
继续研究发现，getaddrinfo的`AI_ADDRCONFIG` flags有点像我们需要实现的功能，要去掉IP，就必须要知道当前的IP stack。它是怎么样实现的？

	//Android的AI_ADDRCONFIG 功能的sample
	_test_connect(int pf, struct sockaddr *addr, socklen_t addrlen) {
	    int s = socket(pf, SOCK_DGRAM, IPPROTO_UDP);
	    if (s < 0)
	        return 0;
	    int ret;
	    do {
	        ret = connect(s, addr, addrlen);
	    } while (ret < 0 && errno == EINTR);
	    int success = (ret == 0);
	    do {
	        ret = close(s);
	    } while (ret < 0 && errno == EINTR);
	    return success;
	}
	
	static int
	_have_ipv6() {
	    static const struct sockaddr_in6 sin6_test = {
	        .sin6_len = sizeof(sockaddr_in6),
	        .sin6_family = AF_INET6,
	        .sin6_port = htons(0xFFFF),
	        .sin6_addr.s6_addr = {
	            0x20, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}
	    };
	    sockaddr_union addr = { .in6 = sin6_test };
	    return _test_connect(PF_INET6, &addr.generic, sizeof(addr.in6));
	}
	
	static int
	_have_ipv4() {
	    static const struct sockaddr_in sin_test = {
	        .sin_len = sizeof(sockaddr_in),
	        .sin_family = AF_INET,
	        .sin_port = htons(0xFFFF),
	        .sin_addr.s_addr = htonl(0x08080808L),  // 8.8.8.8
	    };
	    sockaddr_union addr = { .in = sin_test };
	    return _test_connect(PF_INET, &addr.generic, sizeof(addr.in));
	}

	enum TLocalIPStack {
    	ELocalIPStack_None = 0,
    	ELocalIPStack_IPv4 = 1,
    	ELocalIPStack_IPv6 = 2,
    	ELocalIPStack_Dual = 3,
	};

	void test() {
		TLocal IPlocal_stack = 0;
		int have_ipv4 = _have_ipv4();
	    int have_ipv6 = _have_ipv6();
	    int local_stack = 0;
	    if ( have_ipv4) {
	        local_stack |= ELocalIPStack_IPv4;
	    }
	    if (have_ipv6) {
	        local_stack |= ELocalIPStack_IPv6;
	    }
	}
这里的代码构造成和第一个socket connect的类型，只修改一部分代码。
可以看到，和第一个例子的区别是`socket(pf, SOCK_DGRAM, IPPROTO_UDP)`用了UDP进行连接，UDP可以进行connect，只是进行绑定服务器地址的动作，并不会有网络数据的产生，后续可以直接使用send接口，不需要使用sendto接口（每次都需指定服务器的地址）。</br>
经过测试iOS和Android都能检测出当前可用的IP stack。我们再做一些思考，如果connect接口在UDP的时候，应该是除了TCP发送syn包外的全部事情都做了的。如果这样考虑的话，这个方案成立的依据还是足够的。</br>

###混合的方案（Mac OS，iOS，Linux，Android都支持，Windows/wp待测试）
发现在iOS8/Mac OS上述方案会有点问题（iOS9正常），就是iOS8上IPv6-only网络也会有`169.254.x.x`的自组网的IPv4 stack（其实iOS9上也有，但不影响测试结果），这样会导致IPv4 stack的udp socket能够connect成功（_have_ipv4()返回1）。应对这种情况，我们可以用前面getdefaultgateway的方案，把自组网排除出没有网关的情况。当然，有手机网的时候，IPv4网关是可以获取到的，还是会走到_have_ipv4的路径。当然，如果have_ipv4和have_ipv6只有一个返回1的情况，我们可以认为只有一个IP stack能用。当然如果是local_stack为ELocalIPStack_Dual，还需要用getdnssvraddrs的函数获取当前的dns服务器列表，通过dns服务器的地址确认当前可用的IP stack。必须说明下，这个不是一个准确的判断，如果网络是ELocalIPStack_Dual，但是dns服务只设置了IPv6的地址（如果是dhcp配置的情况，很少出现这样，一般情况都是手工设置才会出现），会判断当前网络为ELocalIPStack_IPv6。这样ELocalIPStack_Dual的网络可能不支持NAT64，这样会导致程序无法访问网络。</br>
这个方案是本地操作，成本低，没有网络流量消耗和耗时问题，暂时是最好的可用IP stack检测方案。（当然NAT64检测不了）
新的实现代码如下：

    TLocalIPStack local_ipstack_detect() {
	    in6_addr addr6_gateway = {0};
	    if (0 != getdefaultgateway6(&addr6_gateway)){ return ELocalIPStack_IPv4;}
	    if (IN6_IS_ADDR_UNSPECIFIED(&addr6_gateway)) { return ELocalIPStack_IPv4;}
	    
	    in_addr addr_gateway = {0};
	    if (0 != getdefaultgateway(&addr_gateway)) { return ELocalIPStack_IPv6;}
	    if (INADDR_NONE == addr_gateway.s_addr || INADDR_ANY == addr_gateway.s_addr ) { return ELocalIPStack_IPv6;}
	    
	    int have_ipv4 = _have_ipv4();
	    int have_ipv6 = _have_ipv6();
	    int local_stack = 0;
	    if (have_ipv4) { local_stack |= ELocalIPStack_IPv4; }
	    if (have_ipv6) { local_stack |= ELocalIPStack_IPv6; }
	    if (ELocalIPStack_Dual != local_stack) { return (TLocalIPStack)local_stack; }
	    
	    int dns_ip_stack = 0;
	    std::vector<socket_address> dnssvraddrs;
	    getdnssvraddrs(dnssvraddrs);
	    
	    for (int i = 0; i < dnssvraddrs.size(); ++i) {
	    if (AF_INET == dnssvraddrs[i].address().sa_family) { dns_ip_stack |= ELocalIPStack_IPv4; }
	    if (AF_INET6 == dnssvraddrs[i].address().sa_family) { dns_ip_stack |= ELocalIPStack_IPv6; }
	    }
	    
	    return (TLocalIPStack)(ELocalIPStack_None==dns_ip_stack? local_stack:dns_ip_stack);
    }

##其他编程问题
建议大家认真看apple的文档[Supporting IPv6 DNS64/NAT64 Networks]和[RFC 4038 - Application Aspects of IPv6 Transition]，里面很多事情都说清楚了，这里说下其他需要关注的地方。

###sockaddr的存储sockaddr_storage
这里千万不要犯傻用`sockaddr`存储`sockaddr_in6`数据，IOS上sockaddr的大小是16，和sockaddr_in一致的，但是sockaddr_in6大小是28(不要问我为什么会知道，都是泪)。通用的sockaddr的存储的结构体是`sockaddr_storage`，它是能存储任何sockaddr的结构。
你可能会问，如果socket用AF_INET6的时候，用`sockaddr_in6`结构体不就好了。不是说不可以，就是代码会变成IPv6专用的了，如果用到其他地方可能会出错。但是如果用AF_INET呢，虽然强转成sockaddr_in没有任何问题，但是程序逻辑上蛋疼，如果大家要写v4/v6通用的逻辑的话，最好还是用`sockaddr_storage`存储，然后通过`ss_family`进行判断，最后做不同分支的处理。

	//sockaddr_storage sample
    socket_address socket_address::getsockname(SOCKET _sock)
    {
	    struct sockaddr_storage addr = {0};
	    socklen_t addr_len = sizeof(addr);
	    ::getsockname(_sock, (sockaddr*)&addr, &addr_len);
	    
	    if (AF_INET == addr.ss_family)
	    {
	   		 return socket_address((const sockaddr_in&)addr);
	    }
	    else if (AF_INET6 == addr.ss_family)
	    {
	    	return socket_address((const sockaddr_in6&)addr);
	    }
	    
	    return socket_address("", 0);
    }

###更加节省空间的方案
`sockaddr_storage`是能够保存所有`sockaddr`下属的类型,但是128字节的大小有时候有点不可接受，而且每次使用都需要做类型转换。下面提供一个更加优雅的方案，大小是28字节，节省了很多。

	union sockaddr_union {
	    struct sockaddr     sa;
	    struct sockaddr_in  in;
	    struct sockaddr_in6 in6;
	} m_addr;

    if (AF_INET == m_addr.sa.sa_family) {
        return ntohs(m_addr.in.sin_port);
    } else if (AF_INET6 == m_addr.sa.sa_family) {
        return ntohs(m_addr.in6.sin6_port);
    }

###NSURLConnection
apple要求大家不要直接用IP访问，不过，中国的DNS环境这么恶劣，没有其他更好的办法。
那NSURLConnection怎么样能够在IPv6访问正常的访问呢？我们应该构建怎么样的URL呢？</br>
我们先看看wikipedia的说法
> 
> **Literal IPv6 addresses in network resource identifiers**  [^4]
> 
> Colon (:) characters in IPv6 addresses may conflict with the established syntax of resource identifiers, such as URIs and URLs. The colon has traditionally been used to terminate the host path before a port number.[6] To alleviate this conflict, literal IPv6 addresses are enclosed in square brackets in such resource identifiers, for example:
> 
>     http://[2001:db8:85a3:8d3:1319:8a2e:370:7348]/
> 
> When the URL also contains a port number the notation is:
> 
>     https://[2001:db8:85a3:8d3:1319:8a2e:370:7348]:443/

我们可以看到除了IP形式不一样外，还多了中括号。当然，上面我们说到**IPv4-mapped IPv6 address**和**NAT64 mapped address**同样也是适用的,例如：`http://[::ffff:14.17.32.21]:80`走IPv4协议栈或`http://[64:ff9b::14.17.32.21]`走NAT64。关键点还是判断当前
**客户端可用的IP stack**。

###IOS下CoreFoudation或者更高级的API
引用手Q同事的原话：
> 如果使用CoreFoudation或者更高级的API,即使在纯IPv6环境下使用IPv4的ip进行网络通信，iOS9会自动把IPv4地址转换成IPv6地址。换句话说，因为手q里面大部分api都是满足要求的，基本上不用改动。（注意iOS9.0或以上）

例如CFStreamCreatePairWithSocketToCFHost　..待续

###DNS API
apple的文档说了，gethostbyname这些已经不能用了(只支持IPv4)，只能用getaddrinfo。

	struct addrinfo hints, *res, *res0;
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = PF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_DEFAULT;
    getaddrinfo("www.qq.com", "http", &hints, &res0);

这里sample比较简单，其实getaddrinfo的重点在`hints.ai_family`和`hints.ai_flags`的设置上，apple已经给出了一个很好sample。我们分析下这两个变量不同设置下的效果，看看有什么区别。

-	`hints.ai_family = PF_UNSPEC`的意思是v4地址和v6地址都返回，不过呢，这里可是会触发两个UDP的请求，当年微信就给运营商吐槽过，你没有v6地址，就不要做v6请求拉（微信量大）。不过apple爸爸要求用v6地址，怎么办？
-	`hints.ai_family = PF_INET`的意思是只返回v4地址
-	`hints.ai_family = PF_INET6`的意思是只返回v6地址
-	`hints.ai_flags |= AI_V4MAPPED 且 hints.ai_family = PF_INET6`的情况下，如果需要dns的host没有v6地址的情况下，getaddinfo会把v4地址转换成`v4-mapped ipv6 address`，如果有v6地址返回就不会做任何动作。
-	`hints.ai_flags |= AI_ADDRCONFIG`这个是一个很有用的特性，这个flags表示getaddrinfo会根据本地网络情况，去掉不支持的IP协议地址。
-	`hints.ai_flags = AI_DEFAULT`其实就是`AI_V4MAPPED|AI_ADDRCONFIG_CFG`，也是apple推荐的flags设置方式。

> 域名  对应着如下 IP 地址:</br>
> 173.194.127.180</br>
> 173.194.127.176</br>
> 2404:6800:4005:802::1010</br>
> 若本地主机仅配置了 IPV4 地址,则返回的查询结果中不包含 IPV6 地址,即此时只有:</br>
> 173.194.127.180</br>
> 173.194.127.176</br>
> 同样若本地主机仅配置了 IPV6 地址,则返回的查询结果中仅包含IPV6地址.
> 2404:6800:4005:802::1010</br>

用这个API的时候，建议大家还是按照apple的sample来做`hints.ai_family`暂时先`PF_INET`,免得运营商投诉，当然最好是能后台进行控制。

下面一段话是apple文档内对getaddrinfo对NAT64支持的描述。
> The current implementation supports synthesis of NAT64 mapped IPv6 addresses.  If hostname is a numeric string defining an IPv4 address (for example, '192.0.2.1' ) and ai_family is set to PF_UNSPEC or PF_INET6, getaddrinfo() will synthesize the appropriate IPv6 address(es) (for example, '64:ff9b::192.0.2.1' ) if the current interface supports IPv6, NAT64 and DNS64 and does not support IPv4. If the AI_ADDRCONFIG flag is set, the IPv4 address will be suppressed on those interfaces.  On non-qualifying interfaces, getaddrinfo() is guaranteed to return immediately without attempting any resolution, and will return the IPv4 address if ai_family is PF_UNSPEC or PF_INET. NAT64 address synthesis can be disabled by setting the AI_NUMERICHOST flag. To best support NAT64 networks, it is recommended to resolve all IP address literals with ai_family set to PF_UNSPEC and ai_flags set to AI_DEFAULT.

可以看到apple最推荐的getaddrinfo用法就是sample那样。

###iOS SCNetworkReachabilityCreateWithAddress API问题
在iOS下，一般判断网络连通性和WIFI/Mobile网络的判断是使用SCNetworkReachabilityCreateWithAddress API，一般的sample里面只会测试IPv4的IP地址，这样有可能导致在纯IPv6网络下判断出当前是没有网络，这样明显是不对的。

    struct sockaddr_in zeroAddress;
    bzero(&zeroAddress, sizeof(zeroAddress));
    zeroAddress.sin_len = sizeof(zeroAddress);
    zeroAddress.sin_family = AF_INET;
    
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr*)zeroAddress);

针对IPv6可以使用下面的方式来判断

    struct sockaddr_in6 zeroAddress6;
    bzero(&zeroAddress6, sizeof(zeroAddress6));
    zeroAddress6.sin6_len = sizeof(zeroAddress6);
    zeroAddress6.sin6_family = AF_INET6;
    
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr*)zeroAddress6);

当然大家写通用代码的时候，可以IPv6和IPv4都判断。</br>
最后苹果建议的方式是SCNetworkReachabilityCreateWithName这个API，个人暂时不确定这个API是不是会进行DNS解析。

##文档的推荐和说明
[Beej's Guide to Network Programming 简体中文](http://www.netdpi.net/files/beej/Beej-cn-20140429.zip) 这本书不错的，介绍了很多API的使用，当然IPv6部分也有</br>
[Beej's Guide to Network Programming 繁体中文](http://beej-zhtw.netdpi.net) 排版比简体好</br>
unix network programming 不用说了，不过没有v6的部分</br>
[Dual-Stack Sockets for IPv6 Winsock Applications(Windows XP SP1后都支持)](https://msdn.microsoft.com/zh-cn/library/windows/desktop/bb513665(v=vs.85).aspx)
##IPv6下不能使用的API列表
-	gethostbyname()
-	gethostbyaddr()
-	getservbyname()
-	getservbyport()
-	gethostbyname2()
-	inet_addr()
-	inet_aton()
-	inet_lnaof()
-	inet_makeaddr()
-	inet_netof()
-	inet_network()
-	inet_ntoa()
-	inet_ntoa_r()
-	bindresvport()
-	getipv4sourcefilter()
-	setipv4sourcefilter()

下面类型或者结构需要注意使用的正确性

IPv4|IPv6
---|---
AF_INET|AF_INET6
PF_INET|PF_INET6
struct in_addr|struct in_addr6
struct sockaddr_in|struct sockaddr_in6
kDNSServiceProtocol_IPv4|kDNSServiceProtocol_IPv6


[Supporting IPv6 DNS64/NAT64 Networks]:https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/UnderstandingandPreparingfortheIPv6Transition/UnderstandingandPreparingfortheIPv6Transition.html#//apple_ref/doc/uid/TP40010220-CH213-SW11 (Supporting IPv6 DNS64/NAT64 Networks)
[RFC 4038 - Application Aspects of IPv6 Transition]:https://www.ietf.org/rfc/rfc4038 (RFC 4038 - Application Aspects of IPv6 Transitio)
[^1]: [wikipedia Transition from IPv4](https://en.wikipedia.org/wiki/IPv6_address#Transition_from_IPv4)
[^2]: [wikipedia NAT64](https://en.wikipedia.org/wiki/IPv6_address#NAT64)
[^3]: [wikipedia DNS64](https://en.wikipedia.org/wiki/IPv6_address#DNS64)
[^4]: [wikipedia IPv6 address](https://en.wikipedia.org/wiki/IPv6_address#Literal_IPv6_addresses_in_network_resource_identifiers)


