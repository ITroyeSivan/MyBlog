---
title: 基于nonebot2和go-cqhttp的原神机器人实现及开发
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1184384.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-31 10:08:46
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1184913.jpg
---
# 前言
这几天又被朋友拉回去玩原神，打了一周终于打的差不多了，可审计和src搞得少了。但也不能不学习啊！所以就搞了个原神bot。
摸了，但没完全摸。

# 搭建

项目地址：https://github.com/H-K-Y/Genshin_Impact_bot

## python3环境
首先准备好python>=3.7的版本，并确保pip3（pip指向py3也行）使用没有问题，如下所示：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211031102724.png)

如果这一部分出现错误，请优先解决。

## nonebot2安装
1、安装nonebot2及依赖
官方给的教程是这样的：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211031103252.png)

但实际上就2021年10月31日这一天来说，不能这么装。
因为pip装的nonebot似乎不是最新的，我在执行的时候一直报错。
如果你在后面也遇到import的各种报错，可以参考一下我的解决方案：

	pip install nb-cli
	pip uninstall nonebot
	git clone https://github.com/nonebot/nonebot2.git
	cd nonebot2
	pip install .
	cd /usr/local/src/python37/lib/python3.7/site-packages  //根据你python3的目录修改
	ls

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211031104759.png)

如果都装上了，这一步就成功了。如果还会报import的问题，可能是依赖的问题，尝试pip install xx即可。
这里还可能会出现一些问题。比如你执行了这样一条命令：

	pip install nonebot-adapter-cqhttp

它会提示你already satisfied，但是这site-packages下只有nonebot_adapter_cqhttp-2.0.0a16.dist-info而没有nonebot_adapter_cqhttp。于是执行python3 bot.py就会莫名奇妙的报错。

这个问题出现的原因是在之前安装nb-cli时，这个cqhttp适配器被一起装进它的目录了，所以系统找不到（所以这个nb-cli集成在一起真的全是问题，建议分开装）。
解决方法为，手动创建nonebot-adapter-cqhttp目录，把/usr/local/src/python37/lib/python3.7/site-packages/nonebot/adapters/cqhttp下的内容复制进来即可。
别的东西报错也可以用此方法尝试解决。

2、创建nonebot项目

python3 -m nb_cli，然后选择创建新项目。

	[?] Project Name: GS_bot
	注：名字，随便填即可
	[?] Where to store the plugin?
	> In a "genshinimpact_bot" folder
	> In a src folder
	注：选择插件放置目录，键盘↑↓选择，回车确认，这里选第二个In a src folder
	[?] Load Nonebot Builtin Plugin? (y/N)
	注：加载Nonebot2内置插件，回车跳过即可
	[?] which adapter(s) would you like to use?(Use ↑ and ↓ to move，Space to select，Enter to submit)
    >  ● cqhttp
       o ding
       o mirai
       o feishu
	注：上下移动选择框，空格选中，回车确认

3、安装GenshinImpact_Bot

	cd GS_bot # 刚才创建的目录
	git clone -b nonebot2 https://github.com/H-K-Y/Genshin_Impact_bot.git

3、编辑启动文件

	vi bot.py

加入如下代码

	nonebot.load_plugins("Genshin_Impact_bot")

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211031110326.png)

## go-cqhttp安装及配置
1、下载go-cqhttp
这里绝对不能下载教程里那个，因为那个老版本不支持扫码登录，现在tx风控严格，只能扫码才能登进去。

找一个目录（最好和上面GS_bot同级，这样操作起来方便）

	wget https://github.com/Mrs4s/go-cqhttp/releases/download/v1.0.0-beta7-fix2/go-cqhttp_1.0.0-beta7-fix2_linux_amd64.deb
	dpkg -i go-cqhttp_1.0.0-beta7-fix2_linux_amd64.deb
	mkdir qq
	cd qq
	go-cqhttp

输入数字3，也就是反向连接那个，回车。

2、配置go-cqhttp

	vi config.hjson

需要修改的数据

	account: # 账号相关
  	uin: 1233456 # QQ账号
  	password: '密码' # 密码为空时使用扫码登录
	servers:
  	- ws-reverse:
      	universal: ws://127.0.0.1:8080/cqhttp/ws

这里我的缩进有点问题，servers那里请参考

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211031110940.png)

格式有一点不对就会报错。
注：universal下三个有个重新连接的设置，好像默认是3000，不要修改，如果没有就添上。

试一下能不能运行。
我在这里遇到的问题就是time out，一开始以为是反向连接出了问题，后来发现能收到消息，所以不用管他，如果异常退出了再试几次即可。

## 配置nonebot2

找到刚刚nb_cli创建的目录

	vi ~/GS_bot/.env.dev

修改

	HOST=127.0.0.1  # 配置 NoneBot 监听的 IP/主机名
	PORT=8080  # 配置 NoneBot 监听的端口
	SUPERUSERS=["123456"]  # 配置 NoneBot 超级用户
	NICKNAME=["bot", "派蒙"]  # 配置机器人的昵称
	COMMAND_START=["/", ""]  # 配置命令起始字符

NICKNAME填写后，发送xxx 操作等同于直接@机器人（例如抽卡功能需要@机器人，而添加nickname之后可以直接发送派蒙 原之井这样来触发命令

## 运行
1、运行go-cqhttp

	cd ~/qq
	nohup go-cqhttp &

我这里不是第一次运行所以直接挂了后台，如果你是第一次打开请执行

	go-cqhttp

2、运行nonebot2

	cd ~/GS_bot
	nohup python3 bot.py &

至此原神bot已经开始正常运行

测试：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211031111842.png)

妙啊。
命令：https://github.com/H-K-Y/Genshin_Impact_bot/wiki/%E5%91%BD%E4%BB%A4
或输入原神帮助

# 插件开发
这个bot目前功能是很有限的，比如不能查询uid。

由于这次用的是nonebot，所以先看一下它插件的实现：https://v2.nonebot.dev/guide/loading-a-plugin.html

https://v2.nonebot.dev/api/rule.html#startswith-msg

