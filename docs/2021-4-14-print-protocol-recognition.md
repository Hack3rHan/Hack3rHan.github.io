# 初探常见网络打印协议的识别



## 0x01 常见网络打印协议

1. LPD/LPR 协议 （TCP 515）
2. IPP协议 （TCP 631）
3. AppSocket / Raw / JetDirect （TCP 9100）
4. SMB 、Telnet
5. Air Print、Mopria Alliance



## 0x02 LPD协议识别

LPD/LPR 全称 Line printer daemon protocol，默认使用TCP 515端口。

迫于没有打印机，协议的数据包只能Fofa、Nmap、Goby等方式获取了。（研究打印协议全程没有打印机，实惨。）

实践发现，Goby + Fofa可以实现较好的分析效果，经过对数台主机的抓包验证以及使用socket手动发包验证，可以得到Goby对目标主机进行探测所得响应报文内容与Fofa列出的响应报文一致。（情理之中）

Goby发送的报文：

```
# Hex
0371 7565 7565 3a4c 5054 3120 010a 
# Bytes
\x03queue:LPT1 \x01\n
```

Fofa获取到的部分响应：

```
printer 'queue:LPT1' has illegal character at ':LPT1' in name\n

Printer Name: queue:LPT1\nJobs: No Jobs in Queue\n

PortThru lpd: No Jobs on this queue\n

Unable to get printer queue:LPT1: successful-ok

JetDirect lpd: no jobs queued on the port Auto\n

\r\n                        Windows LPD \xbc\xad\xb9\xf6\xbf\xc0\xb7\xf9:\xc1\xf6\xc1\xa4\xc7\xd1\xc7\xc1\xb8\xb0\xc5\xcd\xb0\xa1\xc1\xb8\xc0\xe7\xc7\xcf\xc1\xf6\xbe\xca\xbd\xc0\xb4\xcf\xb4\xd9.\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01

\r\n                        Windows LPD ServerError: specified printer does not exist\x00n\x00a\x00\x03\x00\x00\x03\xff\x0c\x00\x00\xa0\xff%\x00\x00\x00\x00\x00X\x01#\x00\x00\x00\x00\x00\x00\x01
```

发送报文分析：

发送的数据16进制形式为 0371 7565 7565 3a4c 5054 3120 010a

第1个字节，根据RFC 1179第5部分的说明，0x03表示Send queue state (short)，即发送队列状态。

第2到第11字节，queue:LPT1为队列名。

第12字节为空格。

第13字节为操作数，0x01表示打印机队列名。

第14字节为换行符。

参考RFC 1179 5.3，操作数等格式规范参考该文档上半部分。

> ```
> 5.3 03 - Send queue state (short)
> 
>       +----+-------+----+------+----+
>       | 03 | Queue | SP | List | LF |
>       +----+-------+----+------+----+
>       Command code - 3
>       Operand 1 - Printer queue name
>       Other operands - User names or job numbers
> ```

响应报文分析：

开头5条报文疑似来自打印机，有待确认（没设备）。

```
printer 'queue:LPT1' has illegal character at ':LPT1' in name\n

Printer Name: queue:LPT1\nJobs: No Jobs in Queue\n

PortThru lpd: No Jobs on this queue\n

Unable to get printer queue:LPT1: successful-ok

JetDirect lpd: no jobs queued on the port Auto\n
```

后两条包含Windows LPD响应的报文疑似来自Windows LPD打印服务和Windows LPR 端口监视器（ LPD Print Service and LPR Port Monitor）。

![](..\img\2021-4-14-1.PNG)



## 0x03 IPP 协议识别

IPP协议全称Internet Printing Protocol，默认使用TCP 631端口，该协议基于HTTP实现，新标准里亦有基于HTTPS的实现。

我们同样使用Goby和Wireshark获取请求-响应样例。

```http
POST /never_could_exists HTTP/1.1
Host: 125.134.126.211:631
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_3) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0.3 Safari/605.1.15
Content-Length: 14
Cookie: rememberMe=+2ewny/4XWAbAc9751f01TtmSLcxlwGmpMy2PSdrBq0=
Accept-Encoding: gzip

{"abc":"123",}
```

```http
HTTP/1.1 404 Not Found
Connection: close
Content-Language: en_US
Content-Length: 342
Content-Type: text/html; charset=utf-8
Date: Wed, 14 Apr 2021 01:43:36 GMT
Accept-Encoding: gzip, deflate, identity
Server: CUPS/2.0 IPP/2.1
Set-Cookie: rememberMe=+2ewny/4XWAbAc9751f01TtmSLcxlwGmpMy2PSdrBq0=; path=/; httponly;

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<HTML>
<HEAD>
	<META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=utf-8">
	<TITLE>Not Found - CUPS v2.0.0</TITLE>
	<LINK REL="STYLESHEET" TYPE="text/css" HREF="/cups.css">
</HEAD>
<BODY>
<H1>Not Found</H1>
<P></P>
</BODY>
</HTML>
```

经过对Fofa大量样例分析，该协议在Server头和Title等位置具有明显特征。

该协议的深度探究以后进行。

## 0x03 AppSocket 识别



AppSocket又名Raw、JetDirect，默认使用TCP 9100端口。

同样跟一下TCP流，获取一些通信的数据。

```
# 发送
@PJL RDYMSG DISPLAY = "rdymsgarg"
@PJL INFO STATUS

# 接收
@PJL INFO STATUS
CODE=10002
DISPLAY="Ready"
ONLINE=FALSE
.

# 发送
@PJL INFO ID

# 接收
@PJL INFO ID
"SAMSUNG C4060FX"
.

# 发送
@PJL INFO PRODINFO
```

AppSocket使用TCP通信，使用PJL（Printer Job Language，打印机工作语言）交互，从上述报文中可以较清晰地看出使用该语言获取信息的过程。

下面是一些可用于获取信息的PJL。

```
# 获取打印机型号：
@PJL INFO ID

# 获取打印机状态：
@PJL INFO STATUS

# 获取打印机变量
@PJL INFO VARIABLES

# 获取打印机配置
@PJL INFO CONFIG

# 获取打印机计数
@PJL INFO PAGECOUNT
```

更多PJL可以从HP提供的文档中获取。



## 0x04 其他协议说明

SMB 、Telnet等方式待后续有研究设备后继续深入研究。

Air Print、Mopria Alliance等协议采用 mDNS进行设备发现，IPP协议执行打印，常用于无线网络环境，本次暂不继续深入研究。