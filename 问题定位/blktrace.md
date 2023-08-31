1、收集设备/dev/sdb上的io情况60秒，将结果保存到sdb-trace文件中 

blktrace -d /dev/sdb -o sdb-trace -w 60 

2、将sdb-trace开头的所有文件合并为一个sdb-trace.bin，这个过程中的输出放在sdb-trace.txt中 

blkparse -i sdb-trace -d sdb-trace.bin -o sdb-trace.txt 

3、通过btt工具知道磁盘I/O在每一个阶段耗时分布 

btt -i sdb-trace.bin -o btt.txt 

其中btt.txt.avg是请求信息分布情况



