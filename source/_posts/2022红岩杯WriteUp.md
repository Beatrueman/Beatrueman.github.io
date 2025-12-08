---
title: 2022红岩杯WriteUp
date: {{ date }}
tags: [安全]
categories: [Achievements]
---

# PWN

## 一口一个flag

```
题目：
nc一下下
```

```
idea：
根据提示，nc是linux的命令，所以本题需要在linux环境求解
```

```
step：
打开Linux的终端，nc所给的端口
输入：nc 172.20.14.18 31751
得到答案：
redrock{Ech0eCHo-IWaNtF10g_}
```

![1668940725261](D:/%E4%B8%8B%E8%BD%BD%E6%96%87%E6%A1%A3/%E5%AD%A6%E4%B9%A0%E6%96%87%E6%A1%A3/MobileFile/1668940725261.jpg)

## 猫猫flag

```
题目：
nc二下下
```

```
idea:
本题与上题类似
```

```
step:
nc端口后无回显，继续输入指令
ls后出现flag
然后cat flag
得到答案:
redrock{SEhLlCAT_IWaNtf1a9_}
```

![1668941030223](https://gitee.com/beatrueman/images/raw/master/20251208210408819.jpg)

# Web

## 你是哪里的

```
题目：
https://96a84eec-4f7b-45a6-a5bb-224f1064604e.ctf.redrock.team/
```

```
idea:
进入第一层网页，显示（you must be from https://redrock.team）
没什么破绽，按下F12,F5刷新一下，点进网页
显示（A word in the http message header is wrong,do you know?），按下F12并刷新依然没有什么破绽，于是想到用Burpsuite进行抓包
```

```
step：
打开burpsuite，进行抓包，根据题目的暗示（你来自哪里），联想到来自https://redrock.team
于是添加Referer：https://redrock.team后send
找到答案：
redrock{we1c0me t0 redr0ck ctf}
```

![image-20221120185713998](https://gitee.com/beatrueman/images/raw/master/20251208210408822.png)

## RemindYourHead

```
题目：
https://19ab54b4-79ea-469a-b457-f79ca700a343.ctf.redrock.team/
```

```
idea：
点击网页发现是被黑了的网站，按下F12并刷新，没有什么破绽
于是想到抓包
```

```
step：
用Burpsuite抓包，直接找到答案
得到答案：
redrock{d95c187c-df9d-463f-80e2-8eea94e73595}
```

![image-20221120190433399](https://gitee.com/beatrueman/images/raw/master/20251208210408823.png)

# Misc

## 签到

```
题目：
微信公众号关注 "重邮小帮手", 发送 "红岩杯CTF", 获得 flag
```

```
得到答案：
redrock{020408}
```

![image-20221120190708903](https://gitee.com/beatrueman/images/raw/master/20251208210408824.png)

## 啵啵的魔法药水

```
题目：
魔法世界里的小魔女啵啵找不到她的魔法药水了，快来帮帮她ヾ(′▽‘)ﾉ 进入魔法世界需要一点点魔力呢

° (๑˘ ˘๑) ♥啵啵给你的小tips：用docker run 'bobo' 可以短暂的打开进入魔法世界的大门

【拉取镜像的咒语：docker pull wgyao/redrock-ctf:bobo】
```

```
idea:
根据提示，本题需要用到docker。
```

```
step：
载后在cmd命令行里输入
docker pull wgyao/redrock-ctf:bobo
然后在dockers的Image里，找到wgyao/redrock-ctf后点进，发现第21行出现了答案
然后点击右边Command
得到答案：
redrock{Wit-Sharpening_Potion.}
```

![屏幕截图 2022-11-20 191047](C:\Users\李逸雄\Desktop\屏幕截图%202022-11-20%20191047.png)

![image-20221120191324740](https://gitee.com/beatrueman/images/raw/master/20251208210408826.png)

## 流量审计

```
题目：
点击就送flag
```

```
idea：
下载附件，发现.pcapng类型文件需要用Wireshark打开
```

```
step：
用Wireshark打开，发现有线索
下面有text/plain，然后点进
在最后找到答案：
redrock{yyz_is_god}
```

![image-20221120191839016](https://gitee.com/beatrueman/images/raw/master/20251208210408827.png)

![image-20221120191945595](https://gitee.com/beatrueman/images/raw/master/20251208210408828.png)

# Crypto

## 来自红岩的密文1

```
题目：
听说这是一段一直流传在红岩的密文，每一个解密出这段密文的他/她都会开启属于自己的旅程，你能否成功的解密这段密文？开启属于自己的旅程
```

```
idea：
下载附件，发现是数字，想到ASCII
```

```
step：
对应ASCII转码即可求解
redrock{Welc0me-T0-The-CTF-0f-redrock!!!}
```

## 来自红岩的密文2

```
题目：邪恶的力量妄想夺取密文，危难关头清水拿出了一段晦涩的咒语，只要成功的解密这段咒语就能成功的阻止邪恶的Dog&Cat，并且听说这段咒语是开启新冒险的钥匙之一
```

```
idea：
下载附件，发现是一串emoji表情，在网上搜索密码类型，查到这是base100类型，进行转码，没有得到答案，继续寻找转码，多次尝试找到是base64类型转码，依然没有得到答案，继续寻找，多次尝试找到这是base32转码，最终得到答案。
```

```
step：
多次转码，得到答案
redrock{BaSe100_ba5E64_Base32_u_find_me!}
```

![Screenshot_2022-11-15-14-37-05-840_com.android.br](https://gitee.com/beatrueman/images/raw/master/20251208210408829.jpg)

## 你干嘛

```
题目：
听，各种动物的叫声
```

```
idea：
下载附件，打开是一堆有规律的中文。在网上查找密码类型，发现“Ook”刚好可以和“你干嘛”对应
```

```
step：
用word文档将
“你干嘛”换成“Ook”
“。”换成“.”
“？”（中文）换成“?"（英文）
“！”（中文）换成“!"（英文）
在在线Ook转换工具转换出一堆“奇怪的叫声”
查找密码类型，发现是“兽音译者”密码类型
进行转换，得到答案：
redrock{4re y0u Ok?}
```

![image-20221120193539684](https://gitee.com/beatrueman/images/raw/master/20251208210408830.png)

![image-20221120193745280](https://gitee.com/beatrueman/images/raw/master/20251208210408832.png)

## 可惜我年轻无知

```
题目：
但是我是如此的年轻无知，不曾听到她的心声。 救赎之道，就在其中！ yveypbl{kfu_h_kvhsq_mpfsq_dse_appjhgx_rh
ux_xvy_rpfje_spu_dqyvv}
```

```
idea:
根据格式可知
yveypbl=redrock
在网络上寻找解码，找到一个网站https://quipqiup.com/
可以进行解码，刚好与题目对应
得到答案：
redrock{but i being young and foolish with her would not agree}
```

![image-20221120194443214](https://gitee.com/beatrueman/images/raw/master/20251208210408833.png)

![image-20221120194455674](https://gitee.com/beatrueman/images/raw/master/20251208210408834.png)

## A和B

```
题目：
最终flag字母均为小写
```

```
idea:
下载附件，发现是一堆A和B，并且是最终答案的格式
想到培根密码
对照转换表进行手动转换，得到
r{zeyydyyrzzoyzczykz}
隐约看到有些规律，这串字符里包含了redrock
这样排列
r e d r o c k
{ y y z y z z
z y y z z y }
得到答案：
redrock{yyzyzzzyyzzy}
```

![Cache_-22db552fa8704244.](https://gitee.com/beatrueman/images/raw/master/20251208210408835.jpg)![Cache_-678ef29b17e83bff.](https://gitee.com/beatrueman/images/raw/master/20251208210408836.jpg)

# Reverse

## just_re_it

```
题目：
Try to submit everything like redrock{...}
```

```
idea：
先下载附件，百度搜索，reverse类型题目需要用到IDA
```

```
step:
直接把附件拖入IDA64中，查找字符串，得到答案
redrock{This_is_fake_flag}
```

![image-20221120195645245](https://gitee.com/beatrueman/images/raw/master/20251208210408837.png)

## 水水爱听歌

```
题目：
你能猜中我的心思吗~ Python Message.pyc 就可以运行！
```

```
idea：
下载附件，发现后缀是.pyc，百度搜索，发现这种文件需要反编译，在在线工具中反编译得到后缀为.py的文件，用VS打开。
这是python代码，需要修改些东西。
先把  清水大帅比 修改，改成A，保证可以运行
细看代码，发现if语句与flag的输出有关
修改A=check（flag）为A=True
```

```
观察此句，知道这行代码，如果flag的base64加密和zzz相等，则返回true
所以把zzz解密就可以了
在网上搜索，zzz后的解码类型为URL
进行解码得到She_bid_me_to_take_love_easy_as_the_leaves_grow_on_the_tree
```

![image-20221120200830741](https://gitee.com/beatrueman/images/raw/master/20251208210408838.png)

```
然后运行代码，出现下面的情况
```

![image-20221120201021175](https://gitee.com/beatrueman/images/raw/master/20251208210408839.png)

```
还可以输入东西，于是输入She_bid_me_to_take_love_easy_as_the_leaves_grow_on_the_tree
得到答案：
redrock{044d7a01a972dc5882831e89676220c2dc3a3c142e16379a76a45680137a6b55}
```

![image-20221120201115583](https://gitee.com/beatrueman/images/raw/master/20251208210408840.png)

## 赛博丁真

```
题目：
喜欢我 HeLang 吗？
```

```
idea：
平时网上冲浪，知道本题目和何同学有关
下载附件，发现是补充代码
在网上搜索这段代码是何同学制作键盘时被网友发现的一段错误代码
我尝试把错误代码补全，发现成功得到答案！
redrock{ggg_ding_zhen}
```

![image-20221120201454565](https://gitee.com/beatrueman/images/raw/master/20251208210408841.png)

## jmp_or_nop

```
题目：
下载附件
```

```
idea：
下载附件后发现和反汇编有关，拖入dbg（32）中
```

```
step1:
搜索字符串，找到相关信息，可知最终答案格式为RedRock{}
```

![image-20221121010052260](https://gitee.com/beatrueman/images/raw/master/20251208210408842.png)

```
step2：
观察和答案相关部分的汇编语言，可知有直接输出flag的操作，但打开程序并不显示。所以应该修改汇编指令。
```

![image-20221121110844796](https://gitee.com/beatrueman/images/raw/master/20251208210408843.png)

```
step3：
可以看到这里有cmp进行了判断，再用jae指令判断是否跳跃，将其修改。
以下同理。
```

![image-20221121110348685](https://gitee.com/beatrueman/images/raw/master/20251208210408844.png)

```
step4：
接下来就可以运行了
最终得到答案：
RedRock{jmp_and_nop_is_really_useful}
```

![image-20221121104358090](https://gitee.com/beatrueman/images/raw/master/20251208210408845.png)
