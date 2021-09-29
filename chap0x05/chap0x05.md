##  实验五：基于 Scapy 编写端口扫描器



### 实验目的

- 掌握网络扫描之端口状态探测的基本原理

### 实验环境

- python + [scapy](https://scapy.net/)

- 攻击者主机（Attacker）：Kali-Linux-2021.2

- 网关（Gateway）：Debian 10

- 靶机（Victim）：Kali-Linux-2021.2


### 实验要求(完成度)

- [x] 禁止探测互联网上的 IP ，严格遵守网络安全相关法律法规
- [x] 完成以下扫描技术的编程实现
  - TCP connect scan / TCP stealth scan
  - TCP Xmas scan / TCP fin scan / TCP null scan
  - UDP scan
- [x] 上述每种扫描技术的实现测试均需要测试端口状态为：`开放`、`关闭` 和 `过滤` 状态时的程序执行结果
- [x] 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因；
- [x] 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的
- [x] （可选）复刻 `nmap` 的上述扫描技术实现的命令行参数开关

### Scapy 基础

```python
# 导入模块
from scapy.all import *
# 查看包信息
pkt = IP(dst="")
ls(pkt)
pkt.show()
summary(pkt)
# 发送数据包
send(pkt)  # 发送第三层数据包，但不会受到返回的结果。
sr(pkt)  # 发送第三层数据包，返回两个结果，分别是接收到响应的数据包和未收到响应的数据包。
sr1(pkt)  # 发送第三层数据包，仅仅返回接收到响应的数据包。
sendp(pkt)  # 发送第二层数据包。
srp(pkt)  # 发送第二层数据包，并等待响应。
srp1(pkt)  # 发送第二层数据包，并返回响应的数据包
# 监听网卡
sniff(iface="wlan1",count=100,filter="tcp")
# 应用：简单的SYN端口扫描 （测试中）
pkt = IP("...")/TCP(dport=[n for n in range(22, 3389)], flags="S")
ans, uans = sr(pkt)
ans.summary() # flag为SA表示开放，RA表示关闭
```
### 实验原理

**参考链接：**

1. [B站授课视频](https://www.bilibili.com/video/BV1CL41147vX?p=49)
2. [在线课本](https://c4pr1c3.github.io/cuc-ns/chap0x05/main.html)

**复现一下要点：**

- TCP Connect扫描

| 序号 | 通信方向 | 流程 1      | 流程 2      | 流程 3                  |
| :--- | :------- | :---------- | :---------- | :---------------------- |
| 1    | C -> S   | SYN+Port(n) | SYN+Port(n) | SYN+Port(n)             |
| 2    | S -> C   | SYN/ACK     | RST         | 无响应/其他拒绝反馈报文 |
| 3    | C -> S   | ACK         |             |                         |
| 4    | C -> S   | RST         |             |                         |
|      | 状态推断 | 开放 ✅      | 关闭 ⛔      | 被过滤 ⚠️                |

- TCP stealth scan

| 序号 | 通信方向 | 流程 1      | 流程 2      | 流程 3                  |
| :--- | :------- | :---------- | :---------- | :---------------------- |
| 1    | C -> S   | SYN+Port(n) | SYN+Port(n) | SYN+Port(n)             |
| 2    | S -> C   | SYN/ACK     | RST         | 无响应/其他拒绝反馈报文 |
| 3    | C -> S   | RST         |             |                         |
|      | 状态推断 | 开放 ✅      | 关闭 ⛔      | 被过滤 ⚠️                |

`TCP connect scan` 与 `TCP stealth scan` 都是先发送一个S，然后等待回应。如果有回应且标识为R，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时

`TCP connect scan`会回复一个RA，在完成三次握手的同时断开连接

`TCP stealth scan`只回复一个R，不完成三次握手，直接取消建立连接

- TCP Xmas scan

| 序号 | 通信方向 | 流程 1                            | 流程 2                            |
| :--- | :------- | :-------------------------------- | :-------------------------------- |
| 1    | C -> S   | TCP FIN(1),PUSH(1),URG(1)+Port(n) | TCP FIN(1),PUSH(1),URG(1)+Port(n) |
| 2    | S -> C   | RST                               | 无响应/其他拒绝反馈报文           |
|      | 状态推断 | 关闭 ⛔                            | 开放 ✅ / 关闭 ⛔ / 被过滤 ⚠️        |

- TCP fin scan

| 序号 | 通信方向 | 流程 1          | 流程 2                     |
| :--- | :------- | :-------------- | :------------------------- |
| 1    | C -> S   | TCP FIN+Port(n) | TCP FIN+Port(n)            |
| 2    | S -> C   | RST             | 无响应/其他拒绝反馈报文    |
|      | 状态推断 | 关闭 ⛔          | 开放 ✅ / 关闭 ⛔ / 被过滤 ⚠️ |

- TCP null scan

| 序号 | 通信方向 | 流程 1                            | 流程 2                            |
| :--- | :------- | :-------------------------------- | :-------------------------------- |
| 1    | C -> S   | TCP FIN(0),PUSH(0),URG(0)+Port(n) | TCP FIN(0),PUSH(0),URG(0)+Port(n) |
| 2    | S -> C   | RST                               | 无响应/其他拒绝反馈报文           |
|      | 状态推断 | 关闭 ⛔                            | 开放 ✅ / 关闭 ⛔ / 被过滤 ⚠️        |

`TCP Xmas scan`  `TCP fin scan`  `TCP null scan`这三种扫描方式都属于**隐蔽扫描**，它们的优点是隐蔽性比`TCP connect scan` 与 `TCP stealth scan`好，但都需要自己构造数据包，要求由超级用户或者授权用户访问专门的系统调用。

`TCP Xmas scan`要对TCP 报文头 FIN、URG 和 PUSH 标记进行设置，如果端口关闭则回复R，其他状态无响应

  `TCP fin scan`  ，fin包可以直接通过防火墙，所以端口开放和过滤都对fin包无影响，不会响应，但端口关闭会响应R

`TCP null scan`则是关闭TCP所有的报文头标记，所以也是只有关闭的端口会响应R，其他状态不响应

- UDP scan

| 序号 | 通信方向 | 流程 1               | 流程 2                     |
| :--- | :------- | :------------------- | :------------------------- |
| 1    | C -> S   | UDP+Port(n)          | UDP+Port(n)                |
| 2    | S -> C   | UDP+port(n) 响应数据 | 无响应/其他拒绝反馈报文    |
|      | 状态推断 | 开放 ✅               | 开放 ✅ / 关闭 ⛔ / 被过滤 ⚠️ |



### 实验过程

#### 网络拓扑
![topological](img/topological.jpg)

使用的拓扑图类似第四章实验的拓扑结构：Attacker作为扫描端，Victim作为被扫描的靶机。



#### 端口状态模拟

在**靶机**上安装`ufw`

```
sudo apt-get update 
sudo apt install ufw
```

- **关闭状态**：对应端口没有开启监听, 防火墙没有开启。
  
  ```bash
  ufw disable
  systemctl stop apache2 #关闭端口80
  systemctl stop dnsmasq #关闭端口53
  ```
- **开启状态**：对应端口开启监听: `apache2`基于TCP, 在80端口提供服务; `DNS`服务基于`UDP`,在53端口提供服务。防火墙处于关闭状态。
  
  ```bash
  systemctl start apache2 # port 80
  systemctl start dnsmasq # port 53
  ```
- **过滤状态**：对应端口开启监听, 防火墙开启。
  
  ```bash
  ufw enable && ufw deny 80/tcp
  ufw enable && ufw deny 53/udp
  ```

检查`nmap`：

![check_nmap](img/check_nmap.jpg)

初始状态：

![origin_status](img/origin_status.jpg)



#### TCP connect scan

[tcp_connect_scan——python文件](code/tcp_connect_scan.py)

- Closed(关闭状态)
  
  ![close_status](img/close_status.jpg)
  
  - 攻击机执行代码(要把`.py`文件拖到虚拟机：设备——共享粘贴板——双向；拖放——双向；安装增强功能)：

    ![attacker1](img/attacker1.jpg)
  
  - 靶机抓包（**要先开始抓包，再在攻击机执行代码，否则抓到的包是不全的**）：
  
    ```
    #抓包执行的命令
    sudo tcpdump -i eth0 -enp -w catch_from_eth0.pcap
    ```
  
    用`wireshark`打开抓到的包`catch_from_eth0.pcap`
  
    ![package1](img/package1.jpg)
  
  - `nmap`复刻：
  
    ```
    nmap -sT -p 80 172.16.111.134
    ```
  
    ![nmap1](img/nmap1.jpg)
  
- Open

  ![start](img/start.jpg)

  - 攻击机执行代码：

    ![attacker2](img/attacker2.jpg)

  - 靶机抓包(先开启抓包，再在攻击机执行代码)：

    ![package2](img/package2.jpg)

    用wireshark打开抓到的包：

    ![package2_2](img/package2_2.jpg)

  - `nmap`复刻：

    ![nmap2](img/nmap2.jpg)

- Filtered
  
  ![filter](img/filter.jpg)
  
  - 攻击机执行代码：
  
    ![attacker3](img/attacker3.jpg)
  
  - 靶机抓包(先开启抓包，再在攻击机执行代码)：
  
    ![package3](img/package3.jpg)
  
    用wireshark打开抓到的包：
  
    ![package3_2](img/package3_2.jpg)
  
  - `nmap`复刻：
  
    ![nmap3](img/nmap3.jpg)


#### TCP stealth scan

[tcp_stealth_scan—python文件](code/tcp_stealth_scan.py)

- Closed
  
  ![stealth_scan_close_status](img/stealth_scan_close_status.jpg)
  
  - 攻击机执行代码：

    ![attacker4](img/attacker4.jpg)
  
  - 靶机抓包(先开启抓包，再在攻击机执行代码)：
  
    ![stealth_scan_close_package](img/stealth_scan_close_package.jpg)
  
    用wireshark打开抓到的包：
  
    ![stealth_scan_close_package_wireshark](img/stealth_scan_close_package_wireshark.jpg)
  
  - `nmap`复刻：
  
    ```
    sudo nmap -sS -p 80 172.16.111.134
    ```
  
    ![nmap4](img/nmap4.jpg)
  
- Open

  在靶机执行`systemctl start apache2`

  - 攻击机执行代码(步骤和上面基本上完全一样，不展示图片了)

  - 靶机抓包(先开启抓包，再在攻击机执行代码)

    ![tcp_stealth_scan_open_package](img/tcp_stealth_scan_open_package.jpg)

  - `nmap`复刻：

    ![nmap5](img/nmap5.jpg)

- Filtered
  
  在靶机执行`sudo ufw enable && sudo ufw deny 80/tcp`
  
  - 攻击机执行代码
  
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  
    ![tcp_stealth_scan_filter_package](img/tcp_stealth_scan_filter_package.jpg)
  
  - `nmap`复刻：
  
    ![nmap6](img/nmap6.jpg)



#### TCP Xmas scan

[tcp_xmas_scan——python文件](code/tcp_xmas_scan.py)

- Closed
  
  ![tcp_xmas_scan_close_status](img/tcp_xmas_scan_close_status.jpg)
  
  - 攻击机执行代码：
  
    ![tcp_xmas_scan_closed_attacker](img/tcp_xmas_scan_closed_attacker.jpg)
  
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  
    ![tcp_xmas_scan_close_package](img/tcp_xmas_scan_close_package.jpg)
  
    用wireshark打开抓到的包：
  
    ![tcp_xmas_scan_close_package_wireshark](img/tcp_xmas_scan_close_package_wireshark.jpg)
  
  - `nmap`复刻：
  
    ```
    sudo nmap -sX -p 80 172.16.111.134
    ```
  
    ![nmap7](img/nmap7.jpg)
  
- Open|Filtered

  ![tcp_xmas_scan_open_status](img/tcp_xmas_scan_open_status.jpg)
  
  - 攻击机执行代码
  
    ![tcp_xmas_scan_open_or_filter_attacker](img/tcp_xmas_scan_open_or_filter_attacker.jpg)
  
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  
    ![tcp_xmas_scan_open_or_filter_package](img/tcp_xmas_scan_open_or_filter_package.jpg)
  
    用wireshark打开刚刚抓取到的包：
  
    ![tcp_xmas_scan_open_or_filter_package_wireshark](img/tcp_xmas_scan_open_or_filter_package_wireshark.jpg)
  
  - `nmap`复刻：
  
    ![nmap8](img/nmap8.jpg)



#### TCP fin scan

[tcp_fin_scan——python文件](code/tcp_fin_scan.py)


- Closed
  
  - 攻击机执行代码：
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  - `nmap`复刻：
- Open|Filtered

  - 攻击机执行代码：
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  - `nmap`复刻：



#### TCP null scan

[tcp_null_scan——python文件](code/tcp_null_scan.py)

- Closed
  
  - 攻击机执行代码：
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  - `nmap`复刻：
- Open|Filtered

  - 攻击机执行代码：
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  - `nmap`复刻：



#### UDP scan

[udp_scan——python文件](code/udp_scan.py)


- Closed
  
  - 攻击机执行代码：
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  - `nmap`复刻：
- Open|Filtered

  - 攻击机执行代码：
  - 靶机抓包(先开启抓包，再在攻击机执行代码)
  - `nmap`复刻：



#### 其他实验问题的回答

- 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因。

  答：从上述实验过程可以看出：通过python编程实现的每一次扫描测试的抓包结果与课本中的扫描方法原理完全相符。



### 课后思考题：

- 通过本章网络扫描基本原理的学习，试推测应用程序版本信息的扫描原理，和网络漏洞的扫描原理。
- 网络扫描知识库的构建方法有哪些？



### 遇到的问题和解决办法：

1. 安装ufw时的报错：

   ![error1](img/error1.jpg)

   解决办法：执行`sudo apt install ufw`

2. 有些命令在执行的时候报错`You need to be root to run this script`，解决办法：使用`sudo`。

3. 完成TCP connect scan部分的实验之后要进行TCP stealth scan部分的实验：下一部分实验的第一个状态是closed状态，而靶机在完成TCP connect scan部分的实验之后的状态是**过滤状态**，即对应端口开启监听, 防火墙也开启；这时如果只执行`sudo ufw disable`，只是关闭了防火墙，没有关闭端口监听，(其实就是在open的状态而没有在closed状态)，攻击机执行程序的时候就会出现下图所示的情况：

   ![attacker_error](img/attacker_error.jpg)

   解决办法：执行`systemctl stop apache2` ，关闭端口80。

   ![attacker_correction](img/attacker_correction.jpg)



### 参考连接

- [scapy2.4.4文档](https://scapy.readthedocs.io/en/latest/)
- [B站授课视频](https://www.bilibili.com/video/BV1CL41147vX?p=49)
- [师哥的仓库](https://github.com/CUCCS/2020-ns-public-LyuLumos/tree/ch0x05/ch0x05)