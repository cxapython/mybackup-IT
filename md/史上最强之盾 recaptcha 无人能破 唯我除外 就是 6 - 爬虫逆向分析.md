> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cydata.fun](https://cydata.fun/archives/1716432526617)

> No 1: 那个柔和阳光的下午，领导递给我一个网站链接，却在我心头掀起了波涛汹涌的不安。

No 1:

**那个柔和阳光的下午，领导递给我一个网站链接，却在我心头掀起了波涛汹涌的不安。我盯着那串字符，心中顿时涌起一阵莫名的忐忑不安，就像在荒野中孤身前行，前路未知而危险。我深吸一口气，怀着满心的不安与紧张，慢慢地打开了链接。屏幕亮起的瞬间，映入眼帘的竟然是它，那个传说中无比强大的存在，史上最强之盾——reCAPTCHA。看到它的一刹那，我的心跳仿佛都停止了，这个曾经让我无数次卡在验证上的 “敌人”，如今又出现在我面前。顿时，一股无力感席卷全身，但我知道，必须迎难而上。**

```
console.info(atob('aHR0cHM6Ly93d3cubXBzLmdvdi5jbi9uMjI1NDA5OC9uNDkwNDM1Mi9pbmRleF83NTc0NjExXzI1My5odG1s'))
```

![](https://cydata.fun/upload/image-jomf.png)

No 2:

**reCAPTCHA 是 Google 提供的一项安全服务，旨在保护网站免受垃圾邮件和滥用行为的侵害，同时确保用户体验的顺畅。以下是关于 reCAPTCHA 的详细介绍：**

### 基本功能

1. **区分人类和机器人**：

**reCAPTCHA 的主要功能是区分真实用户和自动化的机器人程序，防止恶意软件进行自动化攻击，如提交垃圾评论、注册虚假账户等。**

2. **易用性**：

**reCAPTCHA 的设计目的是在提供安全保护的同时，尽量减少对真实用户的干扰。它通过简单的挑战或无缝的背景分析来确认用户身份。**

### 主要版本

1. **reCAPTCHA v1**：

**最早版本，要求用户输入图片中的扭曲文本，以验证他们是人类。这种方式在提供基本保护的同时，用户体验较差。**

2. **reCAPTCHA v2**：

**提供了更好的用户体验，通常要求用户勾选一个 “我不是机器人” 的复选框。如果系统无法确定用户身份，可能会进一步要求用户识别图片中的物体（如交通灯、汽车等）。**

3. **reCAPTCHA v3**：

**无需用户互动，完全在后台运行，通过分析用户在网站上的行为来评分（0.0 到 1.0），判断其是否为机器人。站点可以根据评分设置不同的安全策略。**

**### 使用场景**

1. **表单提交保护**：

**防止恶意软件自动提交表单，保护注册、登录、留言和反馈等功能。**

2. **防止刷票**：

**在投票系统中防止自动化刷票行为，确保投票结果的公平性。**

3. **防止账号盗用**：

**防止恶意软件尝试破解用户账户密码，保护用户账户的安全。**

4. **防止垃圾评论**：

**保护博客、论坛等社区平台免受垃圾评论的困扰，提升内容质量。**

**### 优势**

1. **安全性高**：

**通过不断改进的算法和多层次的挑战，reCAPTCHA 可以有效抵御各种自动化攻击。**

2. **易于集成**：

**提供了简单的 API 和丰富的文档，开发者可以轻松将 reCAPTCHA 集成到自己的网站或应用中。**

3. **用户友好**：

**尤其是 reCAPTCHA v3，可以在不打扰用户的情况下进行验证，提升用户体验。**

**### 总结**

**reCAPTCHA 通过多种验证方式和算法，为网站提供了强大的保护措施，防止自动化攻击和滥用行为，同时注重用户体验。它广泛应用于各种在线服务和应用中，是当前保护网站安全的主流解决方案之一。**

No 3:

**随后我对目标地址做了一次请求：**

**第一次 状态码 521 返回了一些 js 代码：**

![](https://cydata.fun/upload/image-gpnl.png)

**执行一下以上返回的 js 会生成类似 cookie：**

```
__jsl_clearance_s=1716434516.578|-1|LwCnx0wpi%2FxW4qmiew%2FeLT1V1IE%3D; Max-age=3600; Path=/; SameSite=None; Secure
```

**第二次 拿着第一次的 cookie 取请求 返回以下 js 对以下 js 做处理：**

![](https://cydata.fun/upload/image-iflh.png)

**第三次 那前面生成的 cookie 请求目标地址状态吗 200 就 ok 了：**

![](https://cydata.fun/upload/image-gygv.png)

No 4:

**demo 代码如下：**

```
import randomimport reimport demjson3import execjsimport requestsimport urllib3from requests.adapters import HTTPAdapterfrom requests.utils import dict_from_cookiejarfrom urllib3.util.ssl_ import create_urllib3_context urllib3.disable_warnings()ORIGIN_CIPHERS = (    'ECDHE-ECDSA-AES256-GCM-SHA384:'    'ECDHE-ECDSA-AES128-GCM-SHA256:'    'AES256-GCM-SHA384:'    'AES128-GCM-SHA256:'    'AES256-SHA256:'    'AES128-SHA256:'    'AES256-SHA:'    'AES128-SHA:'    'DES-CBC3-SHA')  class DESAdapter(HTTPAdapter):    def __init__(self, *args, **kwargs):        CIPHERS = ORIGIN_CIPHERS.split(':')        random.shuffle(CIPHERS)        CIPHERS = ':'.join(CIPHERS)        self.CIPHERS = CIPHERS + ':!aNULL:!eNULL:!MD5'        super().__init__(*args, **kwargs)     def init_poolmanager(self, *args, **kwargs):        context = create_urllib3_context(ciphers=self.CIPHERS)        kwargs['ssl_context'] = context        return super(DESAdapter, self).init_poolmanager(*args, **kwargs)     def proxy_manager_for(self, *args, **kwargs):        context = create_urllib3_context(ciphers=self.CIPHERS)        kwargs['ssl_context'] = context        return super(DESAdapter, self).proxy_manager_for(*args, **kwargs)  def create_session():    session = requests.Session()    session.mount('https://www.xxx.xxx.xxx', DESAdapter())    return session  def get_initial_response(session, url, headers):    return session.get(url, headers=headers, verify=False)  def extract_cookie(response):    ck2_script = response.text.split('document.cookie=')[1].split(';location.href')[0]    return execjs.eval(ck2_script).split(';max-age=3600;path=/')[0]  def compile_js(filename):    with open(filename, encoding='utf-8') as f:        return execjs.compile(f.read())  def update_headers_with_cookie(headers, cookie):    headers.update({'Cookie': cookie})    return headers  def get_encrypted_cookie(jsl, data):    if data['ha'] == 'sha1':        return jsl.call('jsl_sha1', data).split(';Max-age=3600; path = /')[0]    elif data['ha'] == 'sha256':        return jsl.call('jsl_sha256', data).split(';Max-age=3600; path = /')[0]    elif data['ha'] == 'md5':        return jsl.call('jsl_md5', data).split(';Max-age=3600; path = /')[0]    else:        raise ValueError('Unknown encryption method')  def main(url, headers, js_file):    session = create_session()    initial_response = get_initial_response(session, url, headers)    ck2 = extract_cookie(initial_response)    jsl = compile_js(js_file)    ck1 = dict_from_cookiejar(initial_response.cookies)    cookie = '__jsluid_s=' + ck1['__jsluid_s'] + ';' + ck2     headers = update_headers_with_cookie(headers, cookie)    response = session.get(url, headers=headers)    data = demjson3.decode(re.search(';go\((.*?)\)</script>', response.text, re.S).group(1))    encrypted_cookie = get_encrypted_cookie(jsl, data)    cookie = '__jsluid_s=' + ck1['__jsluid_s'] + ';' + encrypted_cookie     headers = update_headers_with_cookie(headers, cookie)    final_response = session.get(url, headers=headers)    final_response.encoding = final_response.apparent_encoding     print(final_response.status_code)    print(final_response.text)  if __name__ == "__main__":    url = "https://www.xxx.xxx.xxx/n2254098/n4904352/index_7574611_253.html"    headers = {        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',        'Accept-Encoding': 'gzip, deflate, br',        'Accept-Language': 'zh-CN,zh;q=0.9',        'Host': 'www.xxx.xxx.xxx',        'Upgrade-Insecure-Requests': '1',        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36'    }    js_file = './jsl.js'    main(url, headers, js_file)
```

```
// jsl.jsfunction jsl_md5(r) {  function n(r) {    function n(r, n) {      return r << n | r >>> 32 - n;    }    function t(r, n) {      var t;      var a;      var e;      var o;      var u;      e = 2147483648 & r;      o = 2147483648 & n;      u = (1073741823 & r) + (1073741823 & n);      if ((t = 1073741824 & r) & (a = 1073741824 & n)) {        return 2147483648 ^ u ^ e ^ o;      } else {        if (t | a) {          if (1073741824 & u) {            return 3221225472 ^ u ^ e ^ o;          } else {            return 1073741824 ^ u ^ e ^ o;          }        } else {          return u ^ e ^ o;        }      }    }    function a(r, a, e, o, u, f, c) {      var i;      return t(n(r = t(r, t(t((i = a) & e | ~i & o, u), c)), f), a);    }    function e(r, a, e, o, u, f, c) {      var i;      return t(n(r = t(r, t(t(a & (i = o) | e & ~i, u), c)), f), a);    }    function o(r, a, e, o, u, f, c) {      return t(n(r = t(r, t(t(a ^ e ^ o, u), c)), f), a);    }    function u(r, a, e, o, u, f, c) {      return t(n(r = t(r, t(t(e ^ (a | ~o), u), c)), f), a);    }    function f(r) {      var n;      var t = "";      var a = "";      for (n = 0; n <= 3; n++) {        t += (a = "0" + (r >>> 8 * n & 255).toString(16)).substr(a.length - 2, 2);      }      return t;    }    var c;    var i;    var h;    var v;    var s;    var g;    var l;    var C;    var A;    var d = Array();    d = function (r) {      t = r.length;      a = t + 8;      e = 16 * ((a - a % 64) / 64 + 1);      o = Array(e - 1);      u = 0;      f = 0;      for (void 0; f < t;) {        var n;        var t;        var a;        var e;        var o;        var u;        var f;        u = f % 4 * 8;        o[n = (f - f % 4) / 4] = o[n] | r.charCodeAt(f) << u;        f++;      }      u = f % 4 * 8;      o[n = (f - f % 4) / 4] = o[n] | 128 << u;      o[e - 2] = t << 3;      o[e - 1] = t >>> 29;      return o;    }(r);    g = 1732584193;    l = 4023233417;    C = 2562383102;    A = 271733878;    for (c = 0; c < d.length; c += 16) {      i = g;      h = l;      v = C;      s = A;      l = u(l = u(l = u(l = u(l = o(l = o(l = o(l = o(l = e(l = e(l = e(l = e(l = a(l = a(l = a(l = a(l, C = a(C, A = a(A, g = a(g, l, C, A, d[c + 0], 7, 3614090360), l, C, d[c + 1], 12, 3905402710), g, l, d[c + 2], 17, 606105819), A, g, d[c + 3], 22, 3250441966), C = a(C, A = a(A, g = a(g, l, C, A, d[c + 4], 7, 4118548399), l, C, d[c + 5], 12, 1200080426), g, l, d[c + 6], 17, 2821735955), A, g, d[c + 7], 22, 4249261313), C = a(C, A = a(A, g = a(g, l, C, A, d[c + 8], 7, 1770035416), l, C, d[c + 9], 12, 2336552879), g, l, d[c + 10], 17, 4294925233), A, g, d[c + 11], 22, 2304563134), C = a(C, A = a(A, g = a(g, l, C, A, d[c + 12], 7, 1804603682), l, C, d[c + 13], 12, 4254626195), g, l, d[c + 14], 17, 2792965006), A, g, d[c + 15], 22, 1236535329), C = e(C, A = e(A, g = e(g, l, C, A, d[c + 1], 5, 4129170786), l, C, d[c + 6], 9, 3225465664), g, l, d[c + 11], 14, 643717713), A, g, d[c + 0], 20, 3921069994), C = e(C, A = e(A, g = e(g, l, C, A, d[c + 5], 5, 3593408605), l, C, d[c + 10], 9, 38016083), g, l, d[c + 15], 14, 3634488961), A, g, d[c + 4], 20, 3889429448), C = e(C, A = e(A, g = e(g, l, C, A, d[c + 9], 5, 568446438), l, C, d[c + 14], 9, 3275163606), g, l, d[c + 3], 14, 4107603335), A, g, d[c + 8], 20, 1163531501), C = e(C, A = e(A, g = e(g, l, C, A, d[c + 13], 5, 2850285829), l, C, d[c + 2], 9, 4243563512), g, l, d[c + 7], 14, 1735328473), A, g, d[c + 12], 20, 2368359562), C = o(C, A = o(A, g = o(g, l, C, A, d[c + 5], 4, 4294588738), l, C, d[c + 8], 11, 2272392833), g, l, d[c + 11], 16, 1839030562), A, g, d[c + 14], 23, 4259657740), C = o(C, A = o(A, g = o(g, l, C, A, d[c + 1], 4, 2763975236), l, C, d[c + 4], 11, 1272893353), g, l, d[c + 7], 16, 4139469664), A, g, d[c + 10], 23, 3200236656), C = o(C, A = o(A, g = o(g, l, C, A, d[c + 13], 4, 681279174), l, C, d[c + 0], 11, 3936430074), g, l, d[c + 3], 16, 3572445317), A, g, d[c + 6], 23, 76029189), C = o(C, A = o(A, g = o(g, l, C, A, d[c + 9], 4, 3654602809), l, C, d[c + 12], 11, 3873151461), g, l, d[c + 15], 16, 530742520), A, g, d[c + 2], 23, 3299628645), C = u(C, A = u(A, g = u(g, l, C, A, d[c + 0], 6, 4096336452), l, C, d[c + 7], 10, 1126891415), g, l, d[c + 14], 15, 2878612391), A, g, d[c + 5], 21, 4237533241), C = u(C, A = u(A, g = u(g, l, C, A, d[c + 12], 6, 1700485571), l, C, d[c + 3], 10, 2399980690), g, l, d[c + 10], 15, 4293915773), A, g, d[c + 1], 21, 2240044497), C = u(C, A = u(A, g = u(g, l, C, A, d[c + 8], 6, 1873313359), l, C, d[c + 15], 10, 4264355552), g, l, d[c + 6], 15, 2734768916), A, g, d[c + 13], 21, 1309151649), C = u(C, A = u(A, g = u(g, l, C, A, d[c + 4], 6, 4149444226), l, C, d[c + 11], 10, 3174756917), g, l, d[c + 2], 15, 718787259), A, g, d[c + 9], 21, 3951481745);      g = t(g, i);      l = t(l, h);      C = t(C, v);      A = t(A, s);    }    return (f(g) + f(l) + f(C) + f(A)).toLowerCase();  }  var t = new Date();  var a = function (a, e) {    o = r.chars.length;    u = 0;    for (void 0; u < o; u++) {      var o;      var u;      for (var f = 0; f < o; f++) {        var c = e[0] + r.chars.substr(u, 1) + r.chars.substr(f, 1) + e[1];        if (n(c) == a) {          return [c, new Date() - t];        }      }    }  }(r.ct, r.bts);  cookie = r.tn + "=" + a[0] + ";Max-age=" + r.vt + "; path = /";  return cookie;}function jsl_sha1(r) {  function n(r) {    function n(r, n) {      return (2147483647 & r) + (2147483647 & n) ^ 2147483648 & r ^ 2147483648 & n;    }    function a(r) {      n = "";      t = 7;      for (void 0; t >= 0; t--) {        var n;        var t;        n += "0123456789abcdef".charAt(r >> 4 * t & 15);      }      return n;    }    function e(r, n) {      return r << n | r >>> 32 - n;    }    h = function (r) {      n = 1 + (r.length + 8 >> 6);      t = new Array(16 * n);      a = 0;      for (void 0; a < 16 * n; a++) {        var n;        var t;        var a;        t[a] = 0;      }      for (a = 0; a < r.length; a++) {        t[a >> 2] |= r.charCodeAt(a) << 24 - 8 * (3 & a);      }      t[a >> 2] |= 128 << 24 - 8 * (3 & a);      t[16 * n - 1] = 8 * r.length;      return t;    }(r);    v = new Array(80);    s = 1732584193;    g = -271733879;    l = -1732584194;    C = 271733878;    A = -1009589776;    d = 0;    for (void 0; d < h.length; d += 16) {      var o;      var u;      var f;      var c;      var i;      var h;      var v;      var s;      var g;      var l;      var C;      var A;      var d;      w = s;      b = g;      y = l;      m = C;      D = A;      S = 0;      for (void 0; S < 80; S++) {        var w;        var b;        var y;        var m;        var D;        var S;        v[S] = S < 16 ? h[d + S] : e(v[S - 3] ^ v[S - 8] ^ v[S - 14] ^ v[S - 16], 1);        var mint1716432199483000 = e(s, 5);        var mint1716432199481000 = n(mint1716432199483000, (f = g, c = l, i = C, (u = S) < 20 ? f & c | ~f & i : u < 40 ? f ^ c ^ i : u < 60 ? f & c | f & i | c & i : f ^ c ^ i));        t = n(mint1716432199481000, n(n(A, v[S]), (o = S) < 20 ? 1518500249 : o < 40 ? 1859775393 : o < 60 ? -1894007588 : -899497514));        A = C;        C = l;        l = e(g, 30);        g = s;        s = t;      }      s = n(s, w);      g = n(g, b);      l = n(l, y);      C = n(C, m);      A = n(A, D);    }    return a(s) + a(g) + a(l) + a(C) + a(A);  }  var a = new Date();  var e = function (t, e) {    o = r.chars.length;    u = 0;    for (void 0; u < o; u++) {      var o;      var u;      for (var f = 0; f < o; f++) {        var c = e[0] + r.chars.substr(u, 1) + r.chars.substr(f, 1) + e[1];        if (n(c) == t) {          return [c, new Date() - a];        }      }    }  }(r.ct, r.bts);  cookie = r.tn + "=" + e[0] + ";Max-age=" + r.vt + "; path = /";  return cookie;}function jsl_sha256(r) {  var n = new Date();  var t = function (t, e) {    o = r.chars.length;    u = 0;    for (void 0; u < o; u++) {      var o;      var u;      for (var f = 0; f < o; f++) {        var c = e[0] + r.chars.substr(u, 1) + r.chars.substr(f, 1) + e[1];        if (a(c) == t) {          return [c, new Date() - n];        }      }    }  }(r.ct, r.bts);  function a(r) {    var n = 8;    var t = 0;    function a(r, n) {      var t = (65535 & r) + (65535 & n);      return (r >> 16) + (n >> 16) + (t >> 16) << 16 | 65535 & t;    }    function e(r, n) {      return r >>> n | r << 32 - n;    }    return function (r) {      var n;      n = t ? "0123456789ABCDEF" : "0123456789abcdef";      a = "";      e = 0;      for (void 0; e < 4 * r.length; e++) {        var a;        var e;        a += n.charAt(r[e >> 2] >> 8 * (3 - e % 4) + 4 & 15) + n.charAt(r[e >> 2] >> 8 * (3 - e % 4) & 15);      }      return a;    }(function (r, n) {      var t;      var o;      var u;      var f;      var c;      var i;      var h;      var v;      var s;      var g;      var l;      var C;      var A;      var d;      var w;      var b;      var y;      var m;      var D = new Array(1116352408, 1899447441, 3049323471, 3921009573, 961987163, 1508970993, 2453635748, 2870763221, 3624381080, 310598401, 607225278, 1426881987, 1925078388, 2162078206, 2614888103, 3248222580, 3835390401, 4022224774, 264347078, 604807628, 770255983, 1249150122, 1555081692, 1996064986, 2554220882, 2821834349, 2952996808, 3210313671, 3336571891, 3584528711, 113926993, 338241895, 666307205, 773529912, 1294757372, 1396182291, 1695183700, 1986661051, 2177026350, 2456956037, 2730485921, 2820302411, 3259730800, 3345764771, 3516065817, 3600352804, 4094571909, 275423344, 430227734, 506948616, 659060556, 883997877, 958139571, 1322822218, 1537002063, 1747873779, 1955562222, 2024104815, 2227730452, 2361852424, 2428436474, 2756734187, 3204031479, 3329325298);      var S = new Array(1779033703, 3144134277, 1013904242, 2773480762, 1359893119, 2600822924, 528734635, 1541459225);      var k = new Array(64);      r[n >> 5] |= 128 << 24 - n % 32;      r[15 + (n + 64 >> 9 << 4)] = n;      for (var p = 0; p < r.length; p += 16) {        t = S[0];        o = S[1];        u = S[2];        f = S[3];        c = S[4];        i = S[5];        h = S[6];        v = S[7];        for (var x = 0; x < 64; x++) {          k[x] = x < 16 ? r[x + p] : a(a(a(e(m = k[x - 2], 17) ^ e(m, 19) ^ m >>> 10, k[x - 7]), e(y = k[x - 15], 7) ^ e(y, 18) ^ y >>> 3), k[x - 16]);          var mint1716432199485000 = a(v, e(b = c, 6) ^ e(b, 11) ^ e(b, 25));          var mint1716432199485000 = a(mint1716432199485000, (w = c) & i ^ ~w & h);          var mint1716432199485000 = a(mint1716432199485000, D[x]);          s = a(mint1716432199485000, k[x]);          g = a(e(d = t, 2) ^ e(d, 13) ^ e(d, 22), (l = t) & (C = o) ^ l & (A = u) ^ C & A);          v = h;          h = i;          i = c;          c = a(f, s);          f = u;          u = o;          o = t;          t = a(s, g);        }        S[0] = a(t, S[0]);        S[1] = a(o, S[1]);        S[2] = a(u, S[2]);        S[3] = a(f, S[3]);        S[4] = a(c, S[4]);        S[5] = a(i, S[5]);        S[6] = a(h, S[6]);        S[7] = a(v, S[7]);      }      return S;    }(function (r) {      t = Array();      a = (1 << n) - 1;      e = 0;      for (void 0; e < r.length * n; e += n) {        var t;        var a;        var e;        t[e >> 5] |= (r.charCodeAt(e / n) & a) << 24 - e % 32;      }      return t;    }(r = function (r) {      var n = new RegExp("\n", "g");      r = r.replace(n, "\n");      t = "";      a = 0;      for (void 0; a < r.length; a++) {        var t;        var a;        var e = r.charCodeAt(a);        if (e < 128) {          t += String.fromCharCode(e);        } else {          if (e > 127 && e < 2048) {            t += String.fromCharCode(e >> 6 | 192);            t += String.fromCharCode(63 & e | 128);          } else {            t += String.fromCharCode(e >> 12 | 224);            t += String.fromCharCode(e >> 6 & 63 | 128);            t += String.fromCharCode(63 & e | 128);          }        }      }      return t;    }(r)), r.length * n));  }  cookie = r.tn + "=" + t[0] + ";Max-age=" + r.vt + "; path = /";  return cookie;}
```

以上内容纯属虚构，如有雷同，纯属巧合。如有冒犯，还请见谅。（仅供学习交流）