> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/xPGg_4VKMRbvB0Yl1tblWQ)

0x1 - 前言:

通过上篇文章的学习, 我们将 libpdd_secure.so 样本通过 unidbg 模拟执行并跑通, 通过 unidbg 环境, 我们可以监控样本的 jni 调用以及文件访问, 系统属性访问等, 这减少了我们分析指纹的阻力.

没有看我上一篇的文章可以跳转看看:

> 跳转上一篇
> 
> Fashion 哥，公众号：Fashion 哥移动安全[如何在 OLLVM 混淆中探寻算法二之 libpdd_secure.so 指纹算法分析 (上篇)](https://mp.weixin.qq.com/s/OCNRA_T-EzMcDscQPBH2RQ)

0x2-unidbg 辅助算法还原:

通过对密文的分析:

```
2afXb3VD2jVXPP9GXOCSOw1r5Lcun4PdlXcQpNytHQCanzWVQa4qFMqTbBy5+dIdr+XLoJ+dvPtoJXy75DkU2CDzSfEsUBDZ9qDyrjfv8ES2TPOn2sDuVcKYyQRYKwDNGwYAiYPPPl3+kOHATfNPLKBf3tPey9IwQehBBTQNZXqQTPYsFM4TWoJm1V/hS9maZteSPMzsvfAN2/wNlGyCNJrmJter3upaNkK8x4f5IvCGH3y2ipeXtioRjZw8dvBj1GXkbqU9TpUOxEjOc8bSEoyoFpAh0XKKJGM6w5mnzKngG8ER3Bb+BR7UuGHKSAqggv6yYQkZwfpO2Xf21Mq0LaLFeP1OK6szlUFelh0IdDZ1cxzXCAvwG3wJgmEUVww583+QNf/sh/l5gVhlrerG1iArZ2x9V2JnVeouRN5pFhgPwg8ShYMhrjreI5rNsPVNyAVsRcCLls06QBbBtuk8h1InzA8nr8aIthdufU86M+EWetK0DVdUOzMQg9dgK2CjrBzIoNa/aAq9hBTHO+DDw7jQe7xzGaZ2KT74WC+PIedimoprujeYgANS0z04bIRd+IbTWJWCijBd6tJKQPQ7mdnn4PoxQHgVfHTCNWiOQTj1WOrMmsxPOf9omT6+XXAqKHlNYKMjAi4Iq8UTALzV3lfenNGYsbbQh4qKu36BSEK6RH8lfiat+PiHtPEtCk2JCGHdJhOVCuVH4kbqEES/n0nz7TNDnqcu2sxh+hd3F0rAgg=
```

读者不难发现, 密文开头前 3 字节总是为 2af, 且密文的尾巴时而带上 =, 且有 /,+ 这种字符出现, 凭借这些发现, 读者也能想到 base64 编码算法, 它可太常见了, 尤其是在参数的加密解密中.

0x01-Base64 编码算法

那我们现在就来验证一下我们的猜想, 那 base64 在样本中的体现形式是什么呢? 最显眼的就是那 64 字符的码表:

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

在 ida 中打开字符串这一栏

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCia2yllCo2vnKMAIsVCLD1utPftIjwu3P8B7Aib2dJicRYQyXw4y9rbckQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=0)

我们在 Strings 这一项中进行字符串搜索

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCgv1L4cPJMb9lXWk1XQk8YsfJU7F8m6h85qyl2ugRpTXfDpibNyUEugg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=1)

发现是有结果的, 鼠标点击跳转

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCBtvov0Dic6ibibGgE85EuicAapL937Oh590lxzXLU7iaV6vgndV3bBxWWOw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=2)

鼠标选择红框内容, 按住 x 查找交叉引用

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCVy9Ry7Evl2AzOvHvexCZjibbAmtxF7NODMeHq4iaTfk1EBcdctibfnf9w/640?wx_fmt=png&from=appmsg#imgIndex=3)

发现结果都聚焦在了 sub_180D78 函数.

对此函数静态分析:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCiakVFibMBNZPLGddHXugxWw304S9LXWucL9b7kwPakAs3QZ6aIPg5slg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=4)

下个断点验证一下

