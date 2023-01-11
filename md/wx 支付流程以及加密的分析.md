> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1085646-1-1.html)

> 本贴仅供学习交流，请勿用作其他任何用途 19 年底捣鼓了一阵 wx 协议，但没有相关资料所以摸索了挺久。

![](https://avatar.52pojie.cn/data/avatar/001/03/30/62_avatar_middle.jpg)三年三班三井寿 _ 本帖最后由 三年三班三井寿 于 2020-1-9 18:47 编辑_  
本贴仅供学习交流，请勿用作其他任何用途  
  
19 年底捣鼓了一阵 wx 协议，但没有相关资料所以摸索了挺久。后来找了一套旧版的协议源码，奈何支付协议是转不了帐的，所以自己就看了一下这个。而有了协议框架，我们只需要去看转账相关的组包逻辑。  
**准备：**  
分析工具：xp,frida,jadx,IDA               wx 版本：都差不多，706，707，708 都看过  
首先开启 wx 的日志 xlog，直接搜索  
 ![](https://attach.52pojie.cn/forum/202001/04/173731dnlfs49zk4433i0i.png)   
然后找到开关 isLogcatOpen 赋值的地方，修改第五个参数为 1 即可  
 ![](https://attach.52pojie.cn/forum/202001/04/174526e91jxodkoykxmxmj.png)   
当然也可以直接找其上层调用  
 ![](https://attach.52pojie.cn/forum/202001/04/174748b5mbn51kg3b7505n.png)   
 ![](https://attach.52pojie.cn/forum/202001/04/174750ipbdwwa5dbw5zmzs.png)   
[JavaScript] _纯文本查看_ _复制代码_

```
var XLOG=Java.use("com.tencent.mm.sdk.platformtools.ab");
//var StringClz=Java.use("java.lang.String");
XLOG.i.overload('java.lang.String', 'java.lang.String', '[Ljava.lang.Object;').implementation=function(s1,s2,s3){
if(s3==null){
    console.log("i:"+s1+","+s2);
}else{
    console.log("i:"+s1+","+s2+s3);
}
    return this.i(arguments[0],arguments[1],arguments[2]);
}
XLOG.f.overload('java.lang.String', 'java.lang.String', '[Ljava.lang.Object;').implementation=function(s1,s2,s3){
if(s3==null){
    console.log("f:"+s1+","+s2);
}else{
    console.log("f:"+s1+","+s2+s3);
}
    return this.f(arguments[0],arguments[1],arguments[2]);
}
```

  
微信组包以及加解密的 so 文件，LibMMProticalJni.so, 其组包函数为 pack  
 ![](https://attach.52pojie.cn/forum/202001/04/175022t6xll3lke171z33l.png)   
我们直接 hook 其 Java 层调用  
 ![](https://attach.52pojie.cn/forum/202001/04/175026orr4rgbbnrepennb.png)   
[JavaScript] _纯文本查看_ _复制代码_

```
var MMProtocalJni=Java.use("com.tencent.mm.protocal.MMProtocalJni");
[/color][/size][/color][/size][size=5][color=red][size=3][color=black]   MMProtocalJni.pack.implementation=function(){  
       console.log("MMpack:"+bytesToHex(arguments[0]));
       return this.pack(arguments[0],arguments[1],arguments[2],arguments[3],arguments[4],arguments[5],arguments[6],arguments[7],arguments[8],arguments[9],arguments[10],arguments[11],arguments[12],arguments[13]);
   }
```

首先我们扫付款码，这是之前用 xp hook 的日志，前面一些基本都是环境设备信息，deviceId,clientVersion 之类的，和其他功能组包类似，我们所需要关注的是后面的字符串。明显的是 transfer_url 就是二维码的字串，进行了简单的编码。WCPaySign 是本地计算出来的一个值，而 WCPaySign 以及 channel 之间还有一段序列特征，具体是啥也没深入研究，有知道的大佬可以告诉我一下。测试中有时会有 encrypt_key 以及 encrypt_userinfo 字段，具体也是由 WCPaySign 以及 channel 之间字段决定的。 ![](https://attach.52pojie.cn/forum/202001/04/175633ga4hrahaara22haz.png)  transfer_url 是直接通过 java 库函数 URLEncoder 解析得到的：  ![](https://attach.52pojie.cn/forum/202001/04/180316wgkuzyukpzmunru0.png)  我们可以在这里替换用户扫描的二维码，使得对方不管扫什么码都会跳到我们自己的二维码上。比较简单的做法就是直接 hook URLEncoder, 通过收款码特征或者堆栈进行判断过滤，当然也可以自己重写这个 w 类构造，构造一个 hashMap 传入[JavaScript] _纯文本查看_ _复制代码_

```
URLEN.encode.overload("java.lang.String").implementation=function(str){
       console.log("URLEN");
       var res=this.encode(str);
       var stack=instance.currentThread().getStackTrace();
       var full_call_stack=where(stack);
       return res.indexOf("wxp%3A%2F%2F")==0&&full_call_stack.indexOf("com.tencent.mm.plugin.remittance.model.w.<init>")?
       "wxp%3A%2F%2Ff2f0NtReekKHV87BM0pY6k3TVjHlljtYL4sQ":res
   }
```

不过如果这样做有一个问题，就是不管是扫谁的二维码，都会跳转到自己的付款页面。那么能不能实现，扫谁的码就出现给谁付款的页面，但实际转账却并不是给他转？换句话说就是页面上显示的一切都是正常的，但实际却转给了另一个人？当然是可以的，不过我们需要先进行模拟扫我们码的操作，然后获取到返回的 openid,ticket 等。之后在后面付款的时候将正常的这些字段替换成我们的就可以了。扯远了，下面进行定位，寻找 WCPaySign 的算法。调用堆栈如下，直接调用函数为 com.tencent.mm.ak.t.a ![](https://attach.52pojie.cn/forum/202001/04/182224tf5s1x7i8v4784gk.png)  这个函数比较长，jadx 反编译的有问题需要修改设置选项，也可以用 jd 直接查看调用位置  ![](https://attach.52pojie.cn/forum/202001/04/182405xhz3almwbeu2aolw.png)  接着找参数赋值的地方，由一个成员变量 req 实例化为 l.b 类型后，进行了序列化  ![](https://attach.52pojie.cn/forum/202001/04/182548nqxrbnqrgnuss1zk.png)  而 req 的初始化在 t 构造函的数中  ![](https://attach.52pojie.cn/forum/202001/04/182643g59nr0nu05v1ne0v.png)  同样的方法 hook 其构造函数，调用堆栈如下 ![](https://attach.52pojie.cn/forum/202001/04/182730oqgar3qdrjylrapa.png)   
前两个不用看了，u 的构造参数 qVar.getReqObj 也是 t 的构造参数，而 u 的构造是在 com.tencent.mm.ak.m.dispatch 中，构造参数是它的第二个参数 q.getReqObj() 的返回值。 ![](https://attach.52pojie.cn/forum/202001/04/184142kw2fvp2havftvfxy.png)  也是 com.tencent.mm.wallet_core.c.u.dispatch 的第二个参数  ![](https://attach.52pojie.cn/forum/202001/04/184229bngns3tf21kt222g.png)  ![](https://attach.52pojie.cn/forum/202001/04/184231w0nmp8nfaqmjomnf.png)   
具体赋值的地方在 com.tencent.mm.wallet_core.tenpay.model.m.doScene 中的 rr ![](https://attach.52pojie.cn/forum/202001/04/184358r2ir61avrrjzri2p.png)  这里的 rr，之前的 q，以及最初的 req，他们的类型其实都不一样。rr 实际上是 com.tencent.mm.wallet_core.tenpay.model.m 所继承的父类的成员，该类也正是我们所需要找的，赋值的地方在 setRequestData 中的最后。在此之前，正是计算了 WCPaySign。  ![](https://attach.52pojie.cn/forum/202001/04/184605xi9mfamt82skrmr9.png)  getEncryptUrl 是 q 中一个抽象的方法，实现如下  ![](https://attach.52pojie.cn/forum/202001/04/184648knwqwcgh10a6aqdq.png)  从名称看就是一个 3DES(md5(str)) 的算法，实际上也确实如此  ![](https://attach.52pojie.cn/forum/202001/04/184803hveydasvqaavpyqz.png)  其 md5 计算只是在 Java 层调用的标准库  ![](https://attach.52pojie.cn/forum/202001/04/184849gnw6vxbt6trvvmvn.png)  3DES 算法 Java 层调用接口为 encrypt，前一个参数是 key，没有则默认  ![](https://attach.52pojie.cn/forum/202001/04/184939wimrm5oppmponnmp.png)  除此之外，接口类 q 中提供了 getUri 接口，其返回值是 post 的 cgi 目录，在 com.tencent.mm.wallet_core.c.u onGYNetEnd 中的第四个参数传入了 q 的实例  ![](https://attach.52pojie.cn/forum/202001/04/185111hive7bc5eeahtiby.png)  通过反射获取到 post 接口为 / cgi-bin/mmpay-bin/transferscanqrcode接下来进入 so 层  ![](https://attach.52pojie.cn/forum/202001/04/185205blk99p58k3829j2z.png)  调用的 so 为 libtenpay_utils.so，也是标准生成的 C 函数  ![](https://attach.52pojie.cn/forum/202001/04/185311ng2vl8akeq2l1lrq.png)  在 encrypt 中很容易发现默认密钥  ![](https://attach.52pojie.cn/forum/202001/04/185357h8o40tko5mfppdvb.png)  但用这 key 尝试了几种加密方式，结果都不正确  ![](https://attach.52pojie.cn/forum/202001/04/185443iggtxg6hwlfqb3gb.png)   ![](https://attach.52pojie.cn/forum/202001/04/185445w3bzqt29yqby2sd9.png)  可能并非标准的算法，看了一下置换表也没发现有什么变化，直接将其算法抠出来Des3Str 就是分组加密函数，Des3 进行了单个分组加密  ![](https://attach.52pojie.cn/forum/202001/04/185645w9czgfgdcefef8gy.png)  流程就是 3DES 的加密方式：EK3(Dk2(Ek1(P)))  ![](https://attach.52pojie.cn/forum/202001/04/185733lh99dxbnwh4rcbx9.png)  DES_Encode:  ![](https://attach.52pojie.cn/forum/202001/04/185759gj9iwpz9hnm9plio.png)  代码太长就不贴了，当时用的 IDA6.8，其反编译的还是多多少少有些坑的，现在用的 7.0 但也懒得去这部分反编译是不是一样的。注意内存结构就行，然后再用 frida 对 so 中的调用依次 hook 定位问题函数，动态调试即可。这里提一点，sub_D86C 函数中反编译的代码中有很多__PAIR__, 如果直接用网上 ida 头文件中的宏定义的话会有问题。而且这个问题并非语法问题，而是 ida 反编译得不够准确，只能通过调试或看反汇编解决  ![](https://attach.52pojie.cn/forum/202001/04/190327rmla8ea0ee0bzmeq.png)  仔细一点观察，就会发现其伪代码逻辑比较奇怪，实际汇编代码如下：  ![](https://attach.52pojie.cn/forum/202001/04/190330jc1l7n5ccwh5no7o.png)  可以将那句伪代码直接用 x86 内联汇编给替代了[Asm] _纯文本查看_ _复制代码_

```
push eax;
mov eax,r5;
sub eax,1;
sbb r5,eax;
shl r5,1;
mov eax,r5;
mov v21,eax;
pop eax;
```

其实稍作分析，其真实逻辑只是在判断 r5 是否为 0，可以将伪代码改成：v21=(BYTE2(v49)&0x20)?2:0; 到此，paysign 算出的结果总算正确了。 ![](https://attach.52pojie.cn/forum/202001/04/190930q600zo5a6rsetcvc.png)  再通过有源码的协议进行封包发送及解包，可以获取到返回的各字段[C] _纯文本查看_ _复制代码_

```
{{
  "retcode": "0",
  "retmsg": "",
  "user_name": "wxid_7vf3tr41v3g921",
  "true_name": "**鹏",
  "fee": "0",
  "desc": "",
  "scene": "32",
  "transfer_qrcode_id": "aOqTgOotZtAyyz2gsfUHWPV9hsUkxMEHkCVpM5OynlvT6Q2fy6Cwv1ffb7NLyPf9PNB-CY1mWSZW0YqQjo39TbJJWLdpPDnX2EROxb1aHTx1FKd6jqZf1wgFS98q0D32",
  "rcvr_ticket": "Y4pH5nL20VA7CcRPboeyg-4PBk3ma7_U_vksGZzWTBYE4ioVEcz_v6PrG_ZS1QtY",
  "get_pay_wifi": 1,
  "receiver_openid": "oX2-vjvhwAutxXTxz85dJeSzzG-k",
  "scan_scene": 1,
  "favor_list": [],
  "amount_remind_bit": 4
}}
```

不过 12 月开始好像就不返回 wxid 了, 也就是 user_name 返回的是空 "" ![](https://attach.52pojie.cn/forum/202001/04/190956g7q7eqcl7fffj70x.png)  类似的，在转账的时候，还有一个密码的算法，快速地说一下，还是一样的通过调用栈去找  ![](https://attach.52pojie.cn/forum/202001/04/191206w3ifwt9c109nh8ig.png)  hook com.tencent.mm.plugin.wallet.pay.a.a.b 构造  ![](https://attach.52pojie.cn/forum/202001/04/191246bjtm0j3ppzuq3ow2.png)  密码是第一个参数 authen 的成员变量  ![](https://attach.52pojie.cn/forum/202001/04/191313tkpxzkxnhz8npymg.png)  com.tencent.mm.plugin.wallet.pay.a.a.a 同理也是一样con.tencent.mm.plugin.wallet.pay.a.a.e 里有 post 地址，当然之前 paysign 里面那个 hook 也能获取到：  ![](https://attach.52pojie.cn/forum/202001/04/191428tzhbdf4pfwlv7apd.png)  再上一层调用：  ![](https://attach.52pojie.cn/forum/202001/04/191449tf1vjzwtu0vdrzhz.png)  Authen 为 cZw() 返回值  ![](https://attach.52pojie.cn/forum/202001/04/191524j3ze2mej3kzz3ipi.png)  密码通过成员变量 hef 进行赋值  ![](https://attach.52pojie.cn/forum/202001/04/192301tvnzatj4tftndnm5.png)  hef 赋值的地方  ![](https://attach.52pojie.cn/forum/202001/04/192339lzxw3w3xxx66e229.png)   ![](https://attach.52pojie.cn/forum/202001/04/192341qz5uucz7u0zhvhug.png)  再往上找密码字串  ![](https://attach.52pojie.cn/forum/202001/04/192408phgatjgra70azh0h.png)  又找到一个字段 vfo  ![](https://attach.52pojie.cn/forum/202001/04/192438ixwnvz4kejxpxvjg.png)   ![](https://attach.52pojie.cn/forum/202001/04/192440ju2klhhflfhtwf0z.png)  密码加密后字符串由 getText() 得到，具体是 com.tencent.mm.wallet_core.ui.formview.c.a.a 的返回值  ![](https://attach.52pojie.cn/forum/202001/04/192550kyqevwvb2p0vd226.png)  跟进去看到，payu 和 tenpay 有两种返回值  ![](https://attach.52pojie.cn/forum/202001/04/192637g11l7l3y1cv991v4.png)  我们好友转账，扫码转账，包括 708 新加入的手机号转账走的都是 tenpay，i 大概是触发的类型，确定交易时是 1，输入金额时是 100，其实现部分  ![](https://attach.52pojie.cn/forum/202001/04/192751f1rkfrk54aafx194.png)  当输入金额时，返回的就是输入的明文数字，其他类型会进行加密。实际上 com.tencent.mm.wallet_core.ui.formview.WalletFormView 以及 com.tencent.mm.wallet_core.ui.formview.EditHintPasswdView 的两个 getText 实现分别返回了输入的金额以及密此码明文，通过此可以修改转账金额，再将生成订单金额还原成之前的金额，即可控制实际转账金额[JavaScript] _纯文本查看_ _复制代码_

```
var WalletOpenViewProxyUI=Java.use("com.tencent.mm.wallet_core.ui.e");
var old="";
WalletOpenViewProxyUI.e.overload("double","java.lang.String").implementation=function(a1,a2){
    //var uPn=Java.cast(Authen.class,clazz).getDeclaredField("uPn");
    //uPn.setAccessible(true);
    //send("uPn:"+uPn.get(a1));
    if(a1==0.02)//显示原有金额
        a1=Number(old) ;
    var res=this.e(a1,a2);
    send("a1:"+a1.toString()+",res:"+res);
    var stack=instance.currentThread().getStackTrace();
    var full_call_stack=where(stack);
    //console.log(full_call_stack);
    return res;
}
var wwww=Java.use("com.tencent.mm.wallet_core.ui.formview.WalletFormView");
wwww.getText.implementation=function(){
    old=this.getText();
    return "0.02";//设置实际转账金额
}
```

 ![](https://attach.52pojie.cn/forum/202001/04/193712edwiwsmm800nd2ls.png) ![](https://attach.52pojie.cn/forum/202001/04/193709urk5o61ksiwld6s6.png)   
当然你有兴趣也可以把转账成功信息给改了，那么用户很可能都不清楚自己转账金额已经被篡改了，好像又扯远了。  
我们继续分析加密函数 getEncryptDataWithHash，另一个加密 get3DesEncryptData 不知是什么情况下进行的，getInputText() 能获取到密码明文，紧接着会判断 mlEncrypt 有没有实现，该成员类型是一个接口类  
 ![](https://attach.52pojie.cn/forum/202001/04/194152eyd1hhag11x7dhua.png)   
 ![](https://attach.52pojie.cn/forum/202001/04/194207p7kz7kii0gi08eiw.png)   
具体实现在 com.tenpay.android.wechat.TenpaySecureEncrypt.encryptPasswd,str 就是传入的密码明文，先计算了一下 md5。str2 传入的是时间戳，时间戳是 com.tenpay.android.wechat.TenpaySecureEditText 的 setSalt 方法设置的  
 ![](https://attach.52pojie.cn/forum/202001/04/194413lz8tu6b3zidlgtgt.png)   
接下来进入 so 层还原其算法即可，算法为 RSA2048。毕竟密码位数太短，取 md5 后也很容易被爆破，在加密之前还需要加盐，盐是时间戳以及随机数填充的。  
代码太长也就不贴了，同样在 IDA6.8 中存在一些错误，提一点 encrypt_pass1 函数中有这三个变量取了同一个地址  
 ![](https://attach.52pojie.cn/forum/202001/04/195543gviw2aof8opz8889.png)   
然而事实并非这样，通过汇编可以看到在赋值前先抬高了 sp，虽然每次都是取的 var_6C 位置，但前后栈顶已经不一样了：  
 ![](https://attach.52pojie.cn/forum/202001/04/195650qovp1ivo2z317233.png)   
也就是说只要操作 sp 的地方，伪代码都是有些问题的，比如这个 else 里面的 v9=&res,res 是传入的参数，这种逻辑一看就是有问题的：  
 ![](https://attach.52pojie.cn/forum/202001/04/195812lj158ya295s0avjb.png)   
通过反汇编看到这里也是通过 sp 去索引的变量，实际上 sp 已经抬高了三次，这里真实的索引不是参数 res，而是在局部变量 s 中：  
 ![](https://attach.52pojie.cn/forum/202001/04/195908dtwdvebtv2zwimwd.png)   
密码加密算法还原后我们就能模拟其支付协议了。不管是好友转账，扫码转账或者手机号转账最后都是需要 req_key 的，差不多就是个订单号的意思  
 ![](https://attach.52pojie.cn/forum/202001/04/200654wl4f4pku8zd3fsvv.png)   
好友转账可以通过 CGI_TENPAY 直接传对方 wxid 然后返回 req_key，扫码比较麻烦些，之前 WCPaySign 那步获取到 openid,ticket,qrcode_id 等字段，再通过这些字段去生成订单走 / cgi-bin/mmpay-bin/busif2fplaceorder 获取到 req_key。但在 12 月之前，扫码仍有 wxid 返回时，可以投机走旧版的协议 / cgi-bin/micromsg-bin/tenpay，逻辑与之前 setRequestData 找的一样，大概就是将 map 中元素拼接成字符串，最后再计算一个 paysign，而 map 中元素也就是扫码返回的数据 ![](https://attach.52pojie.cn/forum/202001/04/202235mh7wz997b8hwowad.png)   
 ![](https://attach.52pojie.cn/forum/202001/04/202359arcgf2gc22fimfkh.png) 获取到 req_key, 最终完成转账操作，bank_type 和 bind_serial 是绑定银行卡的类型 id，都为 CFT 时使用零钱，获取 bind_serial 也很简单，这里就不讨论了。  ![](https://attach.52pojie.cn/forum/202001/04/202506hktskgh8y15g533k.png)  708 新增的通过手机号转账也是分为三步，通过 / cgi-bin/mmpay-bin/transferphonegetrcvr 获取到 openid 等信息，再通过 / cgi-bin/mmpay-bin/transferphoneplaceorder 生成订单，获取到 req_key，最后再由 / cgi-bin/mmpay-bin/tenpay/sns_tf_authen 确认订单完成转账。组包中的金额好像进行了一种序列化之类的操作，但还是很容易看得出来的，其算法我们可以自己实现：[C#] _纯文本查看_ _复制代码_

```
private int Pow(int x, int n)
{
    int res = 1;
    while (n > 0)
    {
        if ((n & 1) == 1) res = res * x;//转化为二进制
        x = x * x;//将x平方
        n >>= 1;
    }
    return res;
}
public string getFee(int money,bool isfirst=false,int sign=-1)
{
    if (money < 0x80 && isfirst == true)
        return String.Format("{0:X2}", money);
    int i = 0;
    int temp = (int)money;
    while ((temp /= 0x80)>=1)
        i++;//递归次数
    if (isfirst == true) sign = i;
    int pow= Pow(128, i);//128 i次方
    int dwRes = (pow==1)? money:money/pow;
    money = (pow == 1) ?0: money-dwRes * pow;
    if (isfirst == false) dwRes += 0x80;
    string res = String.Format("{0:X2}", dwRes);
    return sign == 0?res: getFee(money, false, --sign)+res;
}
```

手机转账其实并没有分析很多，很多数据没有分析，只是写死的，但也是能实现手机转账的功能。太晚了不写了，其实也就初探了下微信支付的流程，加密的算法。当然后续还需要进一步的封包压缩加密，但这都有现成的协议代码。虽然 wx 是一款社交软件，但其加密强度目前来看也是很高的，但在本地上，我们仍能做很多有趣的事情，所以建议大家不要使用 wx 模块之类的插件  
  

![](https://attach.52pojie.cn/forum/202001/04/173720q3j63xx0apvvv618.png)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)xiong1992 是不是可以理解为我买了东西当对方面去扫对方的微 X 二维码，显示支付成功后结果却是我给我的另外一个号转钱过去了；或者是 2 元的东西，显示成功支付 2 元后，实际上我只支付了 1 元；或者是别人给我转账，转 1 元，显示成功支付 1 元，但是实际上我却收到了他转的 2 元。这个如果给不法分子利用就麻烦了。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)努力的 T 先生 看不懂，不知道大神能不能给个变得跟你一样优秀的思路，至少把这个看得懂 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) TedChen ![](https://static.52pojie.cn/static/image/smiley/default/17.gif)![](https://static.52pojie.cn/static/image/smiley/default/17.gif)有点担心我的十几块钱了![](https://avatar.52pojie.cn/images/noavatar_middle.gif)股票亏损员 感觉看到是天书 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) nshark 看着都迷糊![](https://static.52pojie.cn/static/image/smiley/default/42.gif)![](https://avatar.52pojie.cn/data/avatar/001/03/30/62_avatar_middle.jpg)三年三班三井寿

> [老哥还会军体拳 发表于 2020-1-7 18:13](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=29432849&ptid=1085646)  
> 老哥，漏洞搞得怎么样了

刚接触没多久 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) ccc800 大神。路过 看的眼花 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) linfengtai2008 感谢，学习了！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)冷月残星 天书一本 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 88491354@qq.com 厉害，学习了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) winson365 大神。。。 ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEAAAABACAMAAACdt4HsAAAAOVBMVEX///+AgICfn5+Hh4f39/fPz8+vr6+QkJDg4ODv7+/n5+fAwMCoqKiYmJjHx8fX19e4uLjQ0NDIyMhhw/XSAAACBUlEQVRYw6VX2ZKEIAw0ATnk8Pj/j90q3V13SSOxpl+mHEknaUKCUw9xTZnpBLu02ukV4lGoAR9Rbb4Xgii7ztxTF35/ay4p1kdzm2mI/KDn4kkB0w3iICVmbO9IDae31zMkoo8YZkJgl5IrGh1WpHaN34VdDUks//YfrNjsn/cO8NtnAedhivl+G6D9kCH8Buhl/JptLv0dLMIcuqnTBY8TGCdhelvohfGjI3mEEybYZKGdZTKokcfNipCXpg4IysgoMAwDl5KegFGwy2cEYZpfEBQkAjpnehFpQ2HRCwKeDKHdhQiQgGBcEE5NgEWwhgAm/eSo2BchmAXMPUMwBSaEYlE/0RPIGWxLb8BkwiiLam6n/kgzx+3+IOoSBGSbr5+0nN43cz0SnE9Wpr9O0/qz2psf0jCtTrqSR8zFK2lu1D5FjZtpnLX1wfGu/Pxn3e8QillUXGxa3A07J8fimjqb9tixrD8JXJJ8UQp7NcN6n7LrScvQTLAqxtEItTm0/r5wKOH/j9AgutAIc5MzE4VXBLa56UUjOsAwB2+bhuteprCIf9wr+yp7tl5H7C0TzW/tZQwaJQ+i2mX2w92M/BBpMEMhkiGzPtOXhyBCGaYZPFHBH6l2z90cRcfh3QJr8lWjsq1nn2Xe0hHCEvaUmL/btRaxyo/vFKdXiCFlPmk8u9rvdl/WWxJO0oeE7gAAAABJRU5ErkJggg==) 大哥厉害，之后买兰博基尼就靠你了。&#128516; ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEAAAABACAMAAACdt4HsAAAAOVBMVEX///+AgICfn5+Hh4f39/fPz8+vr6+QkJDg4ODv7+/n5+fAwMCoqKiYmJjHx8fX19e4uLjQ0NDIyMhhw/XSAAACBUlEQVRYw6VX2ZKEIAw0ATnk8Pj/j90q3V13SSOxpl+mHEknaUKCUw9xTZnpBLu02ukV4lGoAR9Rbb4Xgii7ztxTF35/ay4p1kdzm2mI/KDn4kkB0w3iICVmbO9IDae31zMkoo8YZkJgl5IrGh1WpHaN34VdDUks//YfrNjsn/cO8NtnAedhivl+G6D9kCH8Buhl/JptLv0dLMIcuqnTBY8TGCdhelvohfGjI3mEEybYZKGdZTKokcfNipCXpg4IysgoMAwDl5KegFGwy2cEYZpfEBQkAjpnehFpQ2HRCwKeDKHdhQiQgGBcEE5NgEWwhgAm/eSo2BchmAXMPUMwBSaEYlE/0RPIGWxLb8BkwiiLam6n/kgzx+3+IOoSBGSbr5+0nN43cz0SnE9Wpr9O0/qz2psf0jCtTrqSR8zFK2lu1D5FjZtpnLX1wfGu/Pxn3e8QillUXGxa3A07J8fimjqb9tixrD8JXJJ8UQp7NcN6n7LrScvQTLAqxtEItTm0/r5wKOH/j9AgutAIc5MzE4VXBLa56UUjOsAwB2+bhuteprCIf9wr+yp7tl5H7C0TzW/tZQwaJQ+i2mX2w92M/BBpMEMhkiGzPtOXhyBCGaYZPFHBH6l2z90cRcfh3QJr8lWjsq1nn2Xe0hHCEvaUmL/btRaxyo/vFKdXiCFlPmk8u9rvdl/WWxJO0oeE7gAAAABJRU5ErkJggg==) 大神, 望尘莫及啊