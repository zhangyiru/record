## ethtool使用

ethtool -S|--statistics DEVNAME	Show adapter statistics



```
[root@euleros-pxe linux-4.18.0-147.5.1.el8_1]# cat /proc/net/dev | column -t
Inter-|      Receive      |          Transmit                                                                                                                   
face         |bytes       packets    errs      drop  fifo  frame  compressed  multicast|bytes  packets       errs       drop  fifo  colls  carrier  compressed  
enp129s0f0:  0            0          0         0     0     0      0           0                0             0          0     0     0      0        0           0
enp129s0f1:  0            0          0         0     0     0      0           0                0             0          0     0     0      0        0           0
enp2s0f2:    0            0          0         0     0     0      0           0                0             0          0     0     0      0        0           0
eth_main:    41522915343  100126486  0         0     0     0      0           1089266          127526207045  126962138  0     0     0      0        0           0
enp2s0f1:    0            0          0         0     0     0      0           0                0             0          0     0     0      0        0           0
enp2s0f3:    0            0          0         0     0     0      0           0                0             0          0     0     0      0        0           0
lo:          31392        220        0         0     0     0      0           0                31392         220        0     0     0      0        0           0
```



/proc/nev/dev中的drop对应于rx_dropped和rx_missed_errors

```c
static void dev_seq_printf_stats(struct seq_file *seq, struct net_device *dev)
{
	...
	stats->rx_dropped + stats->rx_missed_errors,
}
```



```shell
[root@euleros-pxe linux-4.18.0-147.5.1.el8_1]# ethtool -S eth_main | grep drop
     dropped_smbus: 0
     tx_dropped: 0
     rx_queue_0_drops: 0
     rx_queue_1_drops: 0
     rx_queue_2_drops: 0
     rx_queue_3_drops: 0
     rx_queue_4_drops: 0
     rx_queue_5_drops: 0
     rx_queue_6_drops: 0
     rx_queue_7_drops: 0
[root@euleros-pxe linux-4.18.0-147.5.1.el8_1]# ethtool -S eth_main | grep miss
     rx_missed_errors: 0
```



https://blog.huoding.com/2020/04/27/814

```shell
[root@localhost statistics]# ethtool -S bond0
no stats available
```

 kvm 的 virtio_net 不支持 statistics

```shell
[root@localhost ~]# find /sys -name bond0
/sys/class/net/bond0
/sys/devices/virtual/net/bond0
[root@localhost ~]# cd /sys/devices/virtual/net/bond0
[root@localhost bond0]# cd statistics/
[root@localhost statistics]# ls
collisions  rx_bytes       rx_crc_errors  rx_errors       rx_frame_errors   rx_missed_errors  rx_over_errors  tx_aborted_errors  tx_carrier_errors  tx_dropped  tx_fifo_errors       tx_packets
multicast   rx_compressed  rx_dropped     rx_fifo_errors  rx_length_errors  rx_nohandler      rx_packets      tx_bytes           tx_compressed      tx_errors   tx_heartbeat_errors  tx_window_errors
```



1、find /sys -name bond0

2、选择/sys/devices目录的那个路径，类似于/sys/devices/virtual/net/bond0

进入statistics目录把rx_missed_errors和rx_dropped多次打印下
