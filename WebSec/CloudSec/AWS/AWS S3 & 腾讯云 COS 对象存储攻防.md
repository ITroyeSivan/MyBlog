# 云服务简介

云服务，顾名思义就是云上的服务，简单的来说就是在云厂商（例如 AWS、阿里云）那里买的服务。

目前国内云厂商有阿里云、腾讯云、华为云、天翼云、Ucloud、金山云等等，国外有亚马逊的 AWS、Google 的 GCP、微软的 Azure 等等。

各个云厂商对云服务的叫法都不统一，这里统一以 AWS 为例。

S3 对象存储`Simple Storage Service`，简单的说就是一个类似网盘的东西，当然跟网盘是有一定区别的。

EC2 即弹性计算服务`Elastic Compute Cloud`，简单的说就是在云上的一台虚拟机。

RDS 云数据库`Relational Database Service`，简单的说就是云上的一个数据库。

IAM 身份和访问管理`Identity and Access Management`，简单的说就是云控制台上的一套身份管理服务，可以用来管理每个子账号的权限。

因为这些原本都放在本地的东西上了云，相应的就会产生对应的安全风险，因此便有了研究的意义。

# 前言

因为AWS的注册很麻烦，还要添加信用卡，所以就不去尝试了，以学习别人的文章为主。

后面用腾讯云的进行实践。

# S3介绍

对象存储（Object-Based Storage），也可以叫做面向对象的存储，现在也有不少厂商直接把它叫做云存储。

说到对象存储就不得不提 Amazon，Amazon S3 (Simple Storage Service) 简单存储服务，是 Amazon 的公开云存储服务，与之对应的协议被称为 S3 协议，目前 S3 协议已经被视为公认的行业标准协议，因此目前国内主流的对象存储厂商基本上都会支持 S3 协议。

在 Amazon S3 标准下中，对象存储中可以有多个桶（Bucket），然后把对象（Object）放在桶里，对象又包含了三个部分：`Key`、`Data` 和`Metadata`。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240702140950.png)

- Key 是指存储桶中的唯一标识符，例如一个 URL 为：https://teamssix.s3.ap-northeast-2.amazonaws.com/flag，这里的 teamssix 是存储桶 Bucket 的名称，/flag 就是 Key
- Data 就很容易理解，就是存储的数据本体
- Metadata 即元数据，可以简单的理解成数据的标签、描述之类的信息，这点不同于传统的文件存储，在传统的文件存储中这类信息是直接封装在文件里的，有了元数据的存在，可以大大的加快对象的排序、分类和查找。

操作使用 Amazon S3 的方式也有很多，主要有以下几种：

- AWS 控制台操作
- AWS 命令行工具操作
- AWS SDK 操作
- REST API 操作，通过 REST API，可以使用 HTTP 请求创建、提取和删除存储桶和对象。

# Bucket公开访问

在 Bucket 的 ACL 处，可以选择允许那些人访问

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240702141420.png)

如果设置为所有人可列出对象，那么只要知道 URL 链接就能访问，对于设置为私有的情况下，则需要有签名信息才能访问，例如 https://teamssix.s3.ap-northeast-2.amazonaws.com/flag?response-content-disposition=xxx&X-Amz-Security-Token=xxx&X-Amz-Algorithm=xxx&X-Amz-Date=xxx&X-Amz-SignedHeaders=xxx&X-Amz-Expires=xxx&X-Amz-Credential=xxx&X-Amz-Signature=xxx

对于敏感文件，建议权限设置为私有，并培养保护签名信息的安全意识。

理论上，如果公开权限文件的名称设置的很复杂，也能在一定程度上保证安全，但不建议这样做，对于敏感文件，设置为私有权限的安全性要更高。

# Bucket爆破

当不知道 Bucket 名称的时候，可以通过爆破获得 Bucket 名称，这有些类似于目录爆破，只不过目录爆破一般通过状态码判断，而这个通过页面的内容判断。

当 Bucket 不存在有两种返回情况，分别是 InvalidBucketName 和 NoSuchBucket。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240702142545.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240702142552.png)

当 Bucket 存在时也会有两种情况，一种是列出 Object

