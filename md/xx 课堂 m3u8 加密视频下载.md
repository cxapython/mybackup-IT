> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/theseventhson/p/14626792.html)

 　　1、各大视频类网站的核心资产和竞争力莫过于视频本身了，所以很多网站想尽一切办法都要保护自己的视频内容不会被爬取和盗用；最常见的保护办法莫过于加密了：服务端把视频数据加密，再把密文发给客户端（这里一般都是浏览器或 app），客户端再根据事先约定好的加密方案和密钥解密数据，得到明文，然后再播放！所以理论上讲：既然客户端能播放，说明客户端一定有明文，那么爬虫肯定也能解密得到明文！那么现在需要解决的两个问题：

*   加密算法是啥？
*   密钥是啥？

 　　本文以 xx 课堂为例，详细展示破解加密算法后下载完整可播放视频的方法；

　　2、先登陆 xx 课堂，按 F12 键，选择 network 分页，然后随便打开一个视频观看，下面马上列举出了浏览器请求的所有接口，这里目测有上百个；要想破解加密视频，需要找到这么几个要素：

*   视频文件的链接，换句话说浏览器是在哪个接口下载视频文件的？
*   加密方法
*   密钥

　　 xx 课堂采用的 m3u8 格式编码视频文件，关于这种格式的详细介绍请见参考 1，这里不在赘述；m3u8 编码方案的一大特点：将一个大的媒体文件进行分片，将该分片文件资源路径记录于 m3u8 文件（即 playlist）内，其中附带一些额外描述（比如该资源的多带宽信息 ···）用于提供给客户端；客户端依据该 m3u8 文件即可获取对应的媒体资源，进行播放。简单点理解：服务器会把一个大文件切分成 N 多个小文件，小文件的路径记录在 playlist 内。浏览器根据播放进度向服务器请求 playlist 的小文件，然后解密小文件得到明文；小文件的后缀名是. ts，所以可以根据这个先选几个 ts 文件看看，如下：

        ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407122041133-2009000678.png)

 　　这个接口（本质上是服务端的一个文件）后缀是 ts，是不是 ts 文件了？先不管这些了，复制这个链接到浏览器，成功下载了 ts 文件，然后尝试用 windows 自带的 media player 打开，结果报错如下：

         ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407105208588-837116642.png)

 　　直接说不支持该文件类型！根据经验判断，肯定被加密了，遂用 010Editor 打开一看，果然文件头信息已经被改地面目全非！（windwos 下各种文件都有头信息，来标注这是哪种类型的文件。比如最常见的 PE 头，还有 jpg 等文件的头部都有标注。）这里没有任何文件头信息，播放器能识别才怪了！到这里已经明确了两点：

*   　视频文件都在 ts
*      ts 被加密了

         ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407105543778-1769311024.png)

 　　现在又面临上面提到的那 3 个问题了：ts 文件列表在哪? 哪中加密方式？密钥是啥？

　　3、（1）m3u8 格式的一个特点：将一个大的媒体文件进行分片，将该分片文件资源路径记录于 m3u8 文件（即 playlist）内，其中附带一些额外描述（比如该资源的多带宽信息 ···）用于提供给客户端；所以现在要先找到 playlist！既然是 m3u8 格式，那么先根据关键词搜一下，看看都有哪些接口。搜索结果如下，一共有 3 个接口有 m3u8 关键词；先看第一个接口：接口名里面居然后 playlist_m3u8，这个是不是传说中的 playlist 了？

       ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407122413748-182025320.png)

 　　马上看看这个接口的返回，如下：返回了 3 种不同的分辨率（清晰度），每种清晰度后面紧跟着一个接口，也是 m3u8；这个 request URL（注意：这里是 header 里面的 URL，不是 response 的 URL）应该就是 m3u8 接口了，这个很重要，后续会根据这个接口下载视频数据；

    ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407122221893-1208433331.png)

     其中一种清晰度的内容：