```
void Debugger(){
    emulator.attach().addBreakPoint(module,0x180D78);
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCLYicShQEFicjE3ah9DtfTvNh0PqlyI9HHLBQEJk6aib6o3vt4ClHdruEQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=5)

完美断住在此位置

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCxMgicaB1szCj7Pc8bQ3hTLLuuhc9jicw8VYV0cic1Yic7HmccZkQknP4RQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=6)

x0 应该是编码的数据, x1 应该是编码数据的长度, x2 是输出数据的内存缓冲区.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCjaKj9G0ubIvpfUAWkx41JmCpXnicHvuShBlGhjtL4DZEVBtMqXpHrpQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=7)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCKicfKToTsClg6cicUHJVEPtDUm2rv5qZbx0ywiaD3KmDcFCFbznfhvoTg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=8)

手动 base64 一下对比结果

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCuL8yibbckg81FtWQslkCpcVJicicbfv3Iia7oRjBqJ8UuQx6btz4NSphTw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=9)

发现确实是标准的 base64

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCb8N5hsStqwJvac99I4nSZnxYH7fwx6Gcf0m5vaJdUvfpW4pKbfJkHw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=10)

通过静态分析 + 动态调试, 确定了第一个算法 base64.

0x02-AES 算法:

对编码的数据分析, 看不出来是个啥, 我们可以在断在 base64 编码函数的位置, 打印调用堆栈, 通过堆栈回溯能够快速定位编码的数据的来源.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCI7ic13ic6SDuhSNkSviaiacTAsBc2ticGYmV3FWLa8IZqoibh0TsOpYqQPgg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=11)

它的上一个堆栈是偏移 0x02206c,ida 按 g 跳转

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC0xsY736lXzI2nNfFnQmfnVUANmxhd3Jntov1NBab6eGl3um3got95Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=12)

我们的研究对象是这个 v9, 首先他向系统申请了一块 xx 大小的内存, 在 sub_17F3E8 函数中做了写入的操作.

跟进 sub_17F3E8

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCwSCUWicedMDROSQIaDrLB34IBeaepKbfxP1HeMsrS4emucnicCjcib8tA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=13)

发现是个套娃, 继续跟进

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC2oC4FBDJClmoOq66Ofq5m4Oo92IfMLebSmpQnd3721haM9ZbqzLN7g/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=14)

这个才是主要的逻辑.

在此函数 sub_17F458 进行下断点, 看看参数.

```
emulator.attach().addBreakPoint(module,0x17F458);
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC5692ago42XDVZXtzWrzdibCkhOyk29CXMqMLBicKiadKQZ2jrzjJd1tTQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=15)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCcPH3belMKnIhXQicOYdNwaCoRy6RX6oMMvI4wXp2zDicR2UzMhj3ia8yg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=16)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC8rAQkrhxO8JLghYxVQUukuNPibfjWx37LSiaDXh2iczq1iaLB4eZW2W40g/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=17)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCM5X6icyhX2DNArDWLD1XEhYfYVRJ1TDGzCXJtgBPIRa4EO1SCLaFkrg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=18)

这个函数一共 5 个参数, 第一个和第二个不知道是啥, 第三个是疑似加密的数据, 第四个是数据的长度, 第五个是输出数据内存缓存区.

还记得, 上一篇文章我们提到的一个问题, unidbg 补的 jni 调用中, 有这么几个 api 构成了 gzip 压缩算法.

```
Ljava/io/ByteArrayOutputStream;-><init>()V,
Ljava/util/zip/GZIPOutputStream;-><init>(Ljava/io/OutputStream;)V,
Ljava/util/zip/GZIPOutputStream;->write([B)V,
Ljava/util/zip/GZIPOutputStream;->finish()V,
Ljava/io/ByteArrayOutputStream;->toByteArray()[B,
Ljava/util/zip/GZIPOutputStream;->close()V
```

并且根据 gzip 压缩算法的特征, 前两个字节是

```
1f 8b
```

由于 gzip 的双向性, 我们尝试对第三个参数做 gzip 的解压.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCtZFVbN5ZIqK6aR15eLCh5F84HGbITcgdialqib8FCTTnjJcIOfKWMGOA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=19)

解压的结果是采集的指纹, 同时也是算法的第一入参.

对伪 C 代码进行静态分析, 发现了疑似 AES 算法的前置准备工作.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC5zg87Vdia9juYakSxibgxU0u0FSBa0Ndr4eqf7zFcBkXumcD2BDpXibHQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=20)

```
void __fastcall sub_17F458(
        unsigned __int64 key,
        int8x16_t *iv,
        const void *data,
        signed int data_len,
        __int64 outputBuf)
{
  unsigned int _data_len; // w28
  signed int v11; // w8
  int v12; // w27
  char *v13; // x22
  __int64 v14; // x24
  char *v15; // x23
  int8x16_t v16; // [xsp+0h] [xbp-70h] BYREF
  __int64 v17; // [xsp+18h] [xbp-58h]

  _data_len = data_len;
  v17 = *(_ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2)) + 40);
  if ( (data_len & 0xF) != 0 )                  //  检查是否是16字节对齐
  {
    v11 = data_len + 15;
    if ( data_len >= 0 )
      v11 = data_len;
    _data_len = v11 & 0xFFFFFFF0;               // 向上对齐到16字节边界
  }
  v12 = _data_len + 16;                         // 分配额外16字节空间
  v13 = malloc(_data_len + 16);                 // 分配临时缓冲区
  memcpy(v13, data, data_len);                  // 复制原始数据
  if ( (_data_len + 16) > data_len )            // 填充剩余字节
    memset(&v13[data_len], _data_len + 16 - data_len, _data_len + 15 - data_len + 1);
  v16 = 0uLL;
  if ( v12 >= 1 )                               // AES主要流程
  {
    v14 = 0LL;                                  // 块计数器
    do
    {
      v16 = *&v13[v14];                         // 加载16字节数据块
      v16 = veorq_s8(*iv, v16);                 // 与前一密文块（或IV）异或
      v15 = malloc(0xB0u);                      // 分配密钥扩展空间
      sub_17E43C(key, v15, 10, 4);              // 密钥扩展
      sub_17E708(&v16, iv, v15, 10);            // AES加密
      free(v15);
      *(outputBuf + v14) = *iv;                 // 存储加密结果到输出缓冲区
      v14 += 16LL;                              // 移动到下一个块
    }
    while ( v14 < v12 );
  }
  free(v13);
}
```

