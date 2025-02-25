> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-279740.htm)

> [原创] 微信视频号视频加密逆向

拿 WxIsaac64(Isaac64 的变种?) 生成 2^17 个字节。然后和视频的前 2^17 字节做异或。

略去, 总之就是 WeChatVideoDownloader 不能用了。

原文链接: [here](https://bbs.kanxue.com/elink@899K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2S2P5h3&6S2K9$3g2&6j5g2)9J5k6h3y4G2L8g2)9J5c8X3q4J5N6r3W2U0L8r3g2K6i4K6u0r3j5%4c8X3i4K6u0r3N6$3g2U0K9r3q4@1i4K6u0V1N6X3W2V1k6h3!0Q4x3X3c8W2L8X3y4J5P5i4m8@1K9h3!0F1i4K6u0V1M7X3g2$3k6i4u0K6k6g2)9J5k6r3g2F1k6$3W2F1k6h3g2J5i4K6u0r3)

在正式开始逆向之前，我们首先需要能够在微信视频号中打开开发者工具，由于微信默认肯定是不会启用的，所以我们要对微信的某个动态链接库进行小小的修改。

总之就是找到`xweb-enable-inspect`这个启动选项，修改 branch 指令，这个启动选项所在的分支变成永远执行就行了。

最后实现效果如图下

![](https://bbs.kanxue.com/upload/attach/202312/967169_XY8E8PR23RKQ26N.webp)

首先随便打开一个视频，我们可以看到很多请求。其中带有`stodownload`的就是下载的视频文件，但这些视频链接下载下来的内容是加密的。

![](https://bbs.kanxue.com/upload/attach/202312/967169_4QG7BTD5WW9APSD.webp)

先看一下加密前的视频文件头，我们可以明显发现，它的文件头格式并不正确。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p></td><td><p><code>aynakeya @ ThinkStation]:~</code><code>/</code><code>workspace</code><code>/</code><code>weixinshipin</code></p><p><code>01</code><code>:</code><code>59</code><code>:</code><code>21</code> <code>$ xxd </code><code>-</code><code>l </code><code>32</code> <code>v2.</code><code>bin</code></p><p><code>00000000</code><code>: </code><code>75a2</code> <code>b80f </code><code>5db2</code> <code>528b</code> <code>af76 c5f0 </code><code>9407</code> <code>a7e9&nbsp; u...].R..v......</code></p><p><code>00000010</code><code>: </code><code>4c31</code> <code>99a8</code> <code>60ef</code> <code>a5de c64e ce1e </code><code>3ab1</code> <code>6e74</code>&nbsp; <code>L1..`....N..:.nt</code></p></td></tr></tbody></table>

对比之下，一个正常的 mp4 文件的文件头应该如下所示：

<table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>00000000</code><code>: </code><code>0000</code> <code>0020</code> <code>6674</code> <code>7970</code> <code>6973</code> <code>6f6d</code> <code>0000</code> <code>0200</code>&nbsp; <code>... ftypisom....</code></p><p><code>00000010</code><code>: </code><code>6973</code> <code>6f6d</code> <code>6973</code> <code>6f32</code> <code>6176</code> <code>6331</code> <code>6d70</code> <code>3431</code>&nbsp; <code>isomiso2avc1mp41</code></p></td></tr></tbody></table>

那么确认了文件被加密。那么我们要从哪里开始呢。因为解密必然是文件下载完成后才解密的。所以解密的函数或者过程很有可能就在文件下载完成后。

查看请求是从哪行代码发起的，我们可以追踪到`worker_release.js`中的`g.send()`

![](https://bbs.kanxue.com/upload/attach/202312/967169_WCH7EM65C6Y86JX.webp)

这个时候，写过 Javascript XMLRequest 的人可能就很熟悉这个了，在完成所有 callback 设置之后，发送请求用的就是`.send()`，所以往上翻，我们可以找到如下的返回值处理。

![](https://bbs.kanxue.com/upload/attach/202312/967169_94ZXWJF4ZSAJ9J6.webp)

这里我们可以发现解密函数就是函数 M，参数分别为数据和 startIndex(也就是文件的第几个 byte)

函数 M 非常的简单易懂，把数据和 decryptor_array 进行异或即可。如果当前的 startIdx 大于 decryptor_array 的长度，则不进行异或，不改变原有数据。

![](https://bbs.kanxue.com/upload/attach/202312/967169_2JP3R5VZH2JAD7V.webp)

如果我们在这个函数 M 的地方打个断点，我们可以发现这个`decryptor_array`的长度实际上是一个常量 2^17 = 131072 (一直都是这个长度)

![](https://bbs.kanxue.com/upload/attach/202312/967169_V45YR26JDJRWQVD.webp)

从这里我们可以推断出，`decryptor_array`的长度是有限的。

我们从`decryptor_array`的恒定长度可以推断出，视频加密只作用于文件的前 131072 字节。这样的加密策略似乎合理——如果需要对整个视频数据进行加密和解密，那么播放视频时消耗的资源可能会显著增加。

(虽然 DRM 好像就是全文加密的，我也不太了解就是了)

另外，我们还发现，对于同一视频，`decryptor_array`是一致的。不同的视频文件则对应不同的`decryptor_array`。这表明`decryptor_array`是通过某种特定的方法生成或获取的。

经过搜索，我们了解到`decryptor_array`的赋值仅在`wasm_isaac_generate`函数中进行。

![](https://bbs.kanxue.com/upload/attach/202312/967169_8PTMUMJS32RR3FJ.webp)

而`wasm_isaac_generate`函数在代码中只被一个地方调用，即`wasm_video_decode.js`。

在`wasm_video_decode.js`中，`wasm_isaac_generate`作为一个汇编函数，可以在 WebAssembly 中通过`_emscripten_asm_const_int`接口被调用。

![](https://bbs.kanxue.com/upload/attach/202312/967169_DR76TTR8HA6TAXU.webp)

![](https://bbs.kanxue.com/upload/attach/202312/967169_UCDWNQKFR45QCTD.webp)

那么接下来，就要开始逆向可爱的的 wasm 了

下载`wasm_video_decode.wasm`后，我们使用 [wabt](https://bbs.kanxue.com/elink@9b0K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6i4k6h3u0m8M7%4y4W2L8h3u0D9P5g2)9J5c8Y4N6S2j5Y4b7%60.) 工具将其转换为`.o`文件，以便在反编译软件中进行分析。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p></td><td><p><code>.</code><code>/path/to/wasm2c</code> <code>wasm_video_decode.wasm -o wasm_video_decode.c</code></p><p><code>cp</code> <code>/path/to/wasm-rt-impl</code><code>.c .</code></p><p><code>cp</code> <code>/path/to/wasm-rt-impl</code><code>.h .</code></p><p><code>cp</code> <code>/path/to/wasm-rt</code><code>.h .</code></p><p><code>gcc -c wasm_video_decode.c -o wasm_video_decode.o</code></p></td></tr></tbody></table>

完成这些步骤后，我们得到一个二进制文件`wasm_video_decode.o`。将此文件拖入反编译软件，搜索`_emscripten_asm_const_int`的调用。我们发现`wasm_isaac_generate`在函数`f378`处被调用。

![](https://bbs.kanxue.com/upload/attach/202312/967169_UQF2BX7Y8JZW9WW.webp)

进一步通过断点和调用栈的检查，我们发现`worker_release.js`中的`decryptor.generate()`最终触发了`wasm_isaac_generate`的调用。

仔细分析揭示出 decryptor 也是 WebAssembly 环境中的一个对象，即`WxIsaac64`。

![](https://bbs.kanxue.com/upload/attach/202312/967169_Z4XUMCWNRCS8RPG.webp)

经过研究，我们了解到 Isaac64 实际上是一个随机数生成算法。

![](https://bbs.kanxue.com/upload/attach/202312/967169_C5SPKVQWM5ZXHUP.webp)

因此，我们可以合理推测：

1.  decryptor 使用视频对应的 seed 进行初始化。
2.  JavaScript 调用`decryptor.generate()`，指示 wasm 在其内存中生成 2^17 即 131072 个随机数。
3.  wasm 生成随机数后，通过`wasm_isaac_generate`将这些随机数写回 JavaScript，赋值给`decryptor_array`。

现在，我们知道了`decryptor_array`的来源，接下来的问题是确定初始化 Isaac64 算法的 seed 的来源。

接下来就是不停的打断点，看 call stack， 直到找到`seed`最早出现的地方就行了。

![](https://bbs.kanxue.com/upload/attach/202312/967169_P4SQFZC6D9RXJAT.webp)

![](https://bbs.kanxue.com/upload/attach/202312/967169_XKRVK44WHNU7RXZ.webp)

![](https://bbs.kanxue.com/upload/attach/202312/967169_QN37DA2262UHDYB.webp)

简单来说呢就是顺序就是从`FinderGetCommentDetail(objectid)`->`objectDesc.media.decodeKey-`>`seed`。

那么`FinderGetCommentDetail`又是通过什么获取到信息的呢。继续追踪调用。可以发现`FinderGetCommentDetail`最后使用了`window.WeixinJSBridge.invoke`来获取数据。

![](https://bbs.kanxue.com/upload/attach/202312/967169_RVNHKNXQACWMA63.webp)

`window.WeixinJSBridge` ？？？那接下來就要逆向微信的通信协议了。这边有点不想做。所以就没做。

立刻启动后备隐藏能源，发动注入模式

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p></td><td><p><code>(</code><code>function</code> <code>() {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>function</code> <code>wrapper(name,origin) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`injecting ${name}`);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>function</code><code>() {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let cmdName = arguments[0];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`${name}(</code><code>"${cmdName}"</code><code>, ...) =&gt; args: `);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(arguments[1])</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arguments.length == 3) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let original_callback = arguments[2];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>arguments[2] = async </code><code>function</code> <code>() {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`${name}(</code><code>"${cmdName}"</code><code>, ...) =&gt; callback result (length: ${arguments.length}):`);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arguments.length == 1) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(arguments[0]);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>else</code> <code>{</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(arguments);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>await original_callback.apply(</code><code>this</code><code>, arguments);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let result = origin.apply(</code><code>this</code><code>,arguments);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>result;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>window.WeixinJSBridge.invoke = wrapper(</code><code>"WeixinJSBridge.invoke"</code><code>,window.WeixinJSBridge.invoke);</code></p><p><code>})()</code></p></td></tr></tbody></table>

总之结果很好，获得了需要的所有数据

![](https://bbs.kanxue.com/upload/attach/202312/967169_AQD274QZ9ZWAYR2.webp)

![](https://bbs.kanxue.com/upload/attach/202312/967169_HN7UW2Q5T92NRN6.webp)

1.  通过`FinderGetCommentDetail`获取到视频的`decode_key`(就是`seed`)，`url`，`title`等信息
2.  通过 seed 生成`decryptor_array`
3.  通过 url 下载加密后的视频文件，把视频的加密段数据和`decryptor_array`做异或运算即可。

总之为了让 WechatVideoDownloader 能够解密，我思考了以下的思路

由于获取`seed`需要逆向微信协议，我不想在逆向这个协议上花费太多时间。

既然 WechatVideoDownloader 已经使用代理获取视频地址，我们可以进一步使用中间人攻击来获取视频链接及对应的`decode_key`。

只需将注入`WeixinJSBridge.invoke`的代码插入到某个 JS 文件中，当微信客户端请求视频链接时，就把获取到的视频链接发送到本地服务器。

这样不仅解决了 seed 和链接的问题，连视频标题也能获取到。

最后，下载完视频后，通过 seed 生成解密序列并对视频进行解密。

最后的劫持代码

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p><p>29</p><p>30</p><p>31</p><p>32</p><p>33</p><p>34</p><p>35</p><p>36</p><p>37</p><p>38</p><p>39</p><p>40</p><p>41</p><p>42</p><p>43</p><p>44</p><p>45</p><p>46</p><p>47</p><p>48</p><p>49</p><p>50</p><p>51</p><p>52</p><p>53</p></td><td><p><code>(function () {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(window.wvds !</code><code>=</code><code>=</code> <code>undefined) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let receiver_url </code><code>=</code> <code>"https://wechat.video.download"</code><code>;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>function send_response_if_is_video(response) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(response </code><code>=</code><code>=</code> <code>undefined) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code><code>;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(response[</code><code>"err_msg"</code><code>] !</code><code>=</code> <code>"H5ExtTransfer:ok"</code><code>) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code><code>;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let value </code><code>=</code> <code>JSON.parse(response[</code><code>"jsapi_resp"</code><code>][</code><code>"resp_json"</code><code>]);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(value[</code><code>"object"</code><code>] </code><code>=</code><code>=</code> <code>undefined || value[</code><code>"object"</code><code>][</code><code>"object_desc"</code><code>] </code><code>=</code><code>=</code> <code>undefined&nbsp; || value[</code><code>"object"</code><code>][</code><code>"object_desc"</code><code>][</code><code>"media"</code><code>].length </code><code>=</code><code>=</code> <code>0</code><code>) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let media </code><code>=</code> <code>value[</code><code>"object"</code><code>][</code><code>"object_desc"</code><code>][</code><code>"media"</code><code>][</code><code>0</code><code>];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let video_data </code><code>=</code> <code>{</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"decode_key"</code><code>: media[</code><code>"decode_key"</code><code>],</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"url"</code><code>: media[</code><code>"url"</code><code>]</code><code>+</code><code>media[</code><code>"url_token"</code><code>],</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"size"</code><code>: media[</code><code>"file_size"</code><code>],</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"description"</code><code>:&nbsp; value[</code><code>"object"</code><code>][</code><code>"object_desc"</code><code>][</code><code>"description"</code><code>].trim(),</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"uploader"</code><code>: value[</code><code>"object"</code><code>][</code><code>"nickname"</code><code>]</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>fetch(receiver_url, {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>method: </code><code>'POST'</code><code>,</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>mode: </code><code>'no-cors'</code><code>,</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>body: JSON.stringify(video_data),</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}).then((resp) </code><code>=</code><code>&gt; {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`video data </code><code>for</code> <code>${video_data[</code><code>"description"</code><code>]} sent`);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>});</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>function wrapper(name,origin) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`injecting ${name}`);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>function() {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let cmdName </code><code>=</code> <code>arguments[</code><code>0</code><code>];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arguments.length </code><code>=</code><code>=</code> <code>3</code><code>) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let original_callback </code><code>=</code> <code>arguments[</code><code>2</code><code>];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>arguments[</code><code>2</code><code>] </code><code>=</code> <code>async function () {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arguments.length </code><code>=</code><code>=</code> <code>1</code><code>) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>send_response_if_is_video(arguments[</code><code>0</code><code>]);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>await original_callback.</code><code>apply</code><code>(this, arguments);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let result </code><code>=</code> <code>origin.</code><code>apply</code><code>(this,arguments);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>result;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code> <code>Invoke WechatVideoDownloader Service </code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>-</code><code>`);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>window.WeixinJSBridge.invoke </code><code>=</code> <code>wrapper(</code><code>"WeixinJSBridge.invoke"</code><code>,window.WeixinJSBridge.invoke);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>window.wvds </code><code>=</code> <code>true;</code></p><p><code>})()</code></p></td></tr></tbody></table>

