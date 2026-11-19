---
title: "Windows认证过程"
date: "2024-08-12T15:07:00+08:00"
previewImage: "/images/covers/windows-security.svg"
draft: false
categories:
  - "Windows安全"
tags:
  - "Windows"
  - "LM"
  - "NTLM"
  - "Kerberos"
  - "认证"
---
# Windows认证过程

今天超酱问了我个问题“非administrator账户是否能进行pth，以及原因是什么”，这个问题根据我的作战经验是不能进行hash传递的，但是原因是什么嘛，不知道，也同时暴露了一个问题，我从来没有好好的对windows认证过程学习过，写这个文章的目的是对windows认证过程进行一下系统学习，今天是2024.7.17，我希望这个文章是以windows的认证的发展史进行编写，同样也希望这个文章当中可以将我知道的关于windows认证的名词例如LM、NTLM、Kerbors等都讲清楚。

**本文所用的代码均放到了我的个人Github当中WinAuth仓库当中**

## 1. 本地认证

在上世纪80年代，MS-DOS系统当中引入了LM（LAN Manager）认证，最早应用于一些LM软件以及一些网络操作，例如打印机等，它使用了补位加DES的模式进行加密，加密后的密文存储在SAM文件当中。

在上世纪九十年代初，随着NT系统的出现，微软引入了全新的认证系统来取代LM，那就是NTLM（NT Lan Manager），他有着较LM更强更复杂的hash算法，同样他的密文也是存储在SAM当中。

在windows 95 98 的时代，windows会在本地存储明文密码，密码存储在C:\Windows下的\<user>.pwl当中，但是这类系统过于上古，且这种认证方式已经被废弃了，所以不必进行过多描述了。



### 1.1 LM HASH的计算方法

记得在早些年的时候，windows的密码是不能超过14个字符的，这个原因的存在就是因为LM的算法限制。在windows NT时代，为了兼容性，NTLM与LM同事存在，但是在Vista版本以后，LM是默认被禁用的，不过可以通过修改注册表来开启LM HASH

~~~
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa        NoLmHash ---> 0
~~~

使用mimikatz将修改注册表前后的hash来出来对比下，就可以看到LM hash已经存在了

~~~bash
# mimikatz 命令
mimikatz # privilege::debug
mimikatz # lsadump::sam
~~~

![image-20240719160325427](/images/legacy/image-20240719160325427.png)
![image-20240719155843182](/images/legacy/image-20240719155843182.png)

LM算法分为以下五步

1. 密码全部转为大写

2. 密码转为hex字符串，不足14 bytes的部分由0进行填充

3. 将字符串对半分开

4. 使用每个 7 字符补0扩展至8字节，并将其作为密钥，对一个常量值 “KGS!@#$%” 进行 DES加密

5. 将两段加密拼接获得LM HASH的最后值

按照这个算法，写一个golang的程序来计算LM HASH，可以发现跟系统里面的LM HASH一样。

![image-20240719161534227](/images/legacy/image-20240719161534227.png)

### 1.2 LM算法带来的问题

假如一组密码小于7等于七位，那么最后的几位都是由0去补位并进行hash的，这样就会导致7位或7位以下的密码的hash最后几位全是aad3b435b51404ee，这对于密码破解者带来了无限利好，同样，由于LM算法是进行分组加密，那么分组解密也大大减少暴力破解的计算复杂度。

对于密码LM HASH来说，将LM HASH分成两部分，当使用不满14位或者不满7位的密码，密码的强度是呈指数型下降的，假如一个黑客对一个8位的密码进行破解，那么他对后半段密码破解的复杂度只有46，正常情况下满位的一组密码是46^7的复杂度，当每少一位的时候这个幂数就减一。

同理，由于位数限制，且LM HASH是分半计算的，那么一组hash满位的密文最多也就是46^7种可能性，我相信一定是有人拥有完整LM HASH的彩虹表的。

![image-20240719162817820](/images/legacy/image-20240719162817820.png)

----

在写这篇文章的时候，正好碰到朋友打项目，当时将LM的算法记错了，以为是用 “KGS!@#$%” 作为key进行加密，信誓旦旦的去跟朋友讲LM能逆向解出来，但是拿过来以后发现KGS!@#$%是加密的明文，而不是key，若计算彩虹表的话，太耗费时间，我就在想如何将解密的过程的复杂度降低，经过思考过后我想出了如下的解密过程：