竟然提到了 aes 就简单的说一下 aes 的主要流程

```
pkcs7填充
在开始之前，AES 会先将 16 字节的明文 排成一个 4x4 的字节矩阵（称为“状态矩阵”或 State）。所有操作都是在这个状态矩阵上进行的。

密钥扩展（Key Expansion）：通过初始密钥生成多个轮密钥（Round Keys），每一轮使用一个轮密钥
初始轮（Initial Round）：执行一次轮密钥加（AddRoundKey）操作
重复轮（Rounds）：执行多轮变换，每一轮包括：
    字节替代（SubBytes）：使用S盒对每个字节进行非线性替换。
    行移位（ShiftRows）：对状态矩阵的每一行进行循环左移。
    列混合（MixColumns）：对状态矩阵的每一列进行线性变换（最后一轮除外）。
    轮密钥加（AddRoundKey）：将当前状态与轮密钥进行按位异或。

最终轮（Final Round）：与重复轮类似，但省略列混合步骤，即包括：
    字节替代（SubBytes）
    行移位（ShiftRows）
    轮密钥加（AddRoundKey）

128位(16字节)密钥：10轮
192位(24字节)密钥：12轮
256位(32字节)密钥：14轮
```

此函数内部调用了 sub_17E43C 和 sub_17E708 函数, 此函数只是做了 aes 算法的准备工作, 主要逻辑应该在这两个函数内部.

对 sub_17E43C 函数进行分析, 发现与 aes 的密钥扩展算法相似.

这是 aes 密钥扩展的算法的伪代码

```
w[0] ~ w[Nk-1] = 原始密钥
for i from Nk to Nb*(Nr+1)-1:
    temp = w[i-1]
    if (i % Nk == 0):
        temp = SubWord(RotWord(temp)) XOR Rcon[i/Nk]
    else if (Nk > 6 and i % Nk == 4):  # AES-256特有
        temp = SubWord(temp)
    w[i] = w[i-Nk] XOR temp
```

