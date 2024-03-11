> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/wyh0923/p/16590583.html)

#### 站点[#](#1240337573)

[链接地址：](https://36kr.com/search/articles/Lx)

#### 流程分析[#](#3016409716)

第一次请求的结果，返回 Set Cookies， acw_sc__v2

*   请求代码:

```
import requests

cookies = {
    'acw_tc': '2760777b16605488858268722e36f0bfd1a7bc6b6fe327676b71a938b643c8',
    'acw_sc__v2': '62f9f7156a51532ffe88edd2d58bba1851e138ca',
    'Hm_lvt_1684191ccae0314c6254306a8333d090': '1660548886',
    'Hm_lvt_713123c60a0e86982326bae1a51083e1': '1660548886',
    'sajssdk_2015_cross_new_user': '1',
    'sensorsdata2015jssdkcross': '%7B%22distinct_id%22%3A%22182a06d31213eb-0edd19cffc882f-3e604809-2073600-182a06d3122499%22%2C%22%24device_id%22%3A%22182a06d31213eb-0edd19cffc882f-3e604809-2073600-182a06d3122499%22%2C%22props%22%3A%7B%22%24latest_traffic_source_type%22%3A%22%E7%9B%B4%E6%8E%A5%E6%B5%81%E9%87%8F%22%2C%22%24latest_referrer%22%3A%22%22%2C%22%24latest_referrer_host%22%3A%22%22%2C%22%24latest_search_keyword%22%3A%22%E6%9C%AA%E5%8F%96%E5%88%B0%E5%80%BC_%E7%9B%B4%E6%8E%A5%E6%89%93%E5%BC%80%22%7D%7D',
    'Hm_lpvt_1684191ccae0314c6254306a8333d090': '1660550021',
    'Hm_lpvt_713123c60a0e86982326bae1a51083e1': '1660550021',
    'SERVERID': 'd36083915ff24d6bb8cb3b8490c52181|1660550414|1660548887',
}

headers = {
    'authority': '36kr.com',
    'cache-control': 'max-age=0',
    'upgrade-insecure-requests': '1',
    'user-agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.198 Safari/537.36',
    'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
    'sec-fetch-site': 'none',
    'sec-fetch-mode': 'navigate',
    'sec-fetch-user': '?1',
    'sec-fetch-dest': 'document',
    'accept-language': 'zh-CN,zh;q=0.9',
    # Requests sorts cookies= alphabetically
    # 'cookie': 'acw_tc=2760777b16605488858268722e36f0bfd1a7bc6b6fe327676b71a938b643c8; acw_sc__v2=62f9f7156a51532ffe88edd2d58bba1851e138ca; Hm_lvt_1684191ccae0314c6254306a8333d090=1660548886; Hm_lvt_713123c60a0e86982326bae1a51083e1=1660548886; sajssdk_2015_cross_new_user=1; sensorsdata2015jssdkcross=%7B%22distinct_id%22%3A%22182a06d31213eb-0edd19cffc882f-3e604809-2073600-182a06d3122499%22%2C%22%24device_id%22%3A%22182a06d31213eb-0edd19cffc882f-3e604809-2073600-182a06d3122499%22%2C%22props%22%3A%7B%22%24latest_traffic_source_type%22%3A%22%E7%9B%B4%E6%8E%A5%E6%B5%81%E9%87%8F%22%2C%22%24latest_referrer%22%3A%22%22%2C%22%24latest_referrer_host%22%3A%22%22%2C%22%24latest_search_keyword%22%3A%22%E6%9C%AA%E5%8F%96%E5%88%B0%E5%80%BC_%E7%9B%B4%E6%8E%A5%E6%89%93%E5%BC%80%22%7D%7D; Hm_lpvt_1684191ccae0314c6254306a8333d090=1660550021; Hm_lpvt_713123c60a0e86982326bae1a51083e1=1660550021; SERVERID=d36083915ff24d6bb8cb3b8490c52181|1660550414|1660548887',
}

response = requests.get('https://36kr.com/search/articles/Lx', cookies=cookies, headers=headers)


```

*   响应结果

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816090518987-300914406.png)