proof of work  
![](https://bbs.kanxue.com/upload/attach/202312/967169_4GAHQURB27MUVGJ.webp)

CHATGPT 生成的。

1.  微信 v3.9.8.15
2.  [wasm_video_decode.wasm v1.2.46](https://bbs.kanxue.com/elink@ad7K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8r3q4V1K9h3&6Q4x3X3g2%4P5s2q4U0L8r3!0#2k6q4)9J5k6i4q4I4i4K6u0W2j5$3!0E0i4K6u0r3j5h3I4S2k6r3W2F1i4K6u0r3k6X3k6E0k6i4m8W2k6#2)9J5c8Y4k6A6k6r3g2G2i4K6u0V1k6r3g2U0L8$3c8W2i4K6u0r3x3g2)9J5k6e0u0Q4x3X3f1@1y4W2)9J5c8Y4N6S2M7$3#2Q4y4h3k6$3K9h3c8W2L8#2)9#2k6X3c8W2j5$3!0V1k6g2)9J5k6i4N6S2M7$3@1%60.)
3.  [worker_release.js v1.2.46](https://bbs.kanxue.com/elink@58bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8r3q4V1K9h3&6Q4x3X3g2%4P5s2q4U0L8r3!0#2k6q4)9J5k6i4q4I4i4K6u0W2j5$3!0E0i4K6u0r3j5h3I4S2k6r3W2F1i4K6u0r3k6X3k6E0k6i4m8W2k6#2)9J5c8Y4k6A6k6r3g2G2i4K6u0V1k6r3g2U0L8$3c8W2i4K6u0r3x3g2)9J5k6e0u0Q4x3X3f1@1y4W2)9J5c8Y4N6G2M7X3E0W2M7W2)9#2k6Y4u0W2L8r3g2S2M7$3g2Q4x3X3g2B7M7H3%60.%60.)
4.  [wasm_video_decode.js v1.2.46](https://bbs.kanxue.com/elink@4e6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8r3q4V1K9h3&6Q4x3X3g2%4P5s2q4U0L8r3!0#2k6q4)9J5k6i4q4I4i4K6u0W2j5$3!0E0i4K6u0r3j5h3I4S2k6r3W2F1i4K6u0r3k6X3k6E0k6i4m8W2k6#2)9J5c8Y4k6A6k6r3g2G2i4K6u0V1k6r3g2U0L8$3c8W2i4K6u0r3x3g2)9J5k6e0u0Q4x3X3f1@1y4W2)9J5c8Y4N6S2M7$3#2Q4y4h3k6$3K9h3c8W2L8#2)9#2k6X3c8W2j5$3!0V1k6g2)9J5k6h3A6K6)
5.  [wasm_video_decode_fallback.js v1.2.46](https://bbs.kanxue.com/elink@77aK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8r3q4V1K9h3&6Q4x3X3g2%4P5s2q4U0L8r3!0#2k6q4)9J5k6i4q4I4i4K6u0W2j5$3!0E0i4K6u0r3j5h3I4S2k6r3W2F1i4K6u0r3k6X3k6E0k6i4m8W2k6#2)9J5c8Y4k6A6k6r3g2G2i4K6u0V1k6r3g2U0L8$3c8W2i4K6u0r3x3g2)9J5k6e0u0Q4x3X3f1@1y4W2)9J5c8Y4N6S2M7$3#2Q4y4h3k6$3K9h3c8W2L8#2)9#2k6X3c8W2j5$3!0V1k6g2)9#2k6X3k6S2L8r3I4T1j5h3y4C8i4K6u0W2K9Y4x3%60.)

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)