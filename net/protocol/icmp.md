# icmp
ref:
- ICMP数据报文格式(https://blog.csdn.net/qq_35733751/article/details/80052233)
- [ICMP协议格式](https://tonydeng.github.io/sdn-handbook/basic/icmp.html)
- [使用 Linux tracepoint、perf 和 eBPF 跟踪数据包 (2017)](https://github.com/DavadDi/bpf_study/blob/master/trace-packet-with-tracepoint-perf-ebpf/index_zh.md)

ICMP报文是在IP数据报内部传输的格式`| IP头部(前20B) | ICMP报文 |`

ICMP 报文:
- header(8B):

    - 类型(type, 1B)：ICMP报文类型，用于标识错误类型的差错报文或者查询类型的报告报文
    - 代码(code, 1B)：根据ICMP差错报文的类型，进一步分析错误的原因，代码值不同对应的错误也不同，例如：类型为11且代码为0，表示数据传输过程中超时了，超时的具体原因是TTL值为0，数据报被丢弃
    - 校验和(checksum, 2B)：数据发送到目的地后需要对ICMP数据报文做一个校验，用于检查数据报文是否有错误
    - 标识符(Identifier, 2B)：对于每一个发送的数据报进行标识
    - 序列号(Sequence number, 2B)：对于发送的每一个数据报文进行编号，比如：发送的第一个数据报序列号为1，第二个序列号为2
- body(64B)

req demo:
```
Internet Control Message Protocol
Type: 8 (Echo (ping) request)          // ICMP类型为8，说明这是一个ICMP请求查询报文 
Code: 0                           //表示是请求
Checksum: 0xf7e9 [correct]           //校验和
[Checksum Status: Good]             //校验和状态，good表示校验和正确，bad表示数据报被修改或者发生错误
Identifier (BE): 1 (0x0001)             //标识，用于区分在linux下抓包，基本上抓的每一个ICMP包都是1
Identifier (LE): 256 (0x0100)           //标识，用于区分在windows下抓包，基本上抓的每一个ICMP包都是256
Sequence number (BE): 21 (0x0015)    //序列号，用于区分在linux下抓包，每一个ICMP包的序列号都不一样
Sequence number (LE): 5376 (0x1500)  //序列号，用于区分在windows下抓包，每一个ICMP包的序列号都不一样
[No response seen]               
Data (64 bytes)                    //数据部分
```

other example:
```bash
# tcpdump -nn -i eth0 icmp and host 172.16.25.157  -X -vvv
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
13:53:35.498895 IP (tos 0x0, ttl 64, id 6680, offset 0, flags [DF], proto ICMP (1), length 84)
    172.16.25.157 > 172.16.25.1: ICMP echo request, id 20391, seq 1, length 64
        0x0000:  4500 0054 1a18 4000 4001 95d2 ac10 199d  E..T..@.@.......
        0x0010:  ac10 1901 0800 8746 4fa7 0001 df3a c066  .......FO....:.f
        0x0020:  0000 0000 bb9c 0700 0000 0000 1011 1213  ................
        0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!"#
        0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&'()*+,-./0123
        0x0050:  3435 3637                                4567
13:53:35.500524 IP (tos 0x0, ttl 254, id 6680, offset 0, flags [DF], proto ICMP (1), length 84)
    172.16.25.1 > 172.16.25.157: ICMP echo reply, id 20391, seq 1, length 64
        0x0000:  4500 0054 1a18 4000 fe01 d7d1 ac10 1901  E..T..@.........
        0x0010:  ac10 199d 0000 8f46 4fa7 0001 df3a c066  .......FO....:.f
        0x0020:  0000 0000 bb9c 0700 0000 0000 1011 1213  ................
        0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!"#
        0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&'()*+,-./0123
        0x0050:  3435 3637 
```

ICMP echo request(type=8)是请求,  ICMP echo reply(type=0)是响应.

追踪ping包`perf trace --no-syscalls --event 'net:*' ping 172.17.0.2 -c1 > /dev/null`