*   把 js 抠出来格式化之后就可以看出来是经典的 OB 混淆

[工具地址：](https://github.com/DingZaiHub/ob-decrypt)

##### OB 混淆经典代码参照

*   图一 在 JS 逆向的过程中，我们可能经常碰到类似如下的代码

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816091149825-968722241.png)

*   图二 开头定义了一个大数组，然后对这个大数组里的内容进行位移，再定义一个解密函数。后面大部分的值都调用了这个解密函数，以达到混淆的效果。这种代码即为 ob 混淆，不仅变量名混淆了，运行逻辑等也高度混淆，难以理解。使用本工具后可以一键还原，达到的效果如下：

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816091802658-720322542.png)

*   站点内容 JS 如下：

```
var arg1 = "F64C612FC4BB3852CCA8F93A012D3105A724E6EB";
var arg3 = null;
var arg4 = null;
var arg5 = null;
var arg6 = null;
var arg7 = null;
var arg8 = null;
var arg9 = null;
var arg10 = null;

var l = function () {
  while (window["_phantom"] || window["__phantomas"]) {}

  var _0x5e8b26 = "3000176000856006061501533003690027800375";

  String["prototype"]["hexXor"] = function (_0x4e08d8) {
    var _0x5a5d3b = "";

    for (var _0xe89588 = 0; _0xe89588 < this["length"] && _0xe89588 < _0x4e08d8["length"]; _0xe89588 += 2) {
      var _0x401af1 = parseInt(this["slice"](_0xe89588, _0xe89588 + 2), 16);

      var _0x105f59 = parseInt(_0x4e08d8["slice"](_0xe89588, _0xe89588 + 2), 16);

      var _0x189e2c = (_0x401af1 ^ _0x105f59)["toString"](16);

      if (_0x189e2c["length"] == 1) {
        _0x189e2c = "0" + _0x189e2c;
      }

      _0x5a5d3b += _0x189e2c;
    }

    return _0x5a5d3b;
  };

  String["prototype"]["unsbox"] = function () {
    var _0x4b082b = [15, 35, 29, 24, 33, 16, 1, 38, 10, 9, 19, 31, 40, 27, 22, 23, 25, 13, 6, 11, 39, 18, 20, 8, 14, 21, 32, 26, 2, 30, 7, 4, 17, 5, 3, 28, 34, 37, 12, 36];
    var _0x4da0dc = [];
    var _0x12605e = "";

    for (var _0x20a7bf = 0; _0x20a7bf < this["length"]; _0x20a7bf++) {
      var _0x385ee3 = this[_0x20a7bf];

      for (var _0x217721 = 0; _0x217721 < _0x4b082b["length"]; _0x217721++) {
        if (_0x4b082b[_0x217721] == _0x20a7bf + 1) {
          _0x4da0dc[_0x217721] = _0x385ee3;
        }
      }
    }

    _0x12605e = _0x4da0dc["join"]("");
    return _0x12605e;
  };

  var _0x23a392 = arg1["unsbox"]();

  arg2 = _0x23a392["hexXor"](_0x5e8b26);
  setTimeout("reload(arg2)", 2);
};

var _0x4db1c = function () {
  function _0x355d23(_0x450614) {
    if (("" + _0x450614 / _0x450614)["length"] !== 1 || _0x450614 % 20 === 0) {
      (function () {})["constructor"]("undefined"[2] + "true"[3] + ([]["entries"]() + "")[2] + "undefined"[0] + ("false0" + String)[20] + ("false0" + String)[20] + "true"[3] + "true"[1])();
    } else {
      (function () {})["constructor"]("undefined"[2] + "true"[3] + ([]["entries"]() + "")[2] + "undefined"[0] + ("false0" + String)[20] + ("false0" + String)[20] + "true"[3] + "true"[1])();
    }

    _0x355d23(++_0x450614);
  }

  try {
    _0x355d23(0);
  } catch (_0x54c483) {}
};

if (function () {
  var _0x470d8f = function () {
    var _0x4c97f0 = true;
    return function (_0x1742fd, _0x4db1c) {
      var _0x48181e = _0x4c97f0 ? function () {
        if (_0x4db1c) {
          var _0x55f3be = _0x4db1c["apply"](_0x1742fd, arguments);

          _0x4db1c = null;
          return _0x55f3be;
        }
      } : function () {};

      _0x4c97f0 = false;
      return _0x48181e;
    };
  }();

  var _0x501fd7 = _0x470d8f(this, function () {
    var _0x4c97f0 = function () {
      return "dev";
    },
        _0x1742fd = function () {
      return "window";
    };

    var _0x55f3be = function () {
      var _0x3ad9a1 = new RegExp("\\w+ *\\(\\) *{\\w+ *['|\"].+['|\"];? *}");

      return !_0x3ad9a1["test"](_0x4c97f0["toString"]());
    };

    var _0x1b93ad = function () {
      var _0x20bf34 = new RegExp("(\\\\[x|u](\\w){2,4})+");

      return _0x20bf34["test"](_0x1742fd["toString"]());
    };

    var _0x5afe31 = function (_0x178627) {
      var _0x1a0f04 = 0;

      if (_0x178627["indexOf"](false)) {
        _0xd79219(_0x178627);
      }
    };

    var _0xd79219 = function (_0x5792f7) {
      var _0x4e08d8 = 3;

      if (_0x5792f7["indexOf"]("true"[3]) !== _0x4e08d8) {
        _0x5afe31(_0x5792f7);
      }
    };

    if (!_0x55f3be()) {
      if (!_0x1b93ad()) {
        _0x5afe31("ind\u0435xOf");
      } else {
        _0x5afe31("indexOf");
      }
    } else {
      _0x5afe31("ind\u0435xOf");
    }
  });

  _0x501fd7();

  var _0x3a394d = function () {
    var _0x1ab151 = true;
    return function (_0x372617, _0x42d229) {
      var _0x3b3503 = _0x1ab151 ? function () {
        if (_0x42d229) {
          var _0x7086d9 = _0x42d229["apply"](_0x372617, arguments);

          _0x42d229 = null;
          return _0x7086d9;
        }
      } : function () {};

      _0x1ab151 = false;
      return _0x3b3503;
    };
  }();

  var _0x5b6351 = _0x3a394d(this, function () {
    var _0x46cbaa = Function("return (function() {}.constructor(\"return this\")( ));");

    var _0x1766ff = function () {};

    var _0x9b5e29 = _0x46cbaa();

    _0x9b5e29["console"]["log"] = _0x1766ff;
    _0x9b5e29["console"]["error"] = _0x1766ff;
    _0x9b5e29["console"]["warn"] = _0x1766ff;
    _0x9b5e29["console"]["info"] = _0x1766ff;
  });

  _0x5b6351();

  try {
    return !!window["addEventListener"];
  } catch (_0x35538d) {
    return false;
  }
}()) {
  document["addEventListener"]("DOMContentLoaded", l, false);
} else {
  document["attachEvent"]("onreadystatechange", l);
}

_0x4db1c();

setInterval(function () {
  _0x4db1c();
}, 4000);

function setCookie(name, value) {
  var expiredate = new Date();
  expiredate.setTime(expiredate.getTime() + 3600000);
  document.cookie = name + "=" + value + ";expires=" + expiredate.toGMTString() + ";max-age=3600;path=/";
}

function reload(x) {
  setCookie("acw_sc__v2", x);
  document.location.reload();
}



```

