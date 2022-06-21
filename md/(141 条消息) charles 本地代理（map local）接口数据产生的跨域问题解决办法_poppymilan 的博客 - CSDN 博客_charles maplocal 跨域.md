> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/poppymilan/article/details/107468230)

问题描述：Charles 本地代理（Map local）接口数据，发现接口单独访问能成功，但是域名网址访问接口显示跨域

Charles 本地代理（Map local）接口数据，

![](https://img-blog.csdnimg.cn/20200720172034661.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

发现接口单独访问能成功，但是域名网址访问接口显示跨域，具体报错如下：

![](https://img-blog.csdnimg.cn/20200720171945577.png)

各项代理配置都正确，charles [抓包](https://so.csdn.net/so/search?q=%E6%8A%93%E5%8C%85&spm=1001.2101.3001.7020)显示返回结果都没问题

![](https://img-blog.csdnimg.cn/20200720172444553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

 就是在浏览器里面访问的时候显示跨域错误没有返回数据！为什么呢？

经过多方对比成功的请求和代理的请求各项参数，发现跨域是因为相应头部里面缺少允许跨域参数引起的！

代理后失败的响应头：

![](https://img-blog.csdnimg.cn/20200720172635310.png)

没代理的时候成功的响应头：

![](https://img-blog.csdnimg.cn/20200720172715462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

 有两个参数很重要，这是解决接口跨域的关键：

1.  access-control-allow-credentials: true
    
2.  access-control-allow-origin: http://yao.yiyaojd.com:3100
    

（这里多说两句跨域的解决办法：一般跨域 最常用的有两种解决办法，一种是 jsonp, 另一种是 CORS（Cross-Origin-Resource-Sharing），也就是需要设置 withCredentials:ture，这里我的接口是 post 请求，所以 jsonp 不适合我，也不可能为了代理去改变我的接口请求方式）

那么如何在代理的时候加上我想要的响应头，目标明确了之后就可以操作 Charles 行动了。

一、Tools 里面有个 rewrite 的功能，可以帮助我们修改请求信息

![](https://img-blog.csdnimg.cn/20200720174027520.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

二、点击进入，点击 1 允许重写，点击 2 新建一个规则![](https://img-blog.csdnimg.cn/20200720174328675.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

三、点击 3 的 add ，把想要代理的接口填写进去（我看很多文章写的是 *, 我自己试了下，发现 * 会对所有的接口产生影响，导致其他接口也没办法返回正确的数据，所以弃用了 *，写具体的接口名称）

![](https://img-blog.csdnimg.cn/20200720174655827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

四、点击 4，添加我们刚刚说的那两个响应头：

1.  access-control-allow-credentials: true
    
2.  access-control-allow-origin: http://yao.yiyaojd.com:3100
    

![](https://img-blog.csdnimg.cn/20200720174905384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20200720174931496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

ok, 全部添加完成就是这个样子了：

![](https://img-blog.csdnimg.cn/20200720175602571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BvcHB5bWlsYW4=,size_16,color_FFFFFF,t_70)

再次请求页面，成功！不报跨域错误了！