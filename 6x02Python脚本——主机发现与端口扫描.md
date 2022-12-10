# Python脚本——主机发现与端口扫描

# 主机发现

``` python
# -*- coding: utf-8 -*-
# Python主机发现与端口扫描工具
from random import randint
from scapy.all import *
from optparse import OptionParser
import time

'''
利用optparse模块生成命令行参数化形式
对用户输入的参数进行接收和批量处理
最后将处理后的IP传入Scan函数
'''
def main():
    #输出帮助信息
    usage = "Usage: %prog -i <ip address>"
    parse = OptionParser(usage=usage)
    #获取网段地址
    parse.add_option("-i", '--ip', type="string", dest="targetIP", help="specify the IP address")
    #实例化用户输入的参数
    options, args = parse.parse_args()
    if '-' in options.targetIP:
        '''
        代码举例：192.168.1.1-120
        #通过“-”进行分隔，把192.168.1.1和120进行分离
        取最后一个数作为range函数的start，然后把120+1作为range函数得stop
        '''
        for i in range(int(options.targetIP.split('-')[0].split('.')[3]), int(options.targetIP.split('-')[1]) + 1):
            Scan(options.targetIP.split('.')[0] + '.' + options.targetIP.split('.')[1] + '.' + options.targetIP.split('.')[2] + '.' +str(i))
    else:
        Scan(options.targetIP)

def Scan(ip):
    try:
        #随机目的端口
        dport = random.randint(1,65535)
        #构造标志位为ACK的数据包
        packet = IP(dst = ip) / TCP(flags = "A", dport = dport)
        response = sr1(packet, timeout = 1.0, verbose = 0)
        if response:
            if int(response[TCP].flags) == 4:
                time.sleep(0.5)
                print(ip + ' ' + "is up")
            else:
                print(ip + ' ' + "is down")
        else:
            print(ip + ' ' + "is down")
    except:
        pass
    
if __name__ == '__main__':
    main()
```

# 端口扫描

``` python
# Python端口扫描
# -*- coding: utf-8 -*-
import sys
import socket
import optparse
import threading
import queue
import nmap.test_nmap

'''
调用nmap库，当扫描到开放端口时，扫描该端口的服务信息
'''
def PortBanner():
    pass

'''
编写一个端口扫描类，继承threading.Thread
该类需要传递3个参数，IP、端口队列、超时时间
通过该类创建多个子线程来加快扫描进度
'''
class PortScanner(threading.Thread):

    #需要传入端口队列、目标IP、探测超时时间
    def __init__(self, portqueue, ip, timeout = 3):
        threading.Thread.__init__(self)
        self._portqueue = portqueue
        self._ip = ip
        self._timeout = timeout

    def run(self):
        #调用nmap，进行端口服务识别
        nm = nmap.PortScanner(nmap_search_path=('nmap', r"D:\SafeArmory\SafeTools\nmap\nmap.exe"))
        while True:
            #判断端口队列是否为空
            if self._portqueue.empty():
                #端口队列为空，说明已扫描完毕，跳出循环
                break
            #从端口队列中取出端口，超时时间为1s
            port = self._portqueue.get(timeout=0.5)
            try:
                s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                s.settimeout(self._timeout)
                result_code = s.connect_ex((self._ip, port))
                #若端口开放，则返回0
                if result_code == 0:
                    #识别开放端口的服务、产品及版本
                    nm.scan(hosts=self._ip, ports=str(port), arguments='-sC -sV')
                    serviceName = nm[self._ip].tcp(port)['name']
                    serviceProduct = nm[self._ip].tcp(port)['product']
                    serviceVersion = nm[self._ip].tcp(port)['version']
                    if serviceName == '':
                        serviceName = 'unkonwn'
                    if serviceProduct == '':
                        serviceProduct = 'unkonwn'
                    if serviceVersion == '':
                        serviceVersion = 'unkown'
                    sys.stdout.write(f"{port}: OPEN —— {serviceName}  {serviceProduct}  {serviceVersion}\n")
            except Exception as e:
                print(e)
            finally:
                s.close()

'''
编写一个函数，根据用户参数来指定目标IP、端口队列的生成以及子线程的生成
同时能支持单个端口的扫描和范围端口的扫描
'''
def StartScan(targetip, port, threadNum):
    #端口列表
    portList = []
    portNumb = port
    #判断是单个端口还是范围端口
    if '-' in port:
        for i in range(int(port.split('-')[0]), int(port.split('-')[1]) + 1):
            portList.append(i)
    else:
        portList.append(int(port))
    #目标IP地址
    ip = targetip
    #线程列表
    threads = []
    #线程数量
    threadNumber = threadNum
    #端口队列
    portQueue = queue.Queue()
    #生成端口，加入端口队列
    for port in portList:
        portQueue.put(port)
    for t in range(threadNumber):
        threads.append(PortScanner(portQueue, ip, timeout=3))
    #启动线程
    for thread in threads:
        thread.start()
    #阻塞线程
    for thread in threads:
        thread.join()

#编写主函数来制定参数的规则
if __name__ == '__main__':
    parser = optparse.OptionParser('Example: python %prog -i 127.0.0.1 -p 80 \n        python %prog -i 127.0.0.1 -p 1-100\n')
    #目标IP参数-i
    parser.add_option('-i', '--ip', dest = 'targetIP', default = '127.0.0.1', type = 'string', help = 'target IP')
    #添加端口参数-p
    parser.add_option('-p', '--port', dest='port', default='80', type='string', help='scan port')
    #线程数量参数-t
    parser.add_option('-t', '--thread', dest='threadNum', default='100', type='int', help='scan thread number')
    (options, args) = parser.parse_args()
    StartScan(options.targetIP, options.port, options.threadNum)
```