#### JS 文件[#](#1885099409)

*   关键位置一 ：

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816092228885-1923403816.png)

*   关键位置二 ： `arg2 = _0x23a392[_0x55f3("0x1b", "z5O&")](_0x5e8b26);`

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816092528936-335443576.png)

*   arg2 是 acw_sc__v2。 把定时器和不相关代码删掉，补一下 window 环境。
    
*   最终生成 acw_sc__v2 的代码只有几十行。
    
*   关键函数只要 unsbox 和 hexXor，都很容易理解，对字符串进行一些处理。另外经测试，arg1 是动态的，每次生成时需要传入当前返回的 arg1。
    
*   最终 JS 文件
    

```
window = {}

var arg1 = "F64C612FC4BB3852CCA8F93A012D3105A724E6EB";

function l() {
  while (window["_phantom"] || window["__phantomas"]) {}
  var _0x5e8b26 = "3000176000856006061501533003690027800375";
  String["prototype"]["hexXor"] = function (_0x4e08d8) {
    var _0x5a5d3b = "";
    for (var _0xe89588 = 0; _0xe89588 < this["length"] && _0xe89588 < _0x4e08d8["length"]; _0xe89588 += 2) {
      var _0x401af1 = parseInt(this["slice"](_0xe89588, _0xe89588 + 2), 16);

      var _0x105f59 = parseInt(_0x4e08d8["slice"](_0xe89588, _0xe89588 + 2), 16);

      var _0x189e2c = (_0x401af1 ^ _0x105f59)["toString"](16);

      if (_0x189e2c["length"] == 1) {
        _0x189e2c = "0" + _0x189e2c;
      }

      _0x5a5d3b += _0x189e2c;
    }

    return _0x5a5d3b;
  };

  String["prototype"]["unsbox"] = function () {
    var _0x4b082b = [15, 35, 29, 24, 33, 16, 1, 38, 10, 9, 19, 31, 40, 27, 22, 23, 25, 13, 6, 11, 39, 18, 20, 8, 14, 21, 32, 26, 2, 30, 7, 4, 17, 5, 3, 28, 34, 37, 12, 36];
    var _0x4da0dc = [];
    var _0x12605e = "";

    for (var _0x20a7bf = 0; _0x20a7bf < this["length"]; _0x20a7bf++) {
      var _0x385ee3 = this[_0x20a7bf];

      for (var _0x217721 = 0; _0x217721 < _0x4b082b["length"]; _0x217721++) {
        if (_0x4b082b[_0x217721] == _0x20a7bf + 1) {
          _0x4da0dc[_0x217721] = _0x385ee3;
        }
      }
    }

    _0x12605e = _0x4da0dc["join"]("");
    return _0x12605e;
  };

  var _0x23a392 = arg1["unsbox"]();

  arg2 = _0x23a392["hexXor"](_0x5e8b26);
  console.log(arg2)

}

l()


```