![img](https://wiki.teamssix.com/img/1650005558.png)

另一种是返回 AccessDenied

![img](https://wiki.teamssix.com/img/1650005584.png)

这样通过返回内容的不同，就可以进行 Bucket 名称爆破了，知道 Bucket 名称后，Key 的爆破也就很容易了。

# Bucket Object遍历

注：

1.aws cli命令：https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3api

2.ACL：Amazon S3 访问控制列表（ACL）使您可以管理存储桶和对象的访问权限。每个存储桶和对象都有一个作为子资源而附加的 ACL。它定义了哪些 AWS 账户或组将被授予访问权限以及访问的类型。收到针对某个资源的请求后，Amazon S3 将检查相应的 ACL 以验证请求者是否拥有所需的访问权限。

在 s3 中如果在 Bucket 策略处，设置了 s3:ListBucket 的策略，就会导致 Bucket Object 遍历。

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499283-669847-image.png)

在使用 MinIO 的时候，如果 Bucket 设置为公开，那么打开目标站点默认就会列出 Bucket 里所有的 Key

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499288-336338-image.png)

将 Key 里的值拼接到目标站点后，就能访问该 Bucket 里相应的对象了

# 任意文件上传与覆盖

如果对象存储配置不当，比如公共读写，那么可能就会造成任意文件上传与文件覆盖。

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499307-776657-image.png)

如果目标的对象存储支持 html 解析，那就可以利用任意文件上传进行 XSS 钓鱼、挂暗链、挂黑页、供应链投毒等操作。

# AccessKeyId、SecretAccessKey 泄露

如果目标的 AccessKeyId、SecretAccessKey 泄露，那么就能获取到目标对象存储的所有权限，一般可以通过以下几种方法进行收集：

- Github 敏感信息搜索

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499320-108806-image.png)

- 反编译目标 APK
- 目标网站源代码泄露

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499327-842576-image.png)

# Bucket接管

假如在进行渗透时，发现目标的一个子域显示如下内容