```
unsigned __int64 __fastcall sub_17E43C(unsigned __int64 result, unsigned __int64 a2, char a3, int a4)
{
  __int64 v6; // x10
  unsigned int v7; // w9
  unsigned int v8; // w10
  __int64 v9; // x11
  __int64 v10; // x23
  unsigned int v11; // w24
  __int64 v12; // x28
  unsigned __int64 v13; // x10
  __int64 v14; // x8
  __int64 v15; // x27
  __int64 v16; // x26
  __int64 v17; // x25
  int v18; // w9
  unsigned __int8 v19; // w21
  int v20; // w22
  __int64 v21; // x9
  unsigned __int64 v22; // x11
  __int64 v23; // x9
  unsigned int v24; // w11
  unsigned int v25; // w12
  __int128 *v26; // x13
  __int128 v27; // q1
  __int128 v28; // q2
  __int128 v29; // q3
  _OWORD *v30; // x14

  v6 = (a4 - 1);
  _ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2));
  if ( a4 >= 1 )                                // 将原始密钥复制到扩展密钥数组的前key_words个字
  {
    if ( a4 < 0x10 )                            // 使用向量化优化复制（每次16字节）
      goto LABEL_3;
    v7 = 0;
    if ( !a4 )
      goto LABEL_4;
    if ( v6 > 0xFF )
      goto LABEL_4;
    v22 = 4 * v6;
    if ( 4 * v6 > ~(a2 + 2) || v22 > ~(a2 + 3) || v22 > ~(a2 + 1) || v22 > ~a2 )
      goto LABEL_4;
    v23 = 4 * v6 + 4;
    if ( result + v23 > a2 && a2 + v23 > result )
    {
LABEL_3:
      v7 = 0;
    }
    else
    {
      v7 = a4 & 0xFFFFFFF0;
      v24 = 3;
      v25 = a4 & 0xFFFFFFF0;
      do
      {
        v26 = (result + v24 - 3);               // 批量复制：每次处理4个字（16字节）
        v27 = v26[3];                           // 加载4个字
        v28 = *v26;
        v29 = v26[1];
        v30 = (a2 - 3 + v24);
        v25 -= 16;
        v24 += 64;
        v30[2] = v26[2];                        // 存储到扩展密钥数组
        v30[3] = v27;
        *v30 = v28;
        v30[1] = v29;
      }
      while ( v25 );
      if ( v7 == a4 )
        goto LABEL_6;
    }
LABEL_4:
    v8 = v7;
    do
    {
      ++v8;
      *(a2 + 4LL * v7) = *(result + 4LL * v7);
      *(a2 + ((4LL * (v7 & 0x3FFFFFFF)) | 1)) = *(result + ((4LL * (v7 & 0x3FFFFFFF)) | 1));
      *(a2 + ((4LL * (v7 & 0x3FFFFFFF)) | 2)) = *(result + ((4LL * (v7 & 0x3FFFFFFF)) | 2));
      v9 = (4LL * (v7 & 0x3FFFFFFF)) | 3;
      v7 = v8;
      *(a2 + v9) = *(result + v9);
    }
    while ( v8 < a4 );
  }
LABEL_6:
  v10 = a4;
  v11 = (4 * a3 + 4) & 0xFC;                    // 计算总扩展字数
  if ( v11 > a4 )
  {
    result = 2LL;
    do
    {
      v12 = v10 << 32 >> 30;                    // 计算当前字偏移（v10 * 4）
      v13 = a2 + v12;
      v14 = *(v13 - 4);                         // 获取前一个字
      v15 = *(a2 + v12 - 3);
      v16 = *(v13 - 2);
      v17 = *(v13 - 1);
      v18 = v10 / a4;
      if ( v10 % a4 )                           // 每NK个字进行完整变换（RotWord + SubWord + Rcon）
      {
        if ( a4 >= 7 && v10 % a4 == 4 )         // AES-256的特殊情况：每8个字的第4个字进行S盒替换
        {
          LOBYTE(v14) = byte_18DA5C[v14];
          LOBYTE(v15) = byte_18DA5C[v15];
          LOBYTE(v16) = byte_18DA5C[v16];
          LOBYTE(v17) = byte_18DA5C[v17];
        }
      }
      else
      {
        v19 = byte_18DA5C[v15];                 // S盒替换（SubWord）
        LOBYTE(v15) = byte_18DA5C[v16];
        LOBYTE(v16) = byte_18DA5C[v17];
        LOBYTE(v17) = byte_18DA5C[v14];         // 循环左移（RotWord）
        if ( (v10 / a4) )                       // 轮常数异或（Rcon）
        {
          if ( (v10 / a4) == 1 )
          {
            result = 1LL;
          }
          else
          {
            v20 = v18 - 1;
            result = 2LL;
            if ( (v18 - 1) >= 2u )
            {
              do
              {
                result = sub_17F808(result, 2LL);// 计算轮常数
                --v20;
              }
              while ( v20 > 1u );
            }
          }
        }
        LOBYTE(v14) = result ^ v19;             // 异或轮常数
      }
      v21 = ((v10 - a4) << 32) >> 30;
      *(a2 + v12) = *(a2 + v21) ^ v14;          // w[i] = w[i-Nk] XOR 变换后的w[i-1]
      *(a2 + (v12 | 1LL)) = *(a2 + (v21 | 1LL)) ^ v15;
      *(a2 + (v12 | 2LL)) = *(a2 + (v21 | 2LL)) ^ v16;
      v10 = (v10 + 1);                          // 处理下一个字
      *(a2 + (v12 | 3LL)) = *(a2 + (v21 | 3LL)) ^ v17;
    }
    while ( v11 > v10 );
  }
  return result;
}
```

byte_18DA5C 就是 aes 的 s 盒

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCvs7SSGfKC2Io50iapjRkwRVxYxodIBMnf49FYHq3ibjUGhT24cXu3q5g/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=21)

(图为标准 s 盒)

