---
layout: post
title: svn安装过程中出现的问题
date: 2017-09-12 13:32:20 +0300
description: svn安装过程中出现的问题 # Add post description (optional)
img: i-rest.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [svr, install]
---

## 1.无法通过https访问
访问url为: http://ip:/repos
{% highlight bash %}
<Location /repos>
   DAV svn
   SVNParentPath /var/svn
   SVNListParentPath on
   SSLRequireSSL
      AuthType Basic
      AuthName "Authorization Realm"
      AuthUserFile /var/svn/passwd.conf
      Require valid-user
</Location>
{% endhighlight %}

## 2.svn: Server sent unexpected return value (405 Method Not Allowed) in response to OPTIONS request for xxx
访问url svn checkout https://ip/repos/project

## 3.证书问题
[证书配置](https://www.cnblogs.com/wudonghang/p/5167142.html)