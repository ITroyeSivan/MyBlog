# 前言

AWS 的云服务器全称为 Amazon Elastic Compute Cloud，简称为 Amazon EC2，是首家支持英特尔、AMD 和 Arm 处理器的主要云提供商，既是唯一具有按需 EC2 Mac 实例的云，也是唯一具有 400 Gbps 以太网网络的云。

阿里云的云服务器全称为 Elastic Compute Service，简称为 ECS，提供基于 x86 和 ARM 两大主流计算架构的实例产品，产品序列包含通用计算、异构计算、高性能计算三大类。 

腾讯云云服务器全称为 Cloud Virtual Machine，简称为 CVM，覆盖中国、亚太、欧洲及美洲下的多个地域。在靠近您用户的地域部署应用可获得较低的时延。



# EC2 弹性计算

## EC2 所面临的风险

**1、凭证泄露**

云场景下的凭证泄露可以分成以下几种：

- 控制台账号密码泄露，例如登录控制台的账号密码
- 临时凭证泄露
- 访问密钥泄露，即 AccessKeyId、SecretAccessKey 泄露
- 实例登录凭证泄露，例如 AWS 在创建 EC2 生成的证书文件遭到泄露

对于这类凭证信息的收集，一般可以通过以下几种方法进行收集：

- Github 敏感信息搜索
- 反编译目标 APK、小程序
- 目标网站源代码泄露

**2、元数据**

元数据服务是一种提供查询运行中的实例内元数据的服务，当实例向元数据服务发起请求时，该请求不会通过网络传输，如果获得了目标 EC2 权限或者目标 EC2 存在 SSRF 漏洞，就可以获得到实例的元数据。

在云场景下可以通过元数据进行临时凭证和其他信息的收集，在 AWS 下的元数据地址为：`http://169.254.169.254/latest/meta-data` 或者 `http://instance-data/latest/meta-data`，直接 curl 请求该地址即可。

通过元数据，攻击者除了可以获得 EC2 上的一些属性信息之外，有时还可以获得与该实例绑定角色的临时凭证，并通过该临时凭证获得云服务器的控制台权限，进而横向到其他机器。

通过访问元数据的 `/iam/security-credentials/<rolename>` 路径可以获得目标的临时凭证，进而接管目标服务器控制台账号权限，前提是目标需要配置 IAM 角色才行，不然访问会 404

```shell
curl http://169.254.169.254/latest/meta-data/iam/security-credentials
```