- 后7位

  后七位可以进行一些小赌，赌密码不足14位，将后七位的运算从一位慢慢扩展，一位没找到就开始找两位，两位没找到就开始找三位，以此类推，经过测试4位以下运算速度都是蛮快的，五位较慢，但是可接受，五位以上时间成本很高，并不经济

- 前七位

  对于前七位，考虑到自然语言的组成，基本上每一个单词当中都会有元音，中文的韵母、英语的元音、以及日语的五十音，都是有元音组成的，在计算过程当中，可以假设前七位有元音，可以有两位或者一位元音，这样算法复杂度会大大降低。

------



### 1.3 NTLM HASH算法

1. 将密码转为Hex字符串

2. 将转换后的Hex字符串转为unicode字符串，向后补零

3. 将获取到的unicode字符串进行md4加密，获得最后的NTLM HASH

按照算法，计算出了NTLM HASH，与系统SAM文件相同。

![image-20240724145727730](/images/legacy/image-20240724145727730.png)



## 2. 远程认证

在早期的认证当中比如说SMB，他们都使用裸明文在网络上传输，直到LM和NTLM的出现，远程认证便变为了挑战响应机制。

### 挑战/响应

挑战/响应（Challenge/Response）是一种身份验证模式，这个模式是为了不在网络上进行任何形式的密码传输而产生的，他会将密码作为一种key去加密一个无意义的字符串，客户端和服务端对加密后的字符串进行比对，来确定身份权限。

### 2.1 LM的C/R

LM协议是早期由Microsoft与3Com共同开发的，由于其低安全性被抛弃（依我看，挑战响应是没有问题的，问题所在应该是LM HASH的算法），LM的挑战响应算法如下

- 首先计算密码的LM HASH
- 将LM HASH通过补位的方式填充，并分成三组key，每组7个字节
- 将每组key作为DES的key对挑战进行加密，生成三组8字节的密文
- 将三组密文拼接生成响应发送给服务端进行验证

### 2.2 NTLM 的 C/R

#### 2.2.1 NTLM v1

NTLM v1是在LM之后的认证协议，尽管NTLM v1也存在一些安全问题，但它相较于LM的确引入了一些增强的功能和安全性（我觉得功劳是因为NTLM HASH的算法变安全了）。其挑战响应的流程与LM的相同，因为流程相同，其也完全兼容LM系列协议。

#### 2.2.2 NTLM v2

NTLMv2是为了解决NTLM的安全问题所制定的，按照微软官方的算法，v2的响应计算方法如下：

1. 先计算密码的NTLM hash

2. 将unicode的用户名与目标连接起来

3. 将步骤2当中的字符串使用NTLM hash作为key进行HMAC-MD5加密，得到NTLMv2 HASH

4. 将这个数据块与server challenge连接

5. 将第四步得到的字符串再使用第三步得到的NTLMv2 HASH进行HMAC-MD5加密，得到NTProofStr

6. 将NTProofStr与blob进行连接，得到最终的NTLMv2 response

实际上我用代码进行计算不管如何都计算不出包当中的响应，那暂时先不算了，用高中代数的函数形式表示

![image-20240726144316538](/images/legacy/image-20240726144316538.png)

~~~
1. NTLMv2_HASH = hmac_md5(unicode(username+domain),ntlm_hash(password))
2. NTProofStr = hmac_md5(challenge+blob,NTLMv2_HASH)
3. RESPONSE = NTProofStr+blob
其中涉及到五个原始变量，分别为 username、domain、password、challenge、blob
~~~

当使用sniffer抓到NTLMv2的登陆包后，由于你成功拿到了上述计算过程当中的除密码外四个原始变量，但是hmac_md5的算法又不可逆，那么爆破明文密码就需要使用字典对这个计算过程进行正向运算，直到找到计算出的响应与请求包相同，这也就是一些破解工具对NTLMv2离线爆破的原理。

#### 2.2.3 HASH 传递

若拿下一台主机权限以后，能成功的拿到明文密码，下一步我相信大多数的红队运动员都会采用这个明文密码进行其他主机的爆破，但是对于一些版本高的机器来说，系统是不会储存明文密码的，当你拿出了该台机器的NTLM HASH，下一步该如何进行呢？

在上一章节我们讨论了各种挑战响应的实现，那么不难看出在挑战响应过程当中是没有明文密码参与的，实际的参与者其实也是NTLM HASH，那也就意味着在进行同密码横向时，可以跳过计算NTLM的步骤，直接将NTLM HASH应用在挑战响应当中，如果挑战响应通过，那么也就得到了那台主机的权限。