```
.rodata:000000000018DA5C byte_18DA5C     DCB 0x63, 0x7C, 0x77, 0x7B, 0xF2, 0x6B, 0x6F, 0xC5, 0x30
.rodata:000000000018DA5C                                         ; DATA XREF: sub_17E43C+F4↑o
.rodata:000000000018DA5C                                         ; sub_17E43C:loc_17E54C↑o ...
.rodata:000000000018DA5C                 DCB 1, 0x67, 0x2B, 0xFE, 0xD7, 0xAB, 0x76, 0xCA, 0x82
.rodata:000000000018DA5C                 DCB 0xC9, 0x7D, 0xFA, 0x59, 0x47, 0xF0, 0xAD, 0xD4, 0xA2
.rodata:000000000018DA5C                 DCB 0xAF, 0x9C, 0xA4, 0x72, 0xC0, 0xB7, 0xFD, 0x93, 0x26
.rodata:000000000018DA5C                 DCB 0x36, 0x3F, 0xF7, 0xCC, 0x34, 0xA5, 0xE5, 0xF1, 0x71
.rodata:000000000018DA5C                 DCB 0xD8, 0x31, 0x15, 4, 0xC7, 0x23, 0xC3, 0x18, 0x96
.rodata:000000000018DA5C                 DCB 5, 0x9A, 7, 0x12, 0x80, 0xE2, 0xEB, 0x27, 0xB2, 0x75
.rodata:000000000018DA5C                 DCB 9, 0x83, 0x2C, 0x1A, 0x1B, 0x6E, 0x5A, 0xA0, 0x52
.rodata:000000000018DA5C                 DCB 0x3B, 0xD6, 0xB3, 0x29, 0xE3, 0x2F, 0x84, 0x53, 0xD1
.rodata:000000000018DA5C                 DCB 0, 0xED, 0x20, 0xFC, 0xB1, 0x5B, 0x6A, 0xCB, 0xBE
.rodata:000000000018DA5C                 DCB 0x39, 0x4A, 0x4C, 0x58, 0xCF, 0xD0, 0xEF, 0xAA, 0xFB
.rodata:000000000018DA5C                 DCB 0x43, 0x4D, 0x33, 0x85, 0x45, 0xF9, 2, 0x7F, 0x50
.rodata:000000000018DA5C                 DCB 0x3C, 0x9F, 0xA8, 0x51, 0xA3, 0x40, 0x8F, 0x92, 0x9D
.rodata:000000000018DA5C                 DCB 0x38, 0xF5, 0xBC, 0xB6, 0xDA, 0x21, 0x10, 0xFF, 0xF3
.rodata:000000000018DA5C                 DCB 0xD2, 0xCD, 0xC, 0x13, 0xEC, 0x5F, 0x97, 0x44, 0x17
.rodata:000000000018DA5C                 DCB 0xC4, 0xA7, 0x7E, 0x3D, 0x64, 0x5D, 0x19, 0x73, 0x60
.rodata:000000000018DA5C                 DCB 0x81, 0x4F, 0xDC, 0x22, 0x2A, 0x90, 0x88, 0x46, 0xEE
.rodata:000000000018DA5C                 DCB 0xB8, 0x14, 0xDE, 0x5E, 0xB, 0xDB, 0xE0, 0x32, 0x3A
.rodata:000000000018DA5C                 DCB 0xA, 0x49, 6, 0x24, 0x5C, 0xC2, 0xD3, 0xAC, 0x62, 0x91
.rodata:000000000018DA5C                 DCB 0x95, 0xE4, 0x79, 0xE7, 0xC8, 0x37, 0x6D, 0x8D, 0xD5
.rodata:000000000018DA5C                 DCB 0x4E, 0xA9, 0x6C, 0x56, 0xF4, 0xEA, 0x65, 0x7A, 0xAE
.rodata:000000000018DA5C                 DCB 8, 0xBA, 0x78, 0x25, 0x2E, 0x1C, 0xA6, 0xB4, 0xC6
.rodata:000000000018DA5C                 DCB 0xE8, 0xDD, 0x74, 0x1F, 0x4B, 0xBD, 0x8B, 0x8A, 0x70
.rodata:000000000018DA5C                 DCB 0x3E, 0xB5, 0x66, 0x48, 3, 0xF6, 0xE, 0x61, 0x35, 0x57
.rodata:000000000018DA5C                 DCB 0xB9, 0x86, 0xC1, 0x1D, 0x9E, 0xE1, 0xF8, 0x98, 0x11
.rodata:000000000018DA5C                 DCB 0x69, 0xD9, 0x8E, 0x94, 0x9B, 0x1E, 0x87, 0xE9, 0xCE
.rodata:000000000018DA5C                 DCB 0x55, 0x28, 0xDF, 0x8C, 0xA1, 0x89, 0xD, 0xBF, 0xE6
.rodata:000000000018DA5C                 DCB 0x42, 0x68, 0x41, 0x99, 0x2D, 0xF, 0xB0, 0x54, 0xBB
.rodata:000000000018DA5C                 DCB 0x16
```

且是标准的 aes 算法的 s 盒

在继续跟进到 sub_17E708 函数分析伪 C 代码

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCZgY62Ka7DTrIxyIycvZ5ianzKicNelqKHjiaVzC2BT0GeMibpBb60Rl4kw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=22)

