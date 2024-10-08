---

title: 常用的shell脚本和一些好用的命令

date: 2020-08-05 13:45:09

tags: [Linux CLI]

categories: [Linux学习历程]

---

> 使用不同的解释器「bash,zsh...」编写脚本的时候对于[某些语法可能不同](https://apple.stackexchange.com/questions/361870/what-are-the-practical-differences-between-bash-and-zsh#:~:text=Bash%20uses%20.,t%20always%20a%20perfect%20equivalence.)
>
>example：Bash **arrays** are indexed from 0 to (length-1). Zsh arrays are indexed from 1 to length。

##### grep相关

```sh
# 将展示结果的各个参数的header信息也展示出来
grep -E 'nginx|USER' filename
egrep 'nginx|USER' filename
grep --colour=auto -n -H -C  5 'Exception' a.xkx
# -i 忽视大小写 ignore-case
grep --colour=auto -i 'exception' a.xkx
```
##### find相关

```sh
# 使用-exec查找并执行正则匹配
find . -name 'nohup.log-2020-07-22*' -exec grep --colour=auto -n -H -C  5 'SF2020072217505400001' {} \;
```
##### 批量处理文件的一个大概流程:

```sh
#!/bin/bash

#IFS=,
#numbers="01,02,03,04,05";
numbers=(01 02 03 04 05);
dir='/Users/xukaixuan/git/Lua-Quick-Start-Guide/';

for ((i=0;i<${#numbers[*]};i++))
do
  path="Chapter${numbers[i]}";
  curPath=$dir${path};
  echo "current path: $curPath";
  find $curPath -name "*.lua" -print0 | while IFS= read -r -d '' file; do 
        sed -i '1i #!/usr/local/bin/lua' $file
        chmod u+x $file        
    done
done
```
##### ps相关

```sh
# 查看的时候把字段名字也展示出来,方便查看
ps aux | grep -E '(nginx|USER)'
```

##### 一次批量删除redis中匹配的key

```sh
redis-cli -h 192.168.4.4 -n 3 -a test --scan --pattern 'xboot:subsernorepeat:*' | xargs redis-cli -h 192.168.4.4 -n 3 -a test -n 3 del
```
##### 使用awk拼接sql

```sh
# test 源文件
# test_result 结果文件
awk -F ',' '{print "UPDATE company_account_type SET `virtual_account` = "$1"  WHERE `account_no` = "$2" AND `account_type` = "$3" AND is_deleted = 0;"}' test > test_result
```

##### maven依赖相关

```sh
# 项目依赖树
mvn dependency:tree >> project.tree
# 分析当前项目的依赖
mvn dependency:analyze >> project.analyze
```

##### socket统计信息，端口使用相关

> ss from iproute2 
> 
> 命令使用详细参考 [这里](https://wangchujiang.com/linux-command/c/ss.html)

```sh
# 说端口的时候要注意是什么协议下的端口
# 经Mac下测试使用，Linux下执行需要root权限
# TCP ^LISTEN 可以查看除了LISTEN之外的状态
# 每个参数的意思建议 man lsof查看
lsof -nP -iTCP -sTCP:LISTEN
# UDP
lsof -nP -iUDP
# 经Centos下测试使用
# 显示套接字（socket）使用概况
ss -s
# 显示监听状态的tcp套接字（socket）使用概况，显示进程，显示ip，同样看进程号也需要root权限
sudo ss -tlnp

#LISTEN状态下 也就是-l下
#Recv-Q：当前全连接队列的大小，也就是当前已完成三次握手并等待服务端 accept() 的 TCP 连接；
#Send-Q：当前全连接最大队列长度，上面的输出结果说明监听 8088 端口的 TCP 服务，最大全连接长度为 128；

# 查看TCP 全连接队列溢出统计
netstat -s | grep overflowed
#Linux 有个参数可以指定当 TCP 全连接队列满了会使用什么策略来回应客户端。丢弃连接只是 Linux 的默认行为，我们还可以选择向客户端发送 RST 复位报文，告诉客户端连接已经建立失败。
cat /proc/sys/net/ipv4/tcp_abort_on_overflow # 默认是0
# tcp_abort_on_overflow 共有两个值分别是0和1，其分别表示：
#0 如果全连接队列满了，那么server扔掉client发过来的ack 
#1 如果全连接队列满了，server发送一个reset包给client，表示废掉这个握手过程和这个连接

#非LISTEN状态
#Recv-Q：已收到但未被应用进程读取的字节数；
#Send-Q：已发送但未收到确认的字节数

# 显示所有状态为established的SMTP连接
ss -o state established '( dport = :smtp or sport = :smtp )' 
# 显示所有状态为Established的HTTP连接
ss -o state established '( dport = :http or sport = :http )' 
# 列举出处于 FIN-WAIT-1状态的源端口为 80或者 443，目标网络为 193.233.7/24所有 tcp套接字
ss -o state fin-wait-1 '( sport = :http or sport = :https )' dst 193.233.7/24  
```

##### 网络io监测相关

> nload 命令 网路状态和各IP所使用的频宽。[Detail](https://www.tecmint.com/nload-monitor-linux-network-traffic-bandwidth-usage/#:~:text=nload%20is%20a%20command%2Dline,and%20min%2Fmax%20network%20usage.)
> 
> nethogs 命令 按进程或程序实时统计网络带宽使用率。它只能实时监控进程的网络带宽占用情况。[Detail](https://www.tecmint.com/nethogs-monitor-per-process-network-bandwidth-usage-in-real-time/#:~:text=NetHogs%20is%20an%20open%20source,small%20'net%20top'%20tool.)
> 
>  dstat 命令 性能测试、基准测试和排除故障过程中可以很方便监控系统运行状况。可以将详细信息通过cvs输出到一个文件。[Detail](https://blog.csdn.net/yue530tomtom/article/details/75443305)
> 
>  sar 命令 Linux系统运行状态统计和性能分析工具，可从磁盘IO、CPU负载、内存使用等多个维度对系统活动进行报告。[Detail](http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/sar.html)
> 
> iptraf 命令 会有简单界面显示出来详细信息 [Detail](https://blog.csdn.net/quiet_girl/article/details/50777210) 
> 
> iftop 命令
> 
> tcpdump 抓包命令  [Detail](https://juejin.im/post/6844904084168769549)
> 
> RX==receive，接收，从开启到现在接收封包的情况，是下行流量。
>  
> TX==Transmit，发送，从开启到现在发送封包的情况，是上行流量。
> 
> 

```sh
# 经Mac下测试使用
# 不显示流量图，只显示统计数据。
nload -m
# 经Centos下测试使用
# 监测eth0网卡的流量
nload eth0

```

##### 词典的使用
> aspell 命令对于各种文件内容进行拼写检查

```sh
#  经Mac下测试使用
# 然后进入交互模式，可以给出的单词进行拼写建议
aspell -a 

# 交互式检查文件内容单词拼写情况
aspell check file.xkx
```

##### 进程文件打开情况

```sh
# 29190 进程号
ls -l /proc/29190/fd

lsof -p 29190 

```
##### 磁盘情况查看

> du是面向文件的命令，只计算被文件占用的空间，不计算文件系统 metadata 占用的空间。
  df则是基于文件系统总体来计算，通过文件系统中未分配空间来确定系统中已经分配空间的大小。
>df命令可以获取硬盘占用了多少空间，还剩下多少空间，它也可以显示所有文件系统对i节点和磁盘块的使用情况。
>

```sh
df -hl 查看磁盘剩余空间
df -h 查看每个根路径的分区大小
du -sh [目录名] 返回该目录的大小
du -sm [文件夹] 返回该文件夹总M数
```

##### ab的使用
> ab就是Apache Benchmark的缩写，Apache组织开发的一款web压力测试工具，优点是使用方便，统计功能强大。一些参数分析参考这里：https://www.jianshu.com/p/6175456a55be 当然可以直接 `man ab`

``` sh
#  经Mac下测试使用
#  -n 总共多少次请求
#  -c 多少个线程并发请求
#  -t 测试持续多长时间(second)

# GET 请求
ab -n 300 -c 30 'http://localhost:8098/test/primary'

# POST 请求

ab -n 100 -c 10 -p post_data.xkx -T 'application/json' 'http://localhost:8098/query'

```

##### nc的使用
Netcat(often abbreviated to nc) is a featured networking utility which **reads and writes data across network connections, using the TCP/IP protocol.**It is designed to be a reliable "back-end" tool that can be used directly or easily driven by other programs and scripts. At the same time, it is a feature-rich network debugging and exploration tool, since it can create almost any kind of connection you would need and has several interesting built-in capabilities

``` sh
#  经Mac下测试使用
#  -n 总共多少次请求
#  -c 多少个线程并发请求
#  -t 测试持续多长时间(second)

# GET 请求
ab -n 300 -c 30 'http://localhost:8098/test/primary'

# POST 请求

ab -n 100 -c 10 -p post_data.xkx -T 'application/json' 'http://localhost:8098/query'

```



##### 网络相关

> 路由相关：route，traceroute，mtr



##### 一些常用的处理文件的脚本

> 从一次面试决定开始记录，提交shell编写能力，想到可以从leecode上面看一些shell题目
>
> 另一方面是对于一些简单的命令去使用和理解

###### 
文本中出现某些字符频率排名

```sh
 cat num.xkx | sort | uniq -c | sort -nrk 1 | awk -F ' ' '{print $2}' | head -5
```






##### 参考

- [Linux System Monitoring Tools Every SysAdmin Should Know](https://www.cyberciti.biz/tips/top-linux-monitoring-tools.html)