---
layout: post
title: "CTF新生赛'华为杯'WP"
subtitle: '第一次参加比赛, 爆肝几天后艰难老八'
author: "ZolHo"
header-style: text
tags:
  - WriteUp
  - CTF
---
# Writeup

## CRYPTO:

#### 0x00  碰碰车

题目给出一段缺失的字符串和对应的MD5, 求完整的MD5, 字符串只有三个字符未知, 可以直接爆破, 代码如下:
![ss](/img/in-post/huaweibei/01-0.png)

易得符合要求的MD5值

> FLAG值：ctf{a8f738a65b715ea54900b180865b20af}


#### 0x01  EasyRSA

题目给出RSA中的p+q 和p^2+q^2的值, 以及加密指数e和密文c, 求明文m.
由已知可联立解出p和q, 于是搜索python的解方程的库, 可得工具sympy, 遂解出p q, 然后可解出解密因子d和最后的明文m, 代码与运行结果如下:
![1](/img/in-post/huaweibei/02-0.png)

FLAG值：ctf{Rsa_1s_So_Easy!!! }

#### 0x02  block cipher

题目给出一个远程服务, 连接可以看到其提供三个选项:
1. 加密你提供的msg并输出;  
2. 输出加密后的flag;  
3. 查看源码;

研究下它的源码可得知其加密过程为:
1.	将明文填充至16的倍数
2.	每次取16个字组成4*4的矩阵A
3.	用4*4的KEY矩阵乘A得密文矩阵B
4.	将B%256得最终输出的16个byte

首先我以为可以通过求逆矩阵得到KEY的逆矩阵, 然后用来还原明文, 但是因为输出矩阵mod256之后已经改变了矩阵, 所以不行(中间发现pynum的逆矩阵不准, 还特意下了个matlab…)
考虑到单位矩阵乘以KEY就可输出KEY矩阵, 但是0和1没有在ASCII中, 所以不行
由于已知明文和对应密文, 可以考虑构造明文组合来攻击以求得KEY, 突然想到了矩阵其实有乘法分配率, 有K\*(A+E) = K\*A + K, 所以可以用一组字符串A和其对角线位置+1的字符串来求得K, 如果K的数值在0-256之间那么就可以求得秘钥矩阵KEY
求出KEY后验证为真, 但有KEY也没办法直接求出明文(因为密文肯定是被%了若干的256), 所以考虑爆破求明文

根据矩阵乘法规律可知明文的一列仅影响密文对应的那一列, 于是只需爆破4位的字符, 一次一列, 最后16位还有些细节要处理, 这里不展开了, 最后可爆破出明文为flag

> FLAG值：ctf{yes-sure-the-plain-is-blocked-right? }


#### 0x03  Go Home

