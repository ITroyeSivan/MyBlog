**一、SRC平台**

 

**二、一般流程**

1.子域名收集

OneForAll：

python oneforall.py --target example.com run

python3 oneforall.py --targets ./example.txt run

Layer子域名挖掘机

2.系统指纹探测

Ehole：

EHole_windows_amd64.exe finger -u example.com

3.框架型站点漏洞测试

用Nday进行尝试。

Nday库github有很多。

4.非框架型站点漏洞测试

 

5.端口扫描

Nmap：

sudo nmap -sS -Pn -n --open --min-hostgroup 4 --min-parallelism 1024 --host-timeout 30 -T4 -v -p 1-65535 examples.com

6.c段扫描

fofa：ip=”xx.xx.xx.xx/24”

7.目录扫描

dirsearch：

python3 dirsearch.py -u www.xxx.com -e * -t 2

\8. JS信息收集

findsomething：

 