![img](https://wiki.teamssix.com/img/1650005828.png)

通过 cname 记录，可以判断出这是一个 Amazon 的 S3，而且页面显示 NoSuchBucket，说明这个 Bucket 可以接管的，同时 Bucket 的名称在页面中也告诉了我们，为 `test.teamssix.com`

那么我们就直接在 AWS 控制台里创建一个名称为 `test.teamssix.com` 的 Bucket 就可以接管了

![img](https://wiki.teamssix.com/img/1650005891.png)

创建完 Bucket 后，再次访问发现就显示 AccessDenied 了，说明该 Bucket 已经被我们接管了。

![img](https://wiki.teamssix.com/img/1650005911.png)

将该 Bucket 设置为公开，并上传个文件试试

![img](https://wiki.teamssix.com/img/1650005931.png)

在该子域名下访问这个 test.txt 文件

![img](https://wiki.teamssix.com/img/1650005950.png)

可以看到通过接管 Bucket 成功接管了这个子域名的权限

# Bucket ACL可写

列出目标 Bucket 提示被拒绝

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499376-890500-image.png)

查看目标 Bucket ACL 策略发现是可读的，且策略如下

aws s3api get-bucket-acl --bucket teamssix

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499384-341862-image.png)

查询官方文档，内容如下：

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499390-374501-image.png)

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-02-22/1645499395-1106-image.png)

通过官方文档，可以分析出这个策略表示任何人都可以访问、写入当前 Bucket 的 ACL

那么也就是说如果我们把权限修改为 FULL_CONTROL 后，就可以控制这个 Bucket 了，最后修改后的策略如下：

```json
{
    "Owner": {
        "ID": "d24***5"
    },
    "Grants": [
	{
            "Grantee": {
                "Type": "Group", 
                "URI": "http://acs.amazonaws.com/groups/global/AllUsers"
            }, 
            "Permission": "FULL_CONTROL"
        } 
    ]
}
```

```sh
aws s3api put-bucket-acl --bucket teamssix --access-control-policy file://acl.json
```

![img](https://wiki.teamssix.com/img/1650006554.png)

再次尝试，发现就可以列出对象了

![img](https://wiki.teamssix.com/img/1650006615.png)

# Object ACL可写

读取 Object 时提示被禁止

![img](https://wiki.teamssix.com/img/1650006998.png)

查看目标 Object 策略发现是可读的，且内容如下：

```bash
aws s3api get-object-acl --bucket teamssix --key flag
```

![img](https://wiki.teamssix.com/img/1650007025.png)

这个策略和上面的 Bucket ACL 策略一样，表示任何人都可以访问、写入当前 ACL，但是不能读取、写入对象

将权限修改为 FULL_CONTROL 后，Object ACL 策略如下：

```sh
{
    "Owner": {
        "ID": "d24***5"
    },
    "Grants": [
	{
            "Grantee": {
                "Type": "Group", 
                "URI": "http://acs.amazonaws.com/groups/global/AllUsers"
            }, 
            "Permission": "FULL_CONTROL"
        } 
    ]
}
```

将该策略写入后，就可以读取对象了

```sh
aws s3api put-object-acl --bucket teamssix --key flag --access-control-policy file://acl.json
```

![img](https://wiki.teamssix.com/img/1650007059.png)

# Bucket策略可写

##  修改策略获得敏感文件

现有以下 Bucket 策略

![img](https://wiki.teamssix.com/img/1650007548.png)

可以看到根据当前配置，我们可以对 Bucket 策略进行读写，但如果想读取 s3://teamssix/flag 是被禁止的

![img](https://wiki.teamssix.com/img/1650007587.png)

因为当前策略允许我们写入 Bucket 策略，因此可以将策略里原来的 Deny 改为 Allow，这样就能访问到原来无法访问的内容了。

修改后的策略如下：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "*"
                ]
            },
            "Action": [
                "s3:GetBucketPolicy",
                "s3:PutBucketPolicy"
            ],
            "Resource": [
                "arn:aws:s3:::teamssix"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "*"
                ]
            },
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::teamssix/flag"
            ]
        }
    ]
}
```

这里将第 20 行由原来的 Deny 改成了 Allow

![img](https://wiki.teamssix.com/img/1650007686.png)

当策略写入后，可以看到成功获取到了原本 Deny 的内容

![img](https://wiki.teamssix.com/img/1650007708.png)

## 修改网站的S3资源进行钓鱼

例如这样的一个页面

![img](https://wiki.teamssix.com/img/1650007731.png)

查看源代码可以看到引用了 s3 上的资源

![img](https://wiki.teamssix.com/img/1650007750.png)

查看 Bucket 策略，发现该 s3 的 Bucket 是可读可写的

![img](https://wiki.teamssix.com/img/1650007767.png)

这时我们可以修改 Bucket 的静态文件，使用户输入账号密码的时候，将账号密码传到我们的服务器上

![img](https://wiki.teamssix.com/img/1650007791.png)

当用户输入账号密码时，我们的服务器就会收到请求了

![img](https://wiki.teamssix.com/img/1650007813.png)

## 修改 Bucket 策略为 Deny 使业务瘫痪

当策略可写的时候，除了上面的将可原本不可访问的数据设置为可访问从而获得敏感数据外，也可以将原本可访问的资源权限设置为不可访问.

也就是说如果目标网站引用了某个 s3 上的资源文件，而且我们可以对该策略进行读写的话，就可以将原本可访问的资源权限设置为不可访问，这样就会导致网站瘫痪了。

例如这里将策略设置为 Deny

![img](https://wiki.teamssix.com/img/1650007832.png)

当策略 PUT 上去后，网站业务就无法正常使用了

![img](https://wiki.teamssix.com/img/1650007849.png)

# 特定的Bucket策略配置

有些 Bucket 会将策略配置成只允许某些特定条件才允许访问，当我们知道这个策略后，就可以访问该 Bucket 的相关对象了。

例如下面这个策略：

![img](https://wiki.teamssix.com/img/1650007271.png)

```python
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "TeamsSixFlagPolicy",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::teamssix/flag",
            "Condition": {
                "StringLike": {
                    "aws:UserAgent": "TeamsSix"
                }
            }
        }
    ]
}
```

当直接访问 teamssix/flag 的时候会提示 AccessDenied

![img](https://wiki.teamssix.com/img/1650007290.png)

而加上对应的 User-Agent 时，就可以正常访问了



![img](https://wiki.teamssix.com/img/1650007347.png)

在实战中，可以去尝试读取对方的策略，如果对方策略没做读取的限制，也许就能读到。

其次在进行信息收集的时候，可以留意一下对方可能会使用什么策略，然后再去尝试访问看看那些原本是 AccessDenied 的对象是否能够正常访问。

# 使用云函数限制存储桶上传类型

## 前言

相信不少师傅都挖到过存储桶任意文件上传的漏洞，不过由于存储桶的特性，这种任意文件上传的危害性相对传统站点要低一些，但仍然具备一定的风险，比如可以被拿来当做钓鱼或者挂黑页等等。

常规修复这类风险的办法是通过在后端代码里限制文件的上传类型，网上已经有了相应的文章，本文将探讨另外一个方法，即使用云函数去限制存储桶的文件上传类型。

本文将以限制存储桶只能上传图片的场景为例，至于怎么限制其他类型则只需要对函数代码稍加修改就能实现，为了更好的理解本文内容，这里简单绘制了一个流程图如下。

![img](https://wiki.teamssix.com/img/2000000033.jpg)

## 操作步骤

这里以阿里云为例，首先在函数计算 FC 服务里创建一个事件函数，这里函数的区域需要和目标存储桶的区域保持一致。

![img](https://wiki.teamssix.com/img/2000000034.png)

运行环境选择 Python，然后将下面的代码保存为 index.py 文件后压缩成 ZIP 包，再将 ZIP 包上传到云函数中。

```python
import os
import json
import oss2
import imghdr