```
  v11 = v7 ^ *a1;
  v12 = a1[1];
  v13 = a1[5];
  v14 = a1[9];
  v15 = a1[13];
  v16 = a1[2];
  v17 = a1[6];
  v18 = a1[10];
  v19 = a1[14];
  v20 = a1[3];
  v21 = a1[7];
  v22 = a1[11];
  v23 = a1[15];
  v82 = v11;
  v24 = a3[1] ^ v12;
  v86 = v24;
  v25 = a3[2] ^ v16;
  v90 = v25;
  v26 = a3[3] ^ v20;
  v94 = v26;
  v27 = a3[4] ^ v8;
  v83 = v27;
  v28 = a3[5] ^ v13;
  v87 = v28;
  v29 = a3[6] ^ v17;
  v91 = v29;
  v30 = a3[7] ^ v21;
  v95 = v30;
  v31 = a3[8] ^ v9;
  v84 = v31;
  v32 = a3[9] ^ v14;
  v88 = v32;
  v33 = a3[10] ^ v18;
  v92 = v33;
  v34 = a3[11] ^ v22;
  v96 = v34;
  v35 = a3[12] ^ v10;
  v85 = v35;
  v36 = a3[13] ^ v15;
  v89 = v36;
  v37 = a3[14] ^ v19;
  v93 = v37;
  v38 = a3[15] ^ v23;
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCzlIWUN2CBf48aSYlWkZnpXUTnHmQJGJb7hZKWjfx1enfZKqjAktj3A/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=23)

```
for ( i = v38; ; i ^= v39 )
  {
    v40 = v24;
    v41 = v25;
    v42 = v29;
    v43 = v33;
    v44 = v26;
    v45 = v30;
    v46 = v34;
    v47 = v38;
    v48 = byte_18DA5C[v11];
    v49 = byte_18DA5C[v27];
    v50 = byte_18DA5C[v31];
    v51 = byte_18DA5C[v35];
    v52 = byte_18DA5C[v40];
    v53 = byte_18DA5C[v28];
    v54 = byte_18DA5C[v32];
    v55 = byte_18DA5C[v36];
    v56 = byte_18DA5C[v41];
    v57 = byte_18DA5C[v42];
    v58 = byte_18DA5C[v43];
    v59 = byte_18DA5C[v37];
    v60 = byte_18DA5C[v44];
    v61 = byte_18DA5C[v45];
    v62 = byte_18DA5C[v46];
    v63 = byte_18DA5C[v47];
    v82 = v48;
    v83 = v49;
    v84 = v50;
    v85 = v51;
    v86 = v53;
    v87 = v54;
    v88 = v55;
    v89 = v52;
    v90 = v58;
    v91 = v59;
    v92 = v56;
    v93 = v57;
    v94 = v63;
    v95 = v60;
    v96 = v61;
    i = v62;
    if ( a4 <= v6 )
      break;
    sub_180A24(&v82);
    v11 = a3[16 * v6] ^ v82;
    v82 = v11;
    v24 = a3[(16LL * v6) | 1] ^ v86;
    v86 = v24;
    v25 = a3[(16LL * v6) | 2] ^ v90;
    v90 = v25;
    v26 = a3[(16LL * v6) | 3] ^ v94;
    v94 = v26;
    v27 = a3[(16LL * v6) | 4] ^ v83;
    v83 = v27;
    v28 = a3[(16LL * v6) | 5] ^ v87;
    v87 = v28;
    v29 = a3[(16LL * v6) | 6] ^ v91;
    v91 = v29;
    v30 = a3[(16LL * v6) | 7] ^ v95;
    v95 = v30;
    v31 = a3[(16LL * v6) | 8] ^ v84;
    v84 = v31;
    v32 = a3[(16LL * v6) | 9] ^ v88;
    v88 = v32;
    v33 = a3[(16LL * v6) | 0xA] ^ v92;
    v92 = v33;
    v34 = a3[(16LL * v6) | 0xB] ^ v96;
    v96 = v34;
    v35 = a3[(16LL * v6) | 0xC] ^ v85;
    v85 = v35;
    v36 = a3[(16LL * v6) | 0xD] ^ v89;
    v89 = v36;
    v37 = a3[(16LL * v6) | 0xE] ^ v93;
    v93 = v37;
    v39 = a3[(16LL * v6++) | 0xF];
    v38 = v39 ^ i;
  }
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC38eLMBRdWrQ90IxBVGl9ZeeDFNTwerrWZOnAz66x9KjdWMjodK6JoA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=24)

```
v64 = (16 * a4) & 0xFF0LL;
  v80 = a3[v64];
  v77 = a3[v64 | 7];
  v79 = a3[v64 | 0xD];
  v65 = a3[v64 | 1];
  v66 = a3[v64 | 0xF] ^ v62;
  v67 = a3[v64 | 2];
  v68 = a3[v64 | 3];
  v69 = a3[v64 | 4];
  v78 = a3[v64 | 0xE];
  v70 = a3[v64 | 5];
  v71 = a3[v64 | 6];
  v72 = a3[v64 | 8];
  v73 = a3[v64 | 9];
  v74 = a3[v64 | 0xA];
  v75 = a3[v64 | 0xB];
  a2[12] = a3[v64 | 0xC] ^ v51;
  result = v75 ^ v61;
  *a2 = v80 ^ v48;
  a2[4] = v69 ^ v49;
  a2[8] = v72 ^ v50;
  a2[1] = v65 ^ v53;
  a2[5] = v70 ^ v54;
  a2[9] = v73 ^ v55;
  a2[13] = v79 ^ v52;
  a2[2] = v67 ^ v58;
  a2[6] = v71 ^ v59;
  a2[10] = v74 ^ v56;
  a2[14] = v78 ^ v57;
  a2[3] = v68 ^ v63;
  a2[7] = v77 ^ v60;
  a2[11] = result;
  a2[15] = v66;
```

完美贴合了 aes 的主加密流程