题目给出了RSA中的n, p的大部分位数以及c, 因为要出门, 于是写了个爆破p的脚本, 果然回来也没跑出p
用n除以残p, 可得q的位数和p相差不多, 于是用可以yafu框架来求解p q相近时候的n
由于没有e, 我爆破了几天e之后觉得不对劲, 然后研究了一下e为3时算出的明文ardyPq2]2lb]A_1q_p]_p1]qm]dsllw{
发现a+2=c, r+2=t, 遂将整串字符+2得flag

> FLAG值：ctf{Rs4_4nd_Ca3sar_ar3_so_funny}


#### 0x04  Go Home

题目给出了RSA中的n, p的大部分位数以及c, 因为要出门, 于是写了个爆破p的脚本, 果然回来也没跑出p
用n除以残p, 可得q的位数和p相差不多, 于是用可以yafu框架来求解p q相近时候的n
由于没有e, 我爆破了几天e之后觉得不对劲, 然后研究了一下e为3时算出的明文
*ardyPq2]2lb]A_1q_p]_p1]qm]dsllw{*
发现a+2=c, r+2=t, 遂将整串字符+2得flag

> FLAG值：ctf{Rs4_4nd_Ca3sar_ar3_so_funny}

---

## MISC:

#### 0x05  真·签到

按照题目所说, 扫描二维码, 回复公众号可得flag

> FLAG值：ctf{Sign_1n_4nd_Enjoy_Yourself!}

#### 0x06  Look_at_your_keyboard

题目给出提示, 看键盘, 每组字母对应键盘位置可形成一个字母, 组合即为flag

> FLAG值：ctf{ctfisfun}

#### 0x07  Buddha

XCTF做过与佛论禅, 这里放过去发现解不出来, 然后看到是新佛曰, 搜索新佛曰解密即可得base编码Y3tXX3p9dFhuYlJmMlJEUw==
Base64解码后可得字符串
c{W_z}tXnbRf2RDS
明显是栅栏编码, 遍历可得每组为3, 结果即为flag

> FLAG值：ctf{X2WnR_bDzRS}

#### 0x08  Do you know Xp0int

题目一张图片, 放入winhex, 搜索ct可得flag

> FLAG值：ctf{We1c0me_to_Join_Us}

#### 0x09  EasyMisc

题目文件没有格式, 放入linux file一下可知为rar压缩包, 遂解压的两张图片gril 和new
尝试若干无效操作如strings和foremost等之后, 观察girl的字节发现有可疑字节ori.png
之后用StegSolve提取出文件的信息, 发现additional bytes部分中有一个rar文件的字节流(Rar开头), 同时附近有字符串(只有小写), 复制进winHex新建文件, 得到加密压缩包, 考虑提示只有小写, 于是使用爆破rar的程序, 得解压密码为三位字符串, 将其解压后可得图片ori.png, 与new.png看上去相同, 之后用StegSolve, python处理无果
后来在查看去年新生赛的时候看到了类似的题目, 可能为盲水印, 但是去年的工具处理依然无果, 但之前接触过其他盲水印的题目, 知道不同的工具处理的结果不同, 遂用其他的盲水印工具尝试, 最后成功提取出带有flag的图像(艰难读取flag)
![ss](/img/in-post/huaweibei/09-0.png)

> FLAG值：ctf{LSGG_TXDY_ddddhm}


#### 0x0A  集齐五龙珠

题目给出大堆名字意义不明的文件, 如图
![ss](/img/in-post/huaweibei/0A-0.png)
不过还是很明显可以看出大概率是base64编码
于是写个脚本遍历文件名字并输出解码后的文件名
翻看一下可以看到有几个XX_ONE的文件, 于是修改下脚本删除其他的文件, 可得五个文件:
first , smart ,cute ,funny, last!
固定first和last, 可从first的字节看出这是个zip文件(PK开头),中间的文件全排列一遍合成6个文件, 其中就可以看到有flag的图片, 代码和flag图如下:  

> FLAG值： ctf{so_mAny_f!les_g0_r3l@x_y0ur_3y3}

#### 0x0B  快乐修补匠

题目给出了一个底下空了一截的图片, 已有的图片上写着password: zygg_txd1
暂且不管这个密码, 查看字节可在文件尾部发现一串base64编码, 于是想到是编码了部分字节
利用python将编码还原后拼接进文件, 没反应
题目有提示流密码, 于是用password去异或编码, 都没反应, 于是去研究了下png的编码, 才发现应该我将格式头IDAT误以为base编码的一部分(这应该是故意误导的), 取IDAT后面开始直到==部分应该为正确的base串
由于异或密文无果, 百度搜流密码, 发现RC4比较常见, 于是搜一个RC4的函数, 以zygg_txd1为秘钥进行解码后再拼接文件, 最终成功修复图片得flag

> FLAG值： ctf{Y0u_mus7_kn0w_png_f0rmAt_well}

---

## WEB:

#### 0x0C  假的签到

题目上来就一个机器人, 于是进robots.txt, 看到路径phpp_tql.php, 访问之
![ss](/img/in-post/huaweibei/0C-0.png)

查看代码可知, 需要一组md5相同的不同串, 直接百度md5相同的字符串可以找到符合要求的串:
https://blog.csdn.net/qq_42967398/article/details/104522626
用hackbar构造url访问即可得到flag
![ss](/img/in-post/huaweibei/0C-1.png)

> FLAG值： ctf{r0bots_1s_g00d}

#### 0x0D  世界上最简单的后门

打开页面直接给出了一句话木马, 尝试用中国蚁剑连接, 有时成功有时不成功, 尝试多次之后连上了控制台, 发现根目录下的flag, cat 之, 得flag

> FLAG值： ctf{gOoo0od}

#### 0x0E  babysql

Web苦手的我, 在尝试了 ’ or ‘1’=’1 成功之后就不知道如何操作了, 使用union select 会被检测出来并返回”你不对劲”, 于是我去搜索自动注入工具和绕过检测的方法, 捣鼓了半天无果
但毕竟这是baby题, 于是我想到了看看攻防世界里有没有, 结果确实找到了之前看过的类似 supersql
照着网上此题的题解, 先进行堆叠注入
![ss](/img/in-post/huaweibei/0E-0.png)

得表flag, 然后查看字段, 可看到类似flag的字段, 尝试提交可知确实是flag

> FLAG值：ctf{enjoy_Sq1i11_qu3ryy}

#### 0x0F  babyssrf

此题截图时候漏了, 题目给了一个可以访问url的输入点, 由题目百度ssrf是啥, 发现可以使用file:// 协议, 于是用hackbar直接构造了个好像是file:///flag ,具体记不清了, 直接得到了flag
FLAG值： 忘了

#### 0x10  Let's play a simple game again

页面上来就说 “GET me Xp0int with 666” 很容易想到是提交个GET, Xp0int=666
进入第二个页面, 故技重施, 于是用hackbar加个post的Xp0int=JoinUs
最后第三个页面, 说你不是管理员, 于是卡在这里了
期间, 研究过cookie, 发现有个base64编码的cookie, 解码得admin=0, 尝试改为1再请求, 无果
后面看去年新生赛发现了类似的题, 用x-forwarded-for: 127.0.0.1来伪造内网访问, 用burpsuit抓包修改, 结合admin=1和xff(我还删了个session不知道有没有影响), 最后得到了flag
![ss](/img/in-post/huaweibei/10-0.png)

> FLAG值： ctf{Have_4_n1ce_c0mpetition!}

----

## OSINT:
#### 0x11  checkin

题目要求收集三个信息:
1. xp0int战队邮箱
2. JNU举办的比赛
3. xp0int战队第一任队长ID

1. 邮箱简单, 在有题解的那个官网就有个链接
Xp0intjnu@gmail.com
2. 之后找的是队长ID, 直接百度无果, 尝试收集战队最早的信息, 推测出创建大约在2016年前, 翻公众号找到最早的消息, 发现早年新生赛和新生群, 搜索发现群现在还在, 于是直接在QQ群搜索xp0int:
![ss](/img/in-post/huaweibei/11-0.png)
我们有理由认为这个群主就是第一任的队长, 于是抄下他头像上的ID去搜索, 找到了其个人主页, 查看自我介绍确实提到了他就是xp0int战队创始人之一
3. 最后找的是JNU举行的比赛, 当时我觉得是大型比赛, 用(暨南大学举办/主办), site:jnu.edu.cn 等方式寻找主办的大型比赛, 结果很不理想, 最后自暴自弃的直接搜索暨南大学举办比赛, 然后遍历去试, 最后试出来是世安杯

> FLAG值： ctf{xp0intjnushianbeigiantbranch}

----

## BLOCKCHAIN:

#### 0x12  Check in

题目给出了一串HASH, 并告诉你这是transaction hash, 于是百度可知, transaction hash指的是指向一次交易的地址, 查资料可知在以太坊ropsten.etherscan.io上可以查找这些信息
于是查找可进入该交易的信息页面, 在底下可以看到一个Input Data, 内容为一串HEX, 以太坊还贴心的提供了View Input As UTF8, 可直接看到flag
![ss](/img/in-post/huaweibei/12-0.png)

> FLAG值：ctf{ZSJJ_YYDS!!!!}

----

## REVERSE:

####0x13  捉迷藏

将ELF拖入IDA, F5可得
![ss](/img/in-post/huaweibei/13-0.png)
可知flag可能和程序没啥关系, 于是查找string, 无果, 尝试strings | grep ‘ct’, 有果:
![ss](/img/in-post/huaweibei/13-1.png)
直接strings输出全部, 找到
![ss](/img/in-post/huaweibei/13-2.png)
直接复制过去发现incorrect, 观察它, 我们有理由认为每一串末尾的H是无用的, 去掉后提交, 正确

> FLAG值：ctf{We1c0me_7o_R3_wor!d}

#### 0x14  ByteCode

打开文件, 根据提示是python的字节码, 尝试直接改后缀名pyc放网站上逆向, 不行
说明还是要自己看看的, 本来准备学习下python字节码, 不过看了题目, 发现很好理解
没一小块都是先将一个我们输入的数从数组中取出来, 与下面的数, 做下下面操作, 再与下下面的数, 做判断是否相等的操作(额, 总之从上到下就是个逆波兰表达式的过程)
总之, 根据他们的运算推出我们应该输入的数, 如下:
![ss](/img/in-post/huaweibei/14-0.png)
可得到flag(图中省略了ct)

> FLAG值：ctf{ zygg_yyds_ddddhm}

#### 0x15  正道的光

IDA打开, F5后开始研究生成的代码, 在main函数下容易看出下图注释处的信息
![ss](/img/in-post/huaweibei/15-0.png)

随后看到它将四次调用check_flag, 每次会取maze数组的一部分, 而哪一部分是靠输入的前面四个数决定的
再进入check_flag函数, 看到判断输入是否为”wasd”的时候, 我才明白这是个走迷宫的reverse (原谅我不知道maze的意思)
之后就很简单了, 通过下面计算位置的函数, 判断出迷宫应该为10*25大小, 我们从HEX view中, 拿到maze数组的数据放入python处理, print之后, 就能看到全部四副地图的样子了:
![ss](/img/in-post/huaweibei/15-1.png)

我们走完迷宫后判断步数和step数组对应, 很容易理解字符串的前四个数字是决定地图顺序的
按照step数组决定前四位之后按顺序用-字符拼接起来, 运行测试可得flag

> FLAG值：ctf{2143-sdsdsdsdsdsdsddwdwdwdwdwdwdw-ddddddddddsssssaaaaaaaaaawwwa-wwwwwddssddddwwdddsssssd-aaaaaaaaasssssddddddddd}

#### 0x16  basic_hash

IDA打开, F5查看生成的代码:
![ss](/img/in-post/huaweibei/16-0.png)
此处易知, 我们要令输入的md5值低于程序处理过后的一个数组
由于我还没学会gdb调试, 并且校园网拨号器阻止了我主机和虚拟机桥接来IDA远程调试, 所以卡了几天, 最后借到了服务器尝试远程调试
能动态调试就很简单了, 我们在执行判断之前打个断点, 然后查看此时我们对比的值为多少就可以得到我们应该生成的md5, 断点处栈如下:
![ss](/img/in-post/huaweibei/16-1.png)
其中的若干0是我们的输入, 下面那一大串就是我们需要的准备比较的md5串, 我们将其放入网上的md5破解可解出输入值应该为zygg
以此类推的第二串输入应该为yyds
通过后输出flag: ctf{input1+input2}

可得flag为 ctf{zyggyyds} (不记得有没有加号, 没办法验证了)
> FLAG值：ctf{zyggyyds}