def handler(event, context):
    # 获取临时访问凭证并进行认证
    creds = context.credentials
    auth = oss2.StsAuth(creds.access_key_id, creds.access_key_secret, creds.security_token)

    # 获取上传文件的相关信息
    evt_lst = json.loads(event)
    evt = evt_lst['events'][0]
    bucket_name = evt['oss']['bucket']['name']
    object_key = evt['oss']['object']['key']
    endpoint = 'oss-' + evt['region'] + '-internal.aliyuncs.com'
    bucket = oss2.Bucket(auth, endpoint, bucket_name)

    # 获取上传文件的后缀类型
    upload_file_suffix_type = os.path.splitext(object_key)[1].replace('.', '')

    # 获取上传文件的内容类型
    head_result = bucket.head_object(object_key)
    upload_file_content_type = head_result.headers.get('Content-Type')

    # 获取上传文件的文件头类型
    upload_file_content = bucket.get_object(object_key).read()
    upload_file_header_type = imghdr.what(None, h=upload_file_content[:32])

    print(f'[+] 文件上传路径：oss://{bucket_name}/{object_key}')
    print(f'[+] 文件头类型：{upload_file_header_type}')
    print(f'[+] 文件后缀类型：{upload_file_suffix_type}')
    print(f'[+] 文件 Content-Type：{upload_file_content_type}')

    # 允许上传的文件类型列表
    allowed_types = ['jpg', 'jpeg', 'png', 'gif']
    allowed_content_types = ['image/jpeg', 'image/png', 'image/gif']

    # 检查文件类型是否合规
    Compliant = 0
    print('[+] 正在检查上传的文件是否合规 ……')
    if upload_file_header_type not in allowed_types:
        print('[-] 文件头类型检查不通过。')
    else:
        print('[+] 文件头类型检查通过。')
        if upload_file_suffix_type not in allowed_types:
            print('[-] 文件后缀类型检查不通过。')
        else:
            print('[+] 文件后缀类型检查通过。')
            if upload_file_content_type not in allowed_content_types:
                print('[-] 文件 Content Type 检查不通过。')
            else:
                print('[+] 文件 Content Type 检查通过。')
                Compliant = 1

    # 删除不允许上传的文件
    if Compliant:
        print(f'[+] 文件 oss://{bucket_name}/{object_key} 检查通过。')
    else:
        print(f'[-] 文件 oss://{bucket_name}/{object_key} 检查不通过，正在删除该文件。')
        bucket.delete_object(object_key)
        print(f'[!] 文件 oss://{bucket_name}/{object_key} 已被删除。')