问题说回来，没有管理员权限的机器，但是有HASH，到底能不能进行HASH传递？从上一章节讲的过程来看，仿佛是否存在管理员权限跟是否能进行C/R没有什么必要的联系，那么就进行实验去试一遍。

实践证明，在CS上使用普通权限用户是无法进行传递的，究竟是为什么呢？

![](/images/legacy/image-20240807104518661.png)



在此我做了一个猜想，mimikatz会使用提供到的hash，首先会将hash注入到lsass进程的内存，然后再通过该机器的lsass进程进行登陆，这样就不需要再做一套带有认证的客户端了，因此我使用了一台完全没用过的机器进行测试，先使用mimikatz拉取一次密码，再通过mimikatz向一个机器传递一串hash，然后再次拉取密码。

![image-20240812144332983](/images/legacy/image-20240812144332983.png)

果然，在第二次拉取内存中密码的时候，发现了我进行传递所用的密码。

![image-20240812144428462](/images/legacy/image-20240812144428462.png)

此处实验已经证明了，HASH传递跟权限是没有关系的，只是因为CS用来传递的是mimikatz，而mimikatz会将HASH注入到内存当中，理论上只要能完成服务的建立以及挑战响应的流程，就可以达到HASH传递的效果，切不区分是哪种客户端，Linux、MacOS都是可以完成的。

### 2.3 Kerberos

Kerberos的英文来源于希腊语**Κέρβερος**，是冥王哈迪斯的看门狗，也就是人尽皆知的三头犬，负责看守冥界大门，这个名字也正好象征着Kerberos认证的特征-----需要三方进行认证，Kerberos是目前域环境当中常用的认证体系，在Kerberos出现之前，域内采用的是NTLM认证，NTLM认证不管如何都是双方认证，而Kerberos的出现让认证体系正式的变为三方认证。

![image-20240729150828160](/images/legacy/image-20240729150828160.png)

将Kerberos的认证过程进行分解，分为三部分 客户端与AS（认证服务）& 客户端与 TGS（票据服务）& 客户端与目标服务

-------

在我搭建好环境以后，使用域账号去连接smb，发现认证发的还是NTLM协议，我就很纳闷，不是域内的认证默认Kerberos么，为什么会出现这个问题呢？

![image-20240729174652625](/images/legacy/image-20240729174652625.png)

在经过一番思考以后，发觉一个问题，Kerberos是三方认证，我的环境当中是三台机器，PC作为域机器被远程登录，DC作为票据以及身份认证服务器，自己所用的MAC是远程连接的发起方，我的MAC机器并没有将DNS配置为DC，那么也就是说如果PC不跟MAC讲AS以及TGS的地址，那么我的MAC是不会知道AS/TGS的ip的，经过排查发现MAC识别到登陆的是域账号以后，会去查询这个域的AS/TGS地址，如下图

![image-20240729175229353](/images/legacy/image-20240729175229353.png)

发起方发起了大量的DNS请求，包括各种排列组合去搜寻AS/TGS服务地址，但是没找到，从包的编号来看是先去寻的AS/TGS服务，再进行的NTLM服务，这样也侧面证明了Kerberos的优先级是高于NTLM的。

--------

#### 2.3.1 Client & AS

![image-20240730161742155](/images/legacy/image-20240730161742155.png)

在Client & AS 的谈判阶段，Client会将一个包含客户端ID、票据有效期、预身份验证数据的AS-REQ包发送给AS，在我进行第一次认证的过程中被AS返回了KRB5KDC_ERR_PREAUTH_REQUIRED的错误，这个错误是因为请求包当中未携带PreAuth信息，是否携带PreAuth是客户端决定的，而是否驳回未携带PreAuth的请求包是由AS的策略决定的，当客户端因未携带PreAuth被驳回以后，客户端会重新生成一个新的带PreAuth的AS-REQ。

![image-20240730170422087](/images/legacy/image-20240730170422087.png)

##### PreAuth

PreAuth拥有两种认证类型

- PA-ENC-TIMESTAMP

  这种是较为常见的预认证类型，Client会使用登陆Target所用的密码作为基数算出来的HASH来加密当前的时间戳，并将这个时间戳封入padata当中。

- PA-PK-AS-REQ/PA-PK-AS-REP

  这种模式下Client先生成一个随机数与时间戳，并将他们用Target的私钥进行加密，最后将加密后的数据以及证书发送给AS进行认证

----------

![image-20240731175521610](/images/legacy/image-20240731175521610.png)

--------