```
#EXT-X-STREAM-INF:PROGRAM-ID=0,BANDWIDTH=232176,RESOLUTION=474x270
voddrm.token.dWluPTE0NDExNTIxNDY1ODI5Nzc5ODt2b2RfdHlwZT0wO2NpZD0yMjc3MzI7dGVybV9pZD0xMDAyNjg3NTY7ZXh0PTkxNzliODBiZTNjYTIxMzA5OTQ4OGJmYTgzZTVlMjBhNzBkN2M1MDVkNTkzOTlhNWE4OTRlMmQ3YzQ5MGY2YTE5ZWQzMTQ3N2VjNjA0N2U4ZGYxYTdkZTU2NzBlOTJlYTVjMGZkMmFmZDRmMzUxNzEyOTVjMzIwZWVjNDk5NTM4YzM3YWE1MzMyZGQ5NzBhNg==.v.f30739.m3u8?exper=0&sign=382269b9e17d37f914942ef7b14894d9&t=60949ee4&us=1981982863631918869
```

 　　（2）m3u8 的 playlist 找到了，接着的问题是找到加密的方法和密钥了；继续看第二个接口的 response，如下：

      ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407142501227-1061593081.png)

 　　这里从字面看也很明显了：METHOD=AES-128：这是加密方法；接着是一大串的 URI，这是啥了？暂时不知道，先复制 URI 到浏览器，下载下来再说，得到 get_dk 文件，从名字看，貌似是 descypt_key、解密文件？

 　　继续向后看，还能看到 IV，这里就实锤了 AES-CBC 的加密方法了！这里吐槽一下：我也不知道 xx 课堂的开发人员是咋想的，所有 ts 的密钥设置成一样就算了，居然还把 IV 设置成 0，这就有点匪夷所思了........

       ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407142606062-564370831.png)

 　　这里 response 的格式也容易理解：第一行是加密的方法、密钥（需要通过接口去服务器取，不是直接暴露的），第三行是 ts 文件的地址，也就是说：第三行这个 ts 文件解密的信息都放在第一行了！

```
#EXT-X-KEY:METHOD=AES-128,URI="https://xxxx/cgi-bin/xxxx/get_dk?edk=CiDDCouW%2B5LTc3PK5i4SUfJ%2B1tEX%2BJb2GENRUeTo21edBBCO08TAChiaoOvUBCokOTMyNDg4YmItOWZjYS00MzFiLWJiYjItNjFmMDhjYjNlYmM3&fileId=5285890788170463121&keySource=VodBuildInKMS&token=dWluPTE0NDExNTIxNDY1ODI5Nzc5ODt2b2RfdHlwZT0wO2NpZD0yMjc3MzI7dGVybV9pZD0xMDAyNjg3NTY7ZXh0PTkxNzliODBiZTNjYTIxMzA5OTQ4OGJmYTgzZTVlMjBhNzBkN2M1MDVkNTkzOTlhNWE4OTRlMmQ3YzQ5MGY2YTE5ZWQzMTQ3N2VjNjA0N2U4ZGYxYTdkZTU2NzBlOTJlYTVjMGZkMmFmZDRmMzUxNzEyOTVjMzIwZWVjNDk5NTM4YzM3YWE1MzMyZGQ5NzBhNg%3D%3D",IV=0x00000000000000000000000000000000
#EXTINF:10.000000,
v.f30741.ts?start=0&end=351007&type=mpegts&exper=0&sign=382269b9e17d37f914942ef7b14894d9&t=60949ee4&us=1981982863631918869
```

      加密方法和 IV 都有了，密钥了？刚才下载了 get_dk 文件，用 010Editor 打开，发现了一串 16byte 的数字，而 16byte=128bit，刚好是 METHOD=AES-128，所以这个是密钥实锤了！

      ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407142717029-638934615.png)　　

　　这里强调几句：从浏览器接口加载的顺序看，是先请求了这 3 个接口，再请求 ts 文件的。逻辑也很简单：通过请求 m3u8 接口，得到包含所有 ts 文件的 playlist，和解密的方法、密钥，然后请求 ts 文件，再在客户端本地解密后播放！

   （3）m3u8 的接口找到，意味着所有 ts 文件的 playlist 找到了；加密方案和密钥都找到了，接下来开始解密！先把密钥复制成 base64 格式，010Editor 的操作方法如下：Edit->Copy as->Copy as Base64；

       ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407144815728-1489026193.png)

 　　然后用到下面的这个软件（下载地址见文章末尾的参考 3）：把 m3u8 链接（注意是那个 request URL）、密钥的 base64 格式和 IV 输入，点击 go 即可；

       ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407144325279-91252406.png)

 　　下载进度：

       ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407144438316-1047634494.png)

        完成后在 download 目录下能看到完整的 mp4 文件和接口信息：

        ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407145008438-299928552.png)

　　mp4 文件能用播放器正常播放！

 ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407144930194-313484117.png)

 　　这是所有原始的 ts 文件，都是解密后、可以正常播放的：

        ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407145120745-367945706.png)

　　随便找个 ts 文件，用 010Editor 打开，这次能看到文件头信息了：是 FFmpeg 格式的，所以播放器能正常播放！

        ![](https://img2020.cnblogs.com/blog/2052730/202104/2052730-20210407145228481-896586170.png)

参考：

1、https://www.jianshu.com/p/e97f6555a070    m3u8 文件格式详解

2、https://www.bilibili.com/video/BV1Lv411a7CP   快速爬取 xx 课堂

3、https://github.com/nilaoda/N_m3u8DL-CLI   m3u8 下载器