```

由于我们在这里的 Python 代码中需要调用到 GetObject 和 DeleteObject 的 API，因此还需要为这个函数配置一个至少具备这两个操作权限的角色。

这里我们在 IAM 里创建一个角色，这个角色的最低权限要求如下：

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:GetObject",
        "oss:DeleteObject"
      ],
      "Resource": "acs:oss:oss-<your_bucket_region>:<your_account_id>:<your_bucket_name>/*"
    }
  ]
}
```

信任策略如下：

```json
{
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "fc.aliyuncs.com"
        ]
      }
    }
  ],
  "Version": "1"
}
```

角色创建完成后，在「高级配置」这里选择我们在函数中要使用的角色名称，其他地方可以保持默认或者根据自己的需求进行修改，最后点击函数「创建」按钮。

![img](https://wiki.teamssix.com/img/2000000035.png)

创建完云函数后，我们还需要为这个函数创建一个触发器，从而让存储桶有文件上传时能够触发这个函数的执行。

在函数配置页面中，选择「触发器」，点击「创建触发器」，触发类型选择 OSS，Bucket 名称选择自己需要限制文件上传类型的 Bucket，触发事件这里选择 PutObject，角色直接使用默认角色，最后点击「确定」，这样所有的配置就完成了。

![img](https://wiki.teamssix.com/img/2000000036.png)

此时我们向存储桶上传一个正常的文件，可以看到三项检查都是通过的，这样文件是能正常上传的。

![img](https://wiki.teamssix.com/img/2000000037.png)

如果上传一个后缀是 html 的文件，那么检查就是不通过的，此时云函数就会把这个文件从存储桶中删掉。

![img](https://wiki.teamssix.com/img/2000000038.png)

如果上传的文件名是 png 后缀的，Content-Type 是 image/jpeg，但 Body 部分不是图片类型同样也会无法上传。

![img](https://wiki.teamssix.com/img/2000000039.png)

到此为止，已经能够基本实现使用云函数去限制文件上传类型了，不过这里还有一些可以优化的地方以及一些局限性留给读者探讨。

**可以优化的地方：**

1. 如果想识别其他类型的文件，例如 zip、txt 等格式，这里使用的 imghdr 库就不够用了，imghdr 只能识别出图片文件的类型，这时可以使用第三方库，例如 python-magic 库。
2. 这里代码中采取的策略是检测到不合规的文件会直接执行删除操作，在实际场景中直接删除可能具有一定的风险性，尤其是存在存储桶同名称覆盖问题的时候，因此这里还是需要根据自身业务情况做适当的调整。

**存在局限性的地方：**

由于云函数里的 OSS 触发器只能采取异步的方式执行，因此云函数在触发时，文件就已经被上传到存储桶里了，所以没办法解决在上传同名称文件时被覆盖的问题，这个问题目前似乎只能通过后端代码去解决。

## 小结

本文为限制存储桶文件上传类型提供了一个新的解决方案，但也具备一定的局限性。

此外在当前存储桶策略里我注意到在资源路径下可以使用 *.png 来限制文件的上传后缀，不过目前还没办法限制 Content Type，在此也希望云厂商能够在存储桶策略里加上限制 Content Type 的策略，这样限制存储桶的文件上传类型就更方便了，提升存储桶的安全性也会变得更加简单。

不过如果想要有更严格的限制策略，那么还是云函数或者后端代码具有更高的自由度。

# 腾讯云COS对象存储攻防

## 0x01 Bucket公开访问

创建一个新的存储桶，设置访问权限为私有读写：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240703153304.png)

账户中的访问策略包括用户组策略、用户策略、存储桶访问控制列表（ACL）和存储桶策略（Policy）等不同的策略类型。当腾讯云 COS 收到请求时，首先会确认请求者身份，并验证请求者是否拥有相关权限。验证的过程包括检查用户策略、存储桶访问策略和基于资源的访问控制列表，对请求进行鉴权。
![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-07/1646619904-160188-image.png)

由于我们仅配置了存储桶访问权限为私有读写，所以此时访问存储桶内文件会显示Access Denied。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240703155214.png)

## 0x02 Bucket Object遍历

在腾讯云的访问策略体系中，如果存储桶访问权限为私有读写，且 Policy 权限为匿名访问，那么 Policy 权限的优先级高于存储桶访问权限。
如果控制台配置了Policy权限，默认是对所有用户生效，并且允许所有操作，这时即使存储桶访问权限配置为私有读写，匿名用户也可通过遍历Bucket Object，获取对应的文件。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240703163949.png)

设置好了以后再次尝试

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240703164113.png)

这时，Key值可以理解为文件的目录，通过拼接可获取对应的文件。

## 0x03 Bucket爆破

访问不存在的Bucket时，会显示NoSuchBucket或InvalidRequest。所以只要根据返回内容的不同，就可以写脚本爆破得出存在的Bucket域名。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240703164511.png)