在AS收到了AS-REQ并校验以后，会向Client返回一个携带着一个票据（ticket字段，也就是TGT）以及一个enc-part，其中ticket字段会在下一阶段直接填充至请求包当中，TGT中的密文是使用krbtgt的密码HASH进行加密的而与ticket平行的enc-part则是一个使用Client的密码HASH加密的Session，这个Session会用于后续流程的加密当中。

![image-20240731115101632](/images/legacy/image-20240731115101632.png)

至此，Client & AS谈判结束。实际上，用来登陆的用户凭据是否正确，在PreAuth认证的过程中就已经确定，也就是Client & AS谈判过程中就已经完成了账户密码的认证。

#### 2.3.2 Client & TGS

![image-20240730161846191](/images/legacy/image-20240730161846191.png)

在Client & TGS谈判阶段，Client首先会将携带ticket、Authenticator（使用C&AS阶段的Session加密时间戳得出）以及一个请求体发给TGS。

![image-20240731162606122](/images/legacy/image-20240731162606122.png)

请求体当中包含请求类型、目标域、目标服务、Client生成的随机数以及加密类型等信息。

![image-20240731164720889](/images/legacy/image-20240731164720889.png)

当TGS收到了请求以后会对TGT进行解密（使用krbtgt hash）校验，确保TGT是由AS发放的以及Ticket是否在有效期内，并对Authenticator进行解密，确定Client是否合法，一切都验证通过后，会生成一个服务的Session，并通过TGS-REP进行Ticket（ST）与访问Session的发放，而这里的服务Session则是与客户端通信所用的Key。

![image-20240731170953080](/images/legacy/image-20240731170953080.png)

与AS谈判相同，返回包中包含了ticket与enc-part，ticket还是下一阶段的票据，而enc-part中包含了下一阶段谈判的Session。

#### 2.3.3 Client & Target

![image-20240801152921217](/images/legacy/image-20240801152921217.png)

在C&T阶段，Client会拿着TGS发放的ticket与Authenticator（由C&TGS阶段获得的Session进行加密）对目标服务进行请求，目标服务会使用自己的服务密钥对Ticket进行解密（TGS密钥），解密后目标服务会验证Ticket当中的有效期身份信息等信息，并在本地ACL当中进行对比，判断是否有该服务的请求权限，所有检查都通过后，发回SessionSetupResponse，Responsse当中会包含时间戳以及会话密钥等信息，此次的会话密钥是对Request阶段的密钥进行二次确认，以保证消息准确。

#### 2.3.4 简述过程

Kerberos的过程蛮繁杂的，经过梳理后，我将这三阶段大致归结了一下

- C&AS阶段：对用户凭证的审查

- C&TGS阶段：对凭证是否有服务权限进行审查（审查凭证是否有某一个服务比如说cifs的权限）

- C&Target阶段：对凭证是否符合本地具体的ACL进行审查（例如：某用户不在文件夹的用户清单中）

  其中涉及到密码哈希的两步为以下两步，既TGT、ST的生成。

  ~~~
  AS ： TGT = f(x){SessionKey,ClientInfo,targetService,krbtgt_HASH}
  TGS： ST = f(x){TGT,SessionKey,CLientInfo,Service_HASH}
  ~~~




#### 2.3.5 Kerberos涉及的攻击

##### **黄金票据**

在Kerberos的过程当中可以发现，用户的认证只发生在Client-AS谈判阶段，当C&AS请求通过后发放krbtgt加密的ticket，后续过程中就再也没有使用过用户的密码了，既然如此，攻击者若拿到了krbtgt的hash，那么攻击者就可以伪造一个ticket直接从C&TGS开始Kerberos认证流程，由于TGT是用户信息以及krbtgt_HASH进行签发的，所以默认情况下，可以签发出域管理员账户的，之所以叫为黄金票据，也是因为他可以访问任何域内服务。

##### **白银票据**

若攻击者拿到了某一个服务的HASH时，可以完全跳过Client&AS/TGS步骤，因为最后的服务只验证ST票据是否能被自己服务HASH所解密，而不关心TGT是否合法。但是在白银票据攻击当中，攻击者的最高权限也只是对拿到HASH的服务的完全掌控。

##### Kerberos用户遍历

在C&AS阶段写过，在AS预认证的过程当中，会对不同情况有不同的回显

![image-20240731175521610](/images/legacy/image-20240731175521610.png)

根据不同的回显可以确定用户是否存在，不过这样进行用户遍历在日志等级高的域内会产生大量日志，事件类型均为kerberos相关，例如4624、4634、4672等

![image-20240806153112780](/images/legacy/image-20240806153112780.png)
