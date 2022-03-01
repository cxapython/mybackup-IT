> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [onejane.github.io](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/#%E6%8A%93%E5%8C%8510-37-13)

> SO 逆向之大众点评 cx

打开首页一篇文章，APP 默认 TCP 连接，通过降级采用 HTTP 连接

jadx 反编译代码中

plain

```
public int g() {
    Object[] objArr = new Object[0];
    ChangeQuickRedirect changeQuickRedirect = a;
    if (PatchProxy.isSupport(objArr, this, changeQuickRedirect, false, "18c862d70fb657f32643dbd9124704f9", RobustBitConfig.DEFAULT_VALUE)) {
        return ((Integer) PatchProxy.accessDispatch(objArr, this, changeQuickRedirect, false, "18c862d70fb657f32643dbd9124704f9")).intValue();
    }
    if ("cip".equals(this.d)) {
        return 2;
    }
    if ("http".equals(this.d)) {
        return 3;
    }
    if ("wns".equals(this.d)) {
        return 4;
    }
    return 2;
}
```

绕过 CIP 和 WNS 代理，直接走 HTTP 通道

plain

```
var nvnetwork_g = Java.use("com.dianping.nvnetwork.g");
nvnetwork_g.g.overload().implementation = function(){
    console.log("----------------------------- Hook g()---------------------------");
    return 3;
}
var URL = Java.use('java.net.URL');
URL.openConnection.overload().implementation = function () {
    var retval = this.openConnection();
    if (retval.toString().indexOf("getfeedcontent.bin") >= 0) {
        console.log('URL openConnection' + retval);
        var stack = threadinstance.currentThread().getStackTrace();
        console.log(">>> openConnection Full call stack:" + Where(stack));
    }
    return retval;
};
```

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220221222700367.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220221222700367.png)

[image-20220221222700367](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220221222700367.png)

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220221223510006.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220221223510006.png)

[image-20220221223510006](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220221223510006.png)

[https://mapi.dianping.com/mapi/note/getfeedcontent.bin](https://mapi.dianping.com/mapi/note/getfeedcontent.bin)

<table><thead><tr><th>mainid</th><th>1022239297</th></tr></thead><tbody><tr><td>feedtype</td><td>1</td></tr><tr><td>lng</td><td>120.657842</td></tr><tr><td>lat</td><td>31.247756</td></tr><tr><td>displaypattern</td><td>1</td></tr><tr><td>firstimagekey</td><td>354e9fb1c37f37e0545c4920b2836838</td></tr><tr><td>bubblepagetype</td><td>null</td></tr><tr><td>cx</td><td>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</td></tr><tr><td>pagecityid</td><td>6</td></tr><tr><td>optimus_partner</td><td>76</td></tr><tr><td>optimus_risk_level</td><td>71</td></tr><tr><td>optimus_code</td><td>10</td></tr><tr><td>picsize</td><td>{“single”:{“width”:900,”height”:900},”many”:{“width”:500,”height”:500},”glance”:{“width”:1440,”minHeight”:810,”maxHeight”:1920}}</td></tr><tr><td>pagesource</td><td>1</td></tr><tr><td>picscale</td><td>0.749651385199241</td></tr></tbody></table>

返回数据加密，但是展示时正常文字展示，说明数据在客户端被解密，monitor.bat 录制轨迹

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012405116.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012405116.png)

[image-20220222012405116](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012405116.png)

plain

```
var MapiModuleCls = Java.use('com.dianping.picasso.commonbridge.MapiModule');
MapiModuleCls.resolveData.overload('com.dianping.archive.DPObject', 'boolean', 'int').implementation = function (a, b, c) {
    var result = this.resolveData(a, b, c);
    console.log("resolveData rc=" + result+ "===boolean===" +b+"===int===" +c);
    var stack = threadinstance.currentThread().getStackTrace();
    console.log(">>> resolveData Full call stack:" + Where(stack));
    return result;
}
```

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012231767.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012231767.png)

[image-20220222012231767](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012231767.png)

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012835605.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012835605.png)

[image-20220222012835605](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012835605.png)

上图类 MapiModule 中的 fetch 和 fetchV2 方法中的`innerFetch`方法中`exec(this.host.getContext().getApplicationContext(), bVar2, bVar, z, hashMap, optInt2);`的 exec 方法中调用了 `MapiModule.this.resolveData((DPObject) gVar.b(), z, i)`

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012612213.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012612213.png)