#### python 复写[#](#2247008080)

```
# -*- coding: utf-8 -*-
# @Time    : 2022/3/23 14:12
# @IDE ：PyCharm
import requests,re

def unsbox(arg1):
    box = [15, 35, 29, 24, 33, 16, 1, 38, 10, 9, 19, 31, 40, 27, 22, 23, 25, 13, 6, 11, 39, 18, 20, 8, 14, 21, 32, 26, 2, 30, 7, 4, 17, 5, 3, 28, 34, 37, 12, 36]
    res = list(range(0, len(arg1)))
    for i in range(0, len(arg1)):
        j = arg1[i]
        for k in range(0, 40):
            if box[k] == i+1:
                res[k] = j
    res = "".join(res)
    return res


def hexXor(arg2):
    box = "3000176000856006061501533003690027800375"
    res = ""
    for i in range(0, 40, 2):
        arg_H = int(arg2[i:i+2], 16)
        box_H = int(box[i:i+2], 16)
        res += hex(arg_H ^ box_H)[2:].zfill(2)
    return res


def get_acw_sc_v2(arg1):
    return hexXor(unsbox(arg1))


sess = requests.session()
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36"
}
sess.headers = headers
base_url = 'https://36kr.com/search/articles/%E5%90%88%E6%88%90%E7%94%9F%E7%89%A9'

script = sess.get(base_url).content.decode()
arg1 = ''.join(re.findall("var arg1='(.*?)';",script,re.S))
headers.update({"cookie":f"acw_sc__v2={get_acw_sc_v2(arg1)}"})

html = sess.get(base_url).content.decode()
print(''.join(re.findall('window.initialState=(.*?})</script>', html, re.S)))



```