```
1. AddRoundKey (初始轮)
2. for round = 1 to Nr-1:
   - SubBytes
   - ShiftRows  
   - MixColumns
   - AddRoundKey
3. 最后一轮:
   - SubBytes
   - ShiftRows
   - AddRoundKey
```

竟然有了分析的方向 (AES 算法), 那么就得确定下是 aes-ecb 呢还是 aes-cbc 呢, 根据 cbc 模式的特点 (比 ecb 多了个 iv, 每个明文块在加密前会与前一个密文块进行异或操作. 第一个明文块与初始化向量（IV）进行异或).

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCNBQZSTs37riaJEwr7oHltxvCjhloTrHMFAiaeLrENolFBjpXXeianwIiaw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=25)

在此处能够得到确定, 每个明文块在加密前会与前一个密文块进行异或操作, 所以他是 aes 的 cbc 模式, 在上面的分析过程中, 提到了两个我们不知道的参数, 有可能就是我们的 key 和 iv. 根据 key 的长度可以确定是那种 aes

```
128位(16字节)密钥：10轮
192位(24字节)密钥：12轮
256位(32字节)密钥：14轮
```

通过验证 sub_17F458 函数第一个参数是 key

```
pdd_aes_180121_1
```

sub_17F458 函数第二个参数是 iv