[image-20220222012612213](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222012612213.png)

cx 字段里面有两个比较明显的特征，大量含有 + / 这两个字符串，末尾是 ==，断定，这个 cx 字段是 Base64 字串。

plain

```
var Base64Class = Java.use("android.util.Base64");
Base64Class.encodeToString.overload('[B', 'int').implementation = function (a, b) {
    var result = this.encodeToString(a, b);

    if (result.indexOf("d41bgtH") >= 0) {
        console.log("Base64 1 rc=" + result);
        var stack = threadinstance.currentThread().getStackTrace();
        console.log(">>> Base64 Full call stack:" + Where(stack));
    }

    return result;
}

Base64Class.encodeToString.overload('[B', 'int', 'int', 'int').implementation = function (a, b, c, d) {
    var result = this.encodeToString(a, b, c, d);
    console.log("Base64 2 rc=" + result);
    return result;
}
```

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014442156.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014442156.png)

[image-20220222014442156](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014442156.png)

jadx 直接查看`com.meituan.android.common.fingerprint.encrypt.DESHelper.encryptByPublic`

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014633708.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014633708.png)

[image-20220222014633708](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014633708.png)

plain

```
var fingerprintCls = Java.use('com.meituan.android.common.fingerprint.encrypt.DESHelper');
fingerprintCls.encryptByPublic.implementation = function(a,b){
	var result = this.encryptByPublic(a,b);
	console.log('encryptByPublic inbuf= '+ a +',key='+b +',outBuf=' + result);
	return result;
}
```

[![](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014747432.png)](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014747432.png)

[image-20220222014747432](https://onejane.github.io/2022/02/07/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%A4%A7%E4%BC%97%E7%82%B9%E8%AF%84cx/image-20220222014747432.png)

拿到 key 和 strInBuf，即 dESKeySpec 和 secretKey

plain

```
import java.security.Key;
import java.util.Base64;
import javax.crypto.Cipher;
import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.DESKeySpec;
import javax.crypto.spec.IvParameterSpec;

public class Main {

    public static void main(String[] args) {
        String strInBuf = "d41bgtHhx7iFJPuvzpwy...9jFK8DgJ63ysfpx6KvdxodXzD7cg+sAikWZ1/iX1I0P/XNuAAj0/QApN25mtLQ==";
        String key = "kwBq8snI"; 
        String strOut = "";

        try {

            byte[] base64decodedBytes = Base64.getDecoder().decode(strInBuf);

            DESKeySpec dESKeySpec;
            SecretKey secretKey = null;
            dESKeySpec = new DESKeySpec(key.getBytes());
            secretKey = SecretKeyFactory.getInstance("DES").generateSecret(dESKeySpec);

            Cipher instance = Cipher.getInstance("DES/CBC/PKCS5Padding");
            instance.init(2, secretKey, new IvParameterSpec(dESKeySpec.getKey()));
            strOut = new String(instance.doFinal(base64decodedBytes));
            
            /* com.dianping.nvnetwork.tunnel.tool.c 的加密数据 解密
            strInBuf = "EcFKDV1V/3X2+rkklRGVjOnl9sRyHe3e+4OFRvluXA+R+1lns804Ew1STfKND1mPA7mQa8cOm0sn3zHuA6Kp58lSRmaeHlH4mlpg3YzWyPDtfOChpOwC4RjRKXargIjpCDnol++yP92Y2L8fCRDL9LtX5KJQDiQa";
            key =""1nzbcvFS1LCtoNHr";

            Key a = SecretKeyFactory.getInstance("DES").generateSecret(new DESKeySpec(key.getBytes()));
            Cipher instance = Cipher.getInstance("DES");
            // Cipher.ENCRYPT_MODE 1 加密
            // Cipher.DECRYPT_MODE 2 解密
            instance.init(2, a);
            strOut = new String(instance.doFinal(base64decodedBytes));
            */

        } catch (Exception e) {
            e.printStackTrace();
        } 
        System.out.println(strOut);
    }
}
```

版权声明: 本博客所有文章除特别声明外，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。转载请注明来自 [万物皆可逆向](http://onejane.github.io/)！

![](http://onejane.gitee.io/picture/alipay.png)

支付宝打赏

![](http://onejane.gitee.io/picture/wx.png)

微信打赏