![img](https://wiki.teamssix.com/img/1649996601.png)

通过元数据获得目标的临时凭证后，就可以接管目标账号权限了，这里介绍一些对于 RT 而言价值相对较高的元数据：

```text
mac    实例 MAC 地址
hostname    实例主机名
iam/info    获取角色名称
local-ipv4    实例本地 IP
public-ipv4    实例公网 IP
instance-id    实例 ID
public-hostname    接口的公有 DNS (IPv4)
placement/region    实例的 AWS 区域
public-keys/0/openssh-key    公有密钥
/iam/security-credentials/<rolename>    获取角色的临时凭证
```

**3.账号劫持**

如果云厂商的控制台存在漏洞的话，用户账号也会存在一定的风险。

例如 AWS 的控制台曾经出现过一些 XSS 漏洞，攻击者就可能会使用这些 XSS 漏洞进行账号劫持，从而获得目标云服务器实例的权限。

**4.恶意的镜像**

AWS 在创建实例的时候，用户可以选择使用公共镜像或者自定义镜像，如果这些镜像中有恶意的镜像，那么目标使用该镜像创建实例就会产生风险。

以 CVE-2018-15869 为例，关于该漏洞的解释是：当人们通过 AWS 命令行使用「ec2 describe-images」功能时如果没有指定 --owners 参数，可能会在无意中加载恶意的 Amazon 系统镜像 ( AMI），导致 EC2 被用来挖矿。

对此，在使用 AWS 命令行时应该确保自己使用的是不是最新版的 AWS 命令行，同时确保从可信的来源去获取 Amazon 系统镜像。

**5.其他的初始访问方法**

除了以上云场景下的方法外，还可以通过云服务上的应用程序漏洞、SSH 与 RDP 的弱密码等传统场景下的方法进入目标实例。

ssrf 的常见元数据点

1.packet

https://metadata.packet.net/userdata

2.google

http://metadata.google.internal/computeMetadata/v1beta1/
http://169.254.169.254/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/
http://metadata/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/hostname
http://metadata.google.internal/computeMetadata/v1/instance/id
http://metadata.google.internal/computeMetadata/v1/project/project-id
http://metadata.google.internal/computeMetadata/v1/instance/disks/?recursive=true

3.orcale

http://192.0.0.192/latest/
http://192.0.0.192/latest/user-data/
http://192.0.0.192/latest/meta-data/
http://192.0.0.192/latest/attributes/

4.alibaba

http://100.100.100.200/latest/meta-data/
http://100.100.100.200/latest/meta-data/instance-id
http://100.100.100.200/latest/meta-data/image-id

## 使用用户数据执行命令

**前言**

在创建云服务器时，用户可以通过指定自定义数据，进行配置实例，当云服务器启动时，自定义数据将以文本的方式传递到云服务器中，并执行该文本，而且文本里的命令默认以 root 权限执行。

通过这一功能，攻击者可以修改实例的用户数据并向其中写入待执行的命令，这些代码将会在实例每次启动时自动执行。

**控制台**

在控制台界面可以直接编辑实例的用户数据。

在 AWS 下，修改用户数据需要停止实例，在 AWS 下停止实例会擦除实例存储卷上的所有数据，如果没设置弹性 IP，实例的公有 IP 也会变化，因此停止实例需谨慎。

修改用户数据的位置在：操作-> 实例设置->编辑用户数据

![img](https://wiki.teamssix.com/img/1649998078.png)

这里以执行 touch 命令为例，如果用户数据设置为以下内容，那么实例只有才第一次启动才会运行

```bash
#!/bin/bash
touch /tmp/teamssix.txt
```

如果用户数据设置为以下内容，那么实例只要重启就会运行

```bash
Content-Type: multipart/mixed; boundary="//"
MIME-Version: 1.0

--//
Content-Type: text/cloud-config; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="cloud-config.txt"

#cloud-config
cloud_final_modules:
- [scripts-user, always]

--//
Content-Type: text/x-shellscript; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="userdata.txt"

#!/bin/bash
touch /tmp/teamssix.txt
--//--
```

![img](https://wiki.teamssix.com/img/1649998187.png)

保存用户数据后启动实例，这时进入实例，就可以看到刚才创建的文件了，这说明刚才的命令已经被执行了

![img](https://wiki.teamssix.com/img/1649998204.png)

**命令行**

除了在控制台上操作的方式外，也可以使用 aws cli 操作

```bash
aws ec2 run-instances --image-id ami-abcd1234 --count 1 --instance-type m3.medium \
--key-name my-key-pair --subnet-id subnet-abcd1234 --security-group-ids sg-abcd1234 \
--user-data file://my_script.txt
```

`my_script.txt`就是要执行的脚本

查看实例用户数据

```bash
aws ec2 describe-instance-attribute --instance-id i-1234567890abcdef0 --attribute userData --output text --query "UserData.Value" | base64 --decode
```

## EC2下的权限维持

**1、用户数据**

在上文描述到用户数据的时候，可以很容易发现用户数据可以被用来做权限维持，只需要将要执行的命令改成反弹 Shell 的命令即可。

但是也许目标可能很长时间都不会重启实例，而且用户数据也只有实例停止时才能修改，因此还是传统的权限维持方式会更具有优势些，这样来看使用用户数据进行权限维持显得就有些鸡肋了。

**2、后门镜像**

当攻击者获取到控制台权限后，可以看看目标的 AMI（Amazon 系统镜像），如果可以对其进行修改或者删除、创建的话，RT 就可以将原来的镜像替换成存在后门的镜像。

这样当下次目标用户在选用该镜像创建实例的时候，就会触发我们在镜像中植入的恶意代码了。

**3、创建访问密钥**

如果当前环境可以创建新的访问密钥，则可以在 IAM 中创建访问密钥进行权限维持。

**4、创建辅助账号**

除了以上的权限维持方法，还可以通过在 IAM 中创建高权限子账号的方式进行权限维持，然后通过这个子账号进行后续的持续攻击行为。

**5、其他的权限维持方法**

除了上述方法外，还可以通过在实例中添加隐藏用户、安装远控软件等等传统方法进行权限维持。

## 获取共享快照内的数据

**前言**

如果当前凭证具有 EC2:CreateSnapshot 权限的话，可以通过创建共享快照的方式，然后将自己 aws 控制台下的实例挂载由该快照生成的卷，从而获取到目标 EC2 中的内容。

**公有快照**

这里以公有快照作为示例。

![img](https://wiki.teamssix.com/img/1650001031.png)

这里随便找一个快照，点击创建卷，卷的大小需要大于或等于快照大小，这里创建一个 20G 大小的卷，另外可用区需要和自己的 EC2 保持一致，最后快照 ID 指定刚才随便找的快照

![img](https://wiki.teamssix.com/img/1650001064.png)

将刚创建的卷挂载到自己的实例中

![img](https://wiki.teamssix.com/img/1650001090.png)

登录到自己的实例，查看刚才添加卷的名称

```bash
sudo fdisk -l
```

![img](https://wiki.teamssix.com/img/1650001092.png)

通过大小可以判断出来 /dev/xvdf 是刚才刚才添加的卷，然后将这个卷挂载到实例中，通过 ls 就可以看到这个公有快照中的数据了。

```bash
sudo mkdir /test
sudo mount /dev/xvdf3 /test
sudo ls /test
```

![img](https://wiki.teamssix.com/img/1650001117.png)

**私有快照**

在拿到目标 AWS 控制台权限时，如果无法登录到实例，可以为目标实例打个快照，然后将快照共享给自己。

![img](https://wiki.teamssix.com/img/1650001136.png)

将自己的账号 ID 添加到快照共享后，在自己的 AWS 快照控制台的私有快照处就能看到目标快照了，此时就可以将其挂载到自己的实例，查看里面的数据了。



![img](https://wiki.teamssix.com/img/1650001161.png)

目前也已经有了自动化工具了，CloudCopy 可以通过提供的访问凭证自动实现创建实例快照、自动挂载快照等操作

不过 CloudCopy 默认只会获取实例中的域成员哈希值，也就是说前提目标实例是一个域控，不过将这个工具稍微加以修改，就可以获取到其他的文件了，下面来看下这个工具的使用。

获取 CloudCopy 工具

```bash
git clone https://github.com/Static-Flow/CloudCopy.git
cd CloudCopy
python3 CloudCopy.py
```

如果运行报错，一般是因为第三方库缺失，根据提示安装对应模块即可

接下来，开始进行相应的配置，输入自己的 AWS 账号 ID 及访问凭证以及目标的访问凭证

```bash
manual_cloudcopy
show_options
set attackeraccountid xxx
set attackerAccessKey xxx
set attackerSecretKey xxx
set victimAccessKey xxx
set victimSecretKey xxx
set region ap-northeast-2
```

![img](https://wiki.teamssix.com/img/1650001184.png)

配置好之后，就可以利用 CloudCopy 进行哈希窃取了

```bash
stealDCHashes
```

这时会提示选择实例，直接输入要窃取的实例编号即可

![img](https://wiki.teamssix.com/img/1650001208.png)

等待一段时间，就可以看到已经读到 DC 的哈希了。

![img](https://wiki.teamssix.com/img/1650001226.png)

## EC2 子域名接管

存在这个漏洞的前提是目标的 AWS EC2 没有配置弹性 IP，此时如果目标子域 CNAME 配置了该公有 IPv4 DNS 地址或者 A 记录配置了公有 IPv4 IP 地址，那么当该 EC2 被关机或者销毁时，该实例的公有 IP 也会随之释放，此时这个 IP 就会被分配给新的 EC2 实例，造成子域名接管。

例如下面这个这个域名，可以看到 CNAME 绑定到了 AWS 的 EC2 上，可以看到现在 EC2 IP 地址是 15 开头的。

![img](https://wiki.teamssix.com/img/1650004075.png)

访问下，可以看到是正常访问的

![img](https://wiki.teamssix.com/img/1650004139.png)

此时我们将这个 EC2 实例停止再开机，在控制台可以看到 IP 地址已经变成了 3 开头的了

![img](https://wiki.teamssix.com/img/1650004158.png)

此时，因为 CNAME 记录还是原来的记录，再次访问 ec2.teamssix.com 可以看到已经访问不到了，因为原来的那个 15 开头的 IP 此时已经不是我们的了，这样便造成了域名接管。

![img](https://wiki.teamssix.com/img/1650004183.png)

只是这个域名不是被我们接管的，而是被别人接管了，或许自己的域名会被定向到别人的 Jenkins 上也说不定【手动狗头】

![img](https://wiki.teamssix.com/img/1650004202.png)



![img](https://wiki.teamssix.com/img/1650004221.png)

> 白帽子：你资产里的有个 Jenkins RCE ！！！
>
> 厂家回复：感谢提交，我们正在调查。哦，这个域名的 DNS 记录看起来是个悬挂 DNS 记录，这个 Jenkins 资产不是我们的哦。

那么其实这里问题来了，怎么判断 DNS 记录里的 EC2 IP 是公有 IP 还是弹性 IP 呢？大概有以下几种方法：

1. 证书判断，如果某个子域绑定了 AWS EC2 IP，但是这个网站证书和其他子域名的证书明显不一致，那么可能这个 EC2 就是使用的公有 IP，而且当前的 IP 已经是别人的 IP 了，当然前提是网站使用了 HTTPS
2. 网络空间搜索引擎历史记录，通过对该 IP 的历史搜索记录进行查询，如果该 IP 的历史扫描记录一直在变化，那么可能就是公有 IP
3. 通过谷歌、百度搜索的历史记录去判断，这个原理和上面的第 2 点一样，都是通过有没有变化去判断。

不过其实上面三种方法也没法百分百的确定，所以其实最好的办法就是直接问对方，当前 IP 是不是对方所属，虽然这种做法不太 Hacker，但确实是最有效的办法了。



![img](https://wiki.teamssix.com/img/1650004239.png)

最后如果子域名要绑定 EC2，建议为 EC2 绑定个弹性 IP，这样即使实例重启，IP 也不会变，避免自己的域名绑到了其他人的 EC2 的尴尬场景。

不过 u1s1，这个问题影响不算太大，毕竟攻击者想劫持到这个域名的成本还是蛮高的。

## 利用 AWS CLI 执行命令

当目标实例被 AWS Systems Manager 代理后，除了在控制台可以执行命令外，拿到凭证后使用 AWS 命令行也可以在 EC2 上执行命令。

列出目标实例 ID

```bash
aws ec2 describe-instances --filters "Name=instance-type,Values=t2.micro" --query "Reservations[].Instances[].InstanceId"
```

在对应的实例上执行命令，注意将 instance-ID 改成自己实例的 ID

```bash
aws ssm send-command \
    --instance-ids "instance-ID" \
    --document-name "AWS-RunShellScript" \
    --parameters commands=ifconfig \
    --output text
```

获得执行命令的结果，注意将 $sh-command-id 改成自己的 Command ID

```bash
aws ssm list-command-invocations \
    --command-id $sh-command-id \
    --details
```

![img](https://wiki.teamssix.com/img/1657788829.png)

## 获取Windows实例权限

**前言**

在平时进行云上攻防的时候，偶尔会碰到虽然有 ECS 实例管理权限但无法在实例上执行命令的情况。

一般可能是因为实例没有安装云助手或者云厂商本身就不支持直接下发执行命令等原因，在遇到这种情况时，对于 Windows 实例我们依然有办法获取到实例的权限。

思路也很简单，就是通过为目标实例打快照 —> 创建磁盘 —> 创建实例 —> 挂载磁盘 —> 利用 SAM 等文件获取密码或哈希 —> 使用密码或哈希远程登录获取权限。

下面将以华为云为例，来演示下这个步骤。

**演示**

先看下当前场景，我们在接管控制台后，账号下有一台 Windows 实例，现在我们需要获取它的权限。

![img](https://wiki.teamssix.com/img/1683955264.png)

根据刚才的思路，为这台实例的系统盘创建一个快照

![img](https://wiki.teamssix.com/img/1683955272.png)

通过这个快照创建磁盘

![img](https://wiki.teamssix.com/img/1683955279.png)

接着创建一个实例，在创建实例的时候，要注意可用区的选择，需要选择和磁盘在一个可用区，否则会无法挂载。

![img](https://wiki.teamssix.com/img/1683955286.png)

将刚创建的磁盘挂载到刚创建的实例中

![img](https://wiki.teamssix.com/img/1683955292.png)

远程连接刚创建的实例，可以看到磁盘已经挂载上了，如果没有看到则可以在磁盘管理里找找

![img](https://wiki.teamssix.com/img/1683955299.png)

这时我们就可以通过 SAM、SYSTEM、SECURITY 文件获取哈希了，如果是域控可以通过 NTDS.dit 文件获取整个域的哈希。

这里以使用 impacket-examples-windows 获取哈希为例。

```bash
.\secretsdump.exe -sam SAM -security SECURITY -system SYSTEM LOCAL
```

1

![img](https://wiki.teamssix.com/img/1683955311.png)

可以看到获取到了 3 个用户的 Hash、2 个明文密码，经过尝试，第 2 个明文密码就是 cloudbase-init 用户的密码，因此这里就可以利用这个密码直接远程登录目标实例了。

![img](https://wiki.teamssix.com/img/1683955318.png)

登陆目标实例后，发现权限是管理员权限，至此，就已经获取到目标实例的权限了，上述我们自己创建的快照、磁盘、实例此时就可以删掉了。

**结语**

可以看到，整个的步骤还是有些繁琐的，对于上面的步骤可以结合 Terraform 等工具实现自动化，如果读者感兴趣，可以自己尝试尝试。

对于 AWS，老外也有相应的利用工具，比如 CloudCopy，直接通过脚本获取快照中的 NTLM 哈希，原理都是差不多的，我之前也写过这个工具的利用文章，详见：[wiki.teamssix.com/CloudService/EC2/ec2-shared-snapshot.html(opens new window)](https://wiki.teamssix.com/CloudService/EC2/ec2-shared-snapshot.html)

不同的云可能在一些地方不太一样，但思路都是差不多的，在实战中，还是要多琢磨琢磨。

看到这里，也许有的人可能会有疑惑，那如果是 Linux 怎么办呢？Linux 的话，感觉能做的不多，可能就是翻翻文件、解密 Shadow 之类的了。

最后，如果对上面的内容有什么想法或疑问，欢迎在评论区交流。

# 腾讯云 CVM 攻防实践

## 初始访问

### SSRF与元数据

腾讯云服务器会公开每个实例的内部服务，如果发现云服务器中的SSRF漏洞，可以直接查询主机实例的元数据从而进一步深入利用。
当发现在云服务器上存在的 SSRF 漏洞，可尝试如下请求：

```perl
http://metadata.tencentyun.com/latest/meta-data/  获取 metadata 版本信息。
查询实例元数据。

http://metadata.tencentyun.com/latest/meta-data/placement/region
获取实例物理所在地信息。

http://metadata.tencentyun.com/latest/meta-data/local-ipv4 
获取实例内网 IP。实例存在多张网卡时，返回 eth0 设备的网络地址。

http://metadata.tencentyun.com/latest/meta-data/public-ipv4
获取实例公网 IP。

http://metadata.tencentyun.com/network/interfaces/macs/${mac}/vpc-id
实例网络接口 VPC 网络 ID。

在获取到角色名称后，可以通过以下链接取角色的临时凭证，${role-name} 为CAM 角色的名称：
http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/${role-name}
```

如果查询到存在SSRF漏洞的服务器在云上，则可以尝试读取角色的临时凭证进而接管该角色的身份，要注意的是只有在实例绑定了 CAM 角色后才能获取到临时凭证，如果没有指定CAM 角色的名称，接口会返回404。
相关漏洞案例：
https://hackerone.com/reports/341876

漏洞代码：

```php
<?php 
if (isset($_POST['url'])) {
    $link = $_POST['url'];
    $curlobj = curl_init();
    curl_setopt($curlobj, CURLOPT_POST, 0);
    curl_setopt($curlobj,CURLOPT_URL,$link);
    curl_setopt($curlobj, CURLOPT_RETURNTRANSFER, 1);
    $result=curl_exec($curlobj);
    curl_close($curlobj);

    $filename = './curled/'.rand().'.txt';
    file_put_contents($filename, $result); 
    echo $result;
}
?>
```

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/QQ_1720244453262.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/QQ_1720244578018.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/QQ_1720244606643.png)

若有public-keys/，还可以构造链接进一步获取ssh_rsa:

http://metadata.tencentyun.com/latest/meta-data/public-keys/0/openssh-key/

在获取到⻆⾊名称后，可以通过以下链接取⻆⾊的临时凭证，${role-name} 为 CAM 角⾊的名称：

http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/${rolename}

![img](https://shs3.b.qianxin.com/butian_public/f606276e13359e8a07d71c11e88837512bf7ea5689ba9.jpg)

关于CAM角色

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/QQ_1720245407726.png)

### 凭证泄露

- **控制台登录凭证泄露**

腾讯云控制台支持邮箱、QQ、子用户的密码登录，这类数据较多泄露在可公开访问的网盘文件中，获取到账号密码后可登录控制台对云服务器执行密码重置，删除实例，shell命令执行等操作。

- **API 密钥泄露**

API 密钥是构建腾讯云 API 请求的重要凭证，这类数据较多泄露在GitHub、逆向后的客户端源码、js前端代码中，获取到SecretId和SecretKey后相当于拥有了对应用户的身份权限，如果该账户下有权限管理云服务器，则可以通过检索主机接管账户下的云服务器。

- **其他关键身份凭证信息泄露。**

比如证书、签名、token、远程连接密码、私钥等数据。这类数据较多泄露在某些配置文件中。

### 账户劫持

利用云厂商控制台本身的漏洞对用户账号进行劫持，常见的比如携带token信息的任意URL跳转、XSS、暴力破解等攻击手段进入其他用户的控制台进而获取云服务器权限。

### 镜像投毒

部分操作系统会出现无人维护更新的情况，云服务商会将这些系统的镜像下线或者替换，自定义服务器镜像可以让用户有更多的选择空间以满足业务的需求。攻击者可以制作带有后门的恶意镜像，通过共享镜像进行操作系统投毒，比如预设一些定时任务、劫持正常系统命令等操作。

## 命令执行

### 通过云控制台执行命令

攻击者在初始访问阶段获取到平台登录凭据后，可以利用平台凭据登录云平台，并直接使用云平台提供的Web控制台登录云服务器实例，在成功登录实例后，攻击者可以在实例内部执行命令。

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622115-501213-image.png)

找到自动化助手后创建命令

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622143-794619-image.png)

保存后执行，点击命令名称后显示返回详情：

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622156-250708-image.png)

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622183-29029-image.png)

### 命令行工具 TCCLI 执行命令

sudo pip install tccli

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/QQ_1720246175562.png)

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622224-973866-image.png)

配置API密钥

```
tccli configure
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622248-458063-image.png)

生成json格式示例：

```
tccli tat RunCommand -generate-cli-skeleton
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622261-833779-image.png)

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622351-387328-image.png)

将生成的json数据保存至本地后，设置Content（base64后的命令）和InstanceIds（实例ID）后执行：

```
tcclitat RunCommand --cli-input-json file://D:\learn\1.json
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622375-382784-image.png)

成功执行ping命令：

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622384-263732-image.png)

### 利用云服务器上的服务漏洞

参考传统渗透测试的方法，可通过SSH弱口令、web应用漏洞、中间件漏洞、操作系统主机漏洞等方式获取云服务器权限。

## 权限维持

### 修改启动配置

访问https://console.cloud.tencent.com/autoscaling/config，修改启动配置，在设置主机步骤时可添加shell脚本，可以加入反弹shell脚本或者其他恶意命令。

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622429-284144-image.png)

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622439-445060-image.png)

### 恶意镜像库插入后门

在控制台中进入实例镜像页面，通过制作带有后门镜像对云服务器进行权限维持，后面一旦用户使用恶意镜像创建实例，便会触发后门程序。

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622446-869122-image.png)

### 创建多个API密钥

作为构建腾讯云 API 请求的重要凭证，API密钥的保管至关重要，若攻击者有了控制台权限，可创建多个API密钥以做备用。

### 创建多个子账号

执行以下命令创建一个用户

```
tccli cam AddUser --Name=test124 --Remark=test --ConsoleLogin=1 --UseApi=1 --Password=Test@123456 --NeedResetPassword=0 --PhoneNum=10086 --CountryCode=+86 --Email=123@qq.com
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622857-530030-image.png)

获取用户权限边界,当前显示改用户没有添加任何策略，因此无法操作cvm:

```
tccli cam GetUserPermissionBoundary --TargetUin=100024621854
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622864-834344-image.png)

列出策略列表：

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622874-170101-image.png)

将PolicyId=1的策略即管理员权限绑定给新建的用户

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622885-5028-image.png)

至此成功创建一个新的管理员账户。

没看懂。

## 权限提升

参考传统渗透测试的方法，可通过内核漏洞、应用程序漏洞、主机服务漏洞等方式对云服务器进行提权。

##  防御绕过

### 关闭告警通知

在控制台-主机安全-告警设置处可关闭包括入侵检测、安全漏洞、安全基线、攻击检测、网页防篡改等通知

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622900-462363-image.png)

关闭异常登录告警实例：

```
tccli cwp ModifyWarningSetting --cli-input-json file://D:\learn\1.json
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648622919-634469-image.png)

### 通用密码或者远程登录密钥

运维配置实例时可能是统一用批量脚本进行创建的，获取控制台权限后可以通过读取统一配置的密码或者远程登录密钥攻击其他云服务器。
使用API密钥查询密钥对列表：

```
tccli cvm DescribeKeyPairs
```

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648623017-391222-image.png)

### 元数据窃取

元数据中检索用户数据的接口也常被用来在漏洞案例中证明危害，很多时候我们希望在创建实例时能够自动对加载一些配置，比如添加用户、配置网络，下载某些软件并安装等等，user-data能让用户自定义配置文件和脚本，这些启动脚本可能包括密码、私钥、源代码等，通过如下地址访问腾讯云服务器实例创建时默认加载的脚本：
http://metadata.tencentyun.com/latest/user-data

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-30/1648623033-579325-image.png)

参考案例：https://hackerone.com/reports/53088

# 参考链接

https://wiki.teamssix.com/CloudService/EC2/

https://zone.huoxian.cn/d/1028-cvm

https://forum.butian.net/share/2412