#### 详情页面 AES[#](#2176671363)

*   详情页面 html 部分已经被加密了

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816093551799-2127139457.png)

#### 定位分析[#](#2240287778)

*   一般情况全局搜索找不到返回内容，大概率是对数据进行了编码或者加密。
*   全局搜索 decrypt，找到解密位置

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816094155787-566567453.png)

###### 代码部分

```
    var ne = ee.a.enc.Utf8.parse("efabccee-b754-4c");
    var re, oe = window.initialState || {};
    oe.isEncrypt && (oe = JSON.parse((re = window.initialState.state,
    ee.a.AES.decrypt(re, ne, {
        mode: ee.a.mode.ECB,
        padding: ee.a.pad.Pkcs7
    }).toString(ee.a.enc.Utf8).toString())));


```

*   开启断点调试控制台打印

![](https://img2022.cnblogs.com/blog/1663687/202208/1663687-20220816094830466-1905671091.png)

#### 本地还原[#](#626994494)

###### 解密代码

```
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad

def decrypt(data):
    html = base64.b64decode(data)
    key = b'efabccee-b754-4c'
    aes = AES.new(key=key, mode=AES.MODE_ECB)
    info = aes.decrypt(html)
    decrypt_data = pad(info, 16).decode()
    return decrypt_data


```

###### 完整代码

```
# -*- coding: utf-8 -*-

import requests,json,re

def unsbox(arg1):
    box = [15, 35, 29, 24, 33, 16, 1, 38, 10, 9, 19, 31, 40, 27, 22, 23, 25, 13, 6, 11, 39, 18, 20, 8, 14, 21, 32, 26, 2, 30, 7, 4, 17, 5, 3, 28, 34, 37, 12, 36]
    res = list(range(0, len(arg1)))
    for i in range(0, len(arg1)):
        j = arg1[i]
        for k in range(0, 40):
            if box[k] == i+1:
                res[k] = j
    res = "".join(res)
    return res

def hexXor(arg2):
    box = "3000176000856006061501533003690027800375"
    res = ""
    for i in range(0, 40, 2):
        arg_H = int(arg2[i:i+2], 16)
        box_H = int(box[i:i+2], 16)
        res += hex(arg_H ^ box_H)[2:].zfill(2)
    return res

def get_acw_sc_v2(arg1 = "91C303C944FD7A5FF2468DB232B4F1BC59108A76"):
    return hexXor(unsbox(arg1))



import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad

def decrypt(data):
    html = base64.b64decode(data)
    key = b'efabccee-b754-4c'
    aes = AES.new(key=key, mode=AES.MODE_ECB)
    info = aes.decrypt(html)
    decrypt_data = pad(info, 16).decode()
    return decrypt_data


headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36',
}
sess = requests.session()
sess.headers = headers
base_url = 'https://36kr.com/search/articles/%E5%90%88%E6%88%90%E7%94%9F%E7%89%A9'
html = sess.get(base_url).content.decode()
arg1 = ''.join(re.findall("var arg1='(.*?)';", html, re.S))
if arg1:
    headers.update({"cookie": f"acw_sc__v2={get_acw_sc_v2(arg1)}"})
    html = sess.get(base_url).content.decode()
data = json.loads(''.join(re.findall('window.initialState=(.*?})</script>', html, re.S)))
for li in data['searchResultData']['data']['searchResult']['data']['itemList']:
    r= sess.get('https://36kr.com/p/' + str(li['itemId']))
    initialState = json.loads(''.join(re.findall('<script>window.initialState=(.*?)</script>',r.text,re.S)))['state']
    print(decrypt(initialState))
    break



```