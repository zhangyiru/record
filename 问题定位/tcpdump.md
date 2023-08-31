tcpdump -i any host *.*.*.* -nne -w  /tmp/130.pcap 

-nn是关闭反向查询功能并以数字格式显示ip地址端口号，和url地址。不带这个选项回显会很慢。

-e用于显示对应的源，目的的mac地址



## 指定 源地址 目的地址

tcpdump src host a or dst host b -w /tmp/228.cap