```
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

尝试解密结果

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCC8Cljs9VSx0ANibknbibuIrA7hWOeHELMV3Iibf0hxTfJh420V1sRI3VbQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=26)

能够解密出指纹数据

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCfR8bKvLjmEPAqd9BgCViaovSsjo0Y87YwclDl5cjuQZXPIGAnP61uqQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=27)

0x03 - 指纹数据分析

接下来是指纹数据的分析, 一共 34 组.

这里先讲一个 api

```
__system_property_get
```

函数原型:

```
int __system_property_get(const char* name, char* value);
```

__system_property_get 是 Android 系统中的一个内部 API，用于获取系统属性（system properties）的值. 很多的收集指纹的样本都会使用此 api.

```
// 属于 libc 库（Bionic C Library）
#include <sys/system_properties.h>  // 头文件
函数原型
int __system_property_get(const char* name, char* value);
参数说明:
name：属性名称（以 null 结尾的字符串）
value：输出缓冲区，用于接收属性值
返回值：成功返回字符串长度，失败返回 0 或负数
```

在 unidbg 中, 如果样本使用__system_property_get 此 api,unidbg 会从

`src/main/resources/android/sdk23/dev/__properties__`文件内取 name 对应的值. 所以在指纹数据中会有些 unidbg 默认的指纹, 这就很不灵活, 不利于随机化, 有两种方案, 其一是, 我们可以从真机上 pull 一份__properties__, 但是这不利于我们随机化. 其二是对这个 api 进行 hook.

这里选用第二种

```
SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
    @Override
    public String getProperty(String key) {
        System.out.println("systemPropertyHook key ====>"+key);
        return null;
});
memory.addHookListener(systemPropertyHook);
```

放在 emulator.getMemory() 之后的位置.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCCpHBgwbicfQ2S0o9kh93f5h0HxA5iaMefrdF01XCmXj501XU9UkiaWrAQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=28)

运行之后可以发现它访问了很多属性

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCddHxtyN0HA6llIDEdfqARNbTnjc0SExKyjBafJNj5Gc6SjpxfN1Yvw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=29)

在 ida 导入表中, 查找这个函数的交叉引用, 发现也很多

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCV8YA9I7oR7W1rXGGCpyNuLaS1HYmnCdI1iapINwFnfkyBqUlzU5YFKg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=30)

处理的方式可以写一个 switch-case, 先按照真机的结果返回, 当然也可以改成显眼的字符串, 这在找指纹时, 就能快速的找到.

```
SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
    @Override
    public String getProperty(String key) {
        System.out.println("systemPropertyHook key ====>"+key);
        switch (key){
            case "ro.kernel.qemu":{
                return "null";
            }
            case "libc.debug.malloc":{
                return "null";
            }
            case "ro.build.version.sdk":{
                return "29";
            }
            case "ro.serialno":{
                return "niubi";
            }
            case "ro.product.brand":{
                return "xiaomi";
            }
            case "ro.product.device":{
                return "iphone";
            }
            case "ro.product.model":{
                return "kool";
            }
            case "ro.product.manufacturer":{
                return "pool";
            }
            case "ro.product.board":{
                return "mooll";
            }
            case "ro.build.display.id":{
                return "QKQ1.190910.002 test-keys";
            }
            case "ro.build.id":{
                return "QKQ1.190910.002";
            }
            case "ro.build.version.incremental":{
                return "V12.0.2.0.QDTCNXM";
            }
            case "ro.build.type":{
                return "user";
            }
            case "ro.build.tags":{
                return "release-keys";
            }
            case "ro.build.version.release":{
                return "10";
            }
            case "ro.build.date.utc":{
                return "1615472652";
            }
        }
        return null;
    }
});
memory.addHookListener(systemPropertyHook);
```

多多的指纹排序采用的 TLV 格式 (Type-Length-Value), 这里讲一下 TLV, 举个例子, 假设数据是 fashion, 它的 TLV 形式是这样的

```
01 07 66617368696f6e
```

01 代表类型, 这个类型可以自定义, 07 代表数据的长度, 66617368696f6e 是数据进行 hex 编码的结果.

那么基于 tlv, 我们可以对指纹数据做数据划分.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06P4G9zgQTBlKTBRo8icQ1BCCjGJu8z0uhIeNicNZ3KuiaziammL2HzK9o6ticJLDMt32eicTv8q9cgeCLlQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=31)

前面 6 字节固定, 后面的一组指纹按照这种格式排序

```
指纹长度+3 类型(0101-0122) 指纹长度 指纹值(Hex编码)
```

所以我们的研究对象是指纹值.

在 unidbg 补的环境中, 我们可以通过控制变量的形式, 确定补的环境是哪一个指纹, 是否参与了收集.

经过我的分析, 控制变量以及对比真机, 采集的指纹如下, 分别是对应着 34 组指纹.

```
1.ro.serialno
2.ro.product.brand
3.ro.product.device
4.ro.product.model
5.ro.product.manufacturer
6.ro.product.board
7.ro.build.display.id
8.ro.build.id + ro.build.version.incremental + ro.build.type + ro.build.tags
9.ro.build.version.release
10.ro.build.version.sdk
11.ro.build.date.utc
12./system/build.prop文件最后的修改时间时间戳16进制
13.0字节空
14.0字节空
15.0字节空
16.0字节空
17.0字节空
18.android/telephony/TelephonyManager->getSimState()I
19.android/telephony/TelephonyManager->getSimOperatorName()Ljava/lang/String;
20.android/telephony/TelephonyManager->getSimCountryIso()Ljava/lang/String;
21.0字节空
22.android/telephony/TelephonyManager->getNetworkType()I 对应的常量标识
23.android/telephony/TelephonyManager->getNetworkOperator()Ljava/lang/String;
24.0字节空
25.android/telephony/TelephonyManager->getNetworkOperatorName()Ljava/lang/String;
26.android/telephony/TelephonyManager->getNetworkCountryIso()Ljava/lang/String;
27.android/telephony/TelephonyManager->getDataState()I
28.android/telephony/TelephonyManager->getDataActivity()I
29.----真机环境不存在此条
30.com/xunmeng/pinduoduo/secure/EU->gad()Ljava/lang/String;(不进行hex编码)
31.1字节空
32.info2方法传入参数时间戳16进制
33.随机数--->lrand48() + (lrand48() >> 32)
34.uuid
```

这个样本至此就分析结束了, 总结来说算法用了以下的结构, 与 libyzwg.so 有着异曲同工之妙.

```
Base64(AES(gzip(Fpdata)))
```

0x3 - 笔者想说的话:

笔者文章中分析的样本为 7.80 版本, 最新版为 7.85 版本, 算法上几乎没有任何变化, key 和 iv 这些都是通用的, 唯一的区别在于, anti-token 有两种, 区分它的种类看开头的前 3 字节, 2af 开头的是加密 gzip 压缩后的指纹数据, 结果一般比较长, 2ag 开头的, 一般比较短加密的数据有两种如下 (不进行 gzip 压缩直接 aes 加密):

```
083f //固定
4841101d993eb65d //com/xunmeng/pinduoduo/secure/EU->gad()结果
0000019ad2e527177d86b6c26337f607 //未知
3eac7328 //uuid的hashcode
306c754e59736b50 //请求头的etag
409eeebfc3f944bda7ecddd0dc11d3d6 //固定
```

```
081f //固定
49b8d5557d5bafd9 //com/xunmeng/pinduoduo/secure/EU->gad()结果
0000019ad32aa3217f98d83917a40b9e //未知
7d02bd9b //uuid的hashcode
6c4b457975624c47 //请求头的eatag
```

另外的话, 多多的风控比较严, 如果点开商品详情是已售空这种的, 基本是设备被标记了 (改机解决) 或者号死了(烧号解决), 但是多多 ios 端的风控相对安卓而言就没有那么的严格, ios 也有 anti-token 参数, 1ab 开头前缀的, 后续有空也会来一篇文章对此进行分析, 毕竟我们的目标是星辰大海嘛.

0x4 - 结语:

考虑到看我文章的人, 很多都是小白, 刚刚入门, 对文章所涉及到的思路以及操作比较陌生, 下篇要讲的 libjdgs.so 补环境涉及到了上下文初始化问题以及 unidbg 检测, 算法不单单有 ollvm 还涉及了 vmp, 后面的文章将会转型, 以 unidbg 补 jni 调用 / 文件访问 / 库函数 / 系统调用 / 上下文环境 (初始化问题) 为核心展开讲解, unidbg 如何处理这些问题后面在扩展至 Hook/Patch/Debugger/Trace,unidbg 检测以及 anti 等.