## 0x04 Bucket接管

由于Bucket 接管是由于管理人员未删除指向该服务的DNS记录，攻击者创建同名Bucket进而让受害域名解析所造成的，关键在于攻击者是否可创建同名Bucket，腾讯云有特定的存储桶命名格式，即`<bucketname>-<appid>+cos.ap-nanjing.myqcloud.com：`

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-07/1646619947-13831-image.png)

而appid是在控制台随机生成的，因此无法创建同名Bucket，故不存在Bucket 接管问题：
![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-07/1646619952-217415-image.png)

## 0x05 任意文件上传与覆盖

由于Bucket不支持重复命名，所以当匿名用户拥有写入权限时，可通过任意文件上传对原有文件进行覆盖，通过PUT请求可上传和覆盖任意文件
![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240704172934.png)

## 0x06 用户身份凭证（签名）泄露

通过 RESTful API 对对象存储（Cloud Object Storage，COS）可以发起 HTTP 匿名请求或 HTTP 签名请求。匿名请求一般用于需要公开访问的场景，例如托管静态网站；此外，绝大部分场景都需要通过签名请求完成。
签名请求相比匿名请求，多携带了一个签名值，签名是基于密钥（SecretId/SecretKey）和请求信息加密生成的字符串。SDK 会自动计算签名，您只需要在初始化用户信息时设置好密钥，无需关心签名的计算；对于通过 RESTful API 发起的请求，需要按照签名算法计算签名并添加到请求中。
--摘自官方文档
代表腾讯云用户签名的参数为：SecretId/SecretKey，在开发过程中可能有如下几处操作失误会导致SecretId/SecretKey泄露，获取到SecretId/SecretKey相当于拥有了对应用户的权限，从而操控Bucket

**1.Github中配置文件中泄露凭证**

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-07/1646619970-831294-image.png)

**2.小程序\APP反编译源码中泄露凭证**

**3.错误使用SDK泄露凭证**

常见场景：代码调试时不是从服务器端获取签名字符串，而是从客户端获取硬编码的签名字符串
官方SDK使用文档：https://cloud.tencent.com/document/product/436/8095
![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-07/1646619978-923960-image.png)

**4.第三方组件配置不当导致泄露凭证**

常见场景：/actuator/heapdump堆转储文件泄露SecretId/SecretKey

## 0x07 Bucket ACL可读/写

腾讯云关于ACL的官方文档：https://cloud.tencent.com/document/product/436/30752#.E6.93.8D.E4.BD.9C-permission

为配合接下来的操作，将策略改为或增加acl读写权限。

列出Bucket Object提示无权访问：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240704180918.png)

查看Bucket的ACL配置

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240704192006.png)

根据官方文档，FULL_CONTROL代表匿名用户有完全控制权限：

![img](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-07/1646620006-451246-image.png)

于是在通过PUT ACL写入策略，将存储桶的访问权限配置为公有读写：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240704192708.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240704192810.png)

此时就能看到我们的存储桶访问权限由私有读写变为了公有读写

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240704192847.png)

# 参考链接

https://wiki.teamssix.com/

https://zone.huoxian.cn/d/949-cos