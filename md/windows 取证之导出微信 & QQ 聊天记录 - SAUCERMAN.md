> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [saucer-man.com](https://saucer-man.com/information_security/1038.html)

> 记录一下 windows 取证的一些步骤，主要是为了导出 windows 上的微信 & qq 的聊天记录，没什么新的东西，把最近实践的过程记录下。1. 微信聊天记录微信聊天记录存储在 sqlite 数据库中，但是加...

> 文章最后更新时间为：2023 年 06 月 01 日 14:01:38

记录一下 windows 取证的一些步骤，主要是为了导出 windows 上的微信 & qq 的聊天记录，没什么新的东西，把最近实践的过程记录下。

1. 微信聊天记录
---------

微信聊天记录存储在 sqlite 数据库中，但是加密了，所以需要先获取到解密密钥。解密密钥的获取一般都是在内存中，所以需要微信在登录状态下才能获取到。

### 1.1 获取数据库解密密钥

这里要利用到开源工具 [https://github.com/AdminTest0/SharpWxDump](https://github.com/AdminTest0/SharpWxDump)，需要自己编译

一般情况下，我用 vs2022 打开项目，会提示当前目标. net 版本为 4.0，是否需要升级该项目的. net 版本，这时候最好别升级，尽可能适配更多的目标环境。安装 4.0 的运行环境可以参考这篇文章：[https://blog.csdn.net/GoodCooking/article/details/125347856](https://blog.csdn.net/GoodCooking/article/details/125347856)

编译选择 release、x86 即可。

[![](https://saucer-man.com/usr/uploads/2023/05/2074925837.png)](https://saucer-man.com/usr/uploads/2023/05/2074925837.png "2023-05-30T07:49:17.png")

直接运行可以获取到 key:

[![](https://saucer-man.com/usr/uploads/2023/05/1507407021.png)](https://saucer-man.com/usr/uploads/2023/05/1507407021.png "2023-05-30T07:50:16.png")

### 1.2 解密微信数据库

微信数据默认情况下位于`C:\Users\xxx\Documents\WeChat Files\`，而聊天记录数据库位于该目录下面的`wxid_xxxx\Msg\Multi`。聊天记录文件命名一般是 MSG.db，超出 240MB 会自动生成 MSG1.db，以此类推

```
wxid_xxxxxxxx\Msg\Multi\MSG0.db > 聊天记录
wxid_xxxxxxxx\Msg\Multi\MSG1.db > 聊天记录
wxid_xxxxxxxx\Msg\Multi\MSG2.db > 聊天记录
wxid_xxxxxxxx\Msg\MicroMsg.db > Contact字段 > 好友列表
wxid_xxxxxxxx\Msg\MediaMsg.db > 语音 > 格式为silk

```

[![](https://saucer-man.com/usr/uploads/2023/05/1148762794.png)](https://saucer-man.com/usr/uploads/2023/05/1148762794.png "2023-05-30T07:53:14.png")

在上一步获取到解密密钥之后，我们可以直接使用下面的脚本解密聊天记录数据库:

```
from Crypto.Cipher import AES
import hashlib, hmac, ctypes, sys, getopt

SQLITE_FILE_HEADER = bytes('SQLite format 3', encoding='ASCII') + bytes(1)
IV_SIZE = 16
HMAC_SHA1_SIZE = 20
KEY_SIZE = 32
DEFAULT_PAGESIZE = 4096
DEFAULT_ITER = 64000
opts, args = getopt.getopt(sys.argv[1:], 'hk:d:')
input_pass = ''
input_dir = ''

for op, value in opts:
    if op == '-k':
        input_pass = value
    else:
        if op == '-d':
            input_dir = value

password = bytes.fromhex(input_pass.replace(' ', ''))

with open(input_dir, 'rb') as (f):
    blist = f.read()
print(len(blist))
salt = blist[:16]
key = hashlib.pbkdf2_hmac('sha1', password, salt, DEFAULT_ITER, KEY_SIZE)
first = blist[16:DEFAULT_PAGESIZE]
mac_salt = bytes([x ^ 58 for x in salt])
mac_key = hashlib.pbkdf2_hmac('sha1', key, mac_salt, 2, KEY_SIZE)
hash_mac = hmac.new(mac_key, digestmod='sha1')
hash_mac.update(first[:-32])
hash_mac.update(bytes(ctypes.c_int(1)))

if hash_mac.digest() == first[-32:-12]:
    print('Decryption Success')
else:
    print('Password Error')
blist = [blist[i:i + DEFAULT_PAGESIZE] for i in range(DEFAULT_PAGESIZE, len(blist), DEFAULT_PAGESIZE)]

with open(input_dir, 'wb') as (f):
    f.write(SQLITE_FILE_HEADER)
    t = AES.new(key, AES.MODE_CBC, first[-48:-32])
    f.write(t.decrypt(first[:-48]))
    f.write(first[-48:])
    for i in blist:
        t = AES.new(key, AES.MODE_CBC, i[-48:-32])
        f.write(t.decrypt(i[:-48]))
        f.write(i[-48:])

```

windows 上如果报错没有 Crypto，就使用下面的命令安装下:

```
pip install pycryptodome

```

运行命令如下：

```
python3 .\jiemi.py -k 数据库密钥 -d .\MSG0.db

```

[![](https://saucer-man.com/usr/uploads/2023/05/2028413655.png)](https://saucer-man.com/usr/uploads/2023/05/2028413655.png "2023-05-30T07:57:25.png")

### 1.3 打开聊天记录

直接使用 navicate 导入 sqllite 即可：

[![](https://saucer-man.com/usr/uploads/2023/05/784168713.png)](https://saucer-man.com/usr/uploads/2023/05/784168713.png "2023-05-30T07:59:44.png")

可以使用查询语句快速检索如查询带 "密码" 的聊天记录，同时包含个人消息、群消息等等

```
SELECT * FROM "MSG" WHERE StrContent  like'%密码%'

```

2. qq 聊天记录
----------

qq 的数据存储也是通过加密的 sqlite，但是和微信不一样的是，qq 数据库解密的密钥是会变化的，每次登录时会下发一个密钥，所以这也导致了密钥和数据库是一一对应的。

qq 聊天记录文件一般保存在`C:\Users\xxxx\Documents\Tencent Files\xxxxxxx\Msg3.0.db`

### 2.1 获取解密密钥并解密数据库

解密密钥的破解可以找到下面几个文章：

*   [https://bbs.kanxue.com/thread-250509.htm](https://bbs.kanxue.com/thread-250509.htm)
*   [https://www.52pojie.cn/thread-1370802-1-1.html](https://www.52pojie.cn/thread-1370802-1-1.html)
*   [https://bbs.kanxue.com/thread-266370.html](https://bbs.kanxue.com/thread-266370.html)

我们可以直接使用项目 [https://github.com/Young-Lord/qq-win-db-key](https://github.com/Young-Lord/qq-win-db-key) 来获取解密密钥，在登录时使用 frida hook KernelUtil.dll，拿到 key 之后通过 hook 解密函数可以直接获取解密之后的 db 文件。

该脚本运行时需要 hook 到登录过程，也就是执行要遵守：备份`Msg3.0.db` -> 打开 QQ -> `python hook.py` -> 登录 -> 得到 key

[![](https://saucer-man.com/usr/uploads/2023/06/1412151283.png)](https://saucer-man.com/usr/uploads/2023/06/1412151283.png "2023-06-01T03:26:37.png")

可以看到解密之后的 db 文件已经保存在当前目录下面了，可以直接使用 navicate 打开 db 文件：

[![](https://saucer-man.com/usr/uploads/2023/06/4113974307.png)](https://saucer-man.com/usr/uploads/2023/06/4113974307.png "2023-06-01T03:31:01.png")

主要的聊天信息记录在 MsgContent 列里面，但是由于被编码了，所以没法直接看到明文。

### 2.2 解码 MsgContent

上一步虽然数据库已经解密了，但是数据都被编码了，没办法直接看到明文，所以还需要解码一下。

[https://github.com/Akegarasu/qmsg-unpacker](https://github.com/Akegarasu/qmsg-unpacker) 这个项目可以用来解码 MsgContent，具体用法可以参考 example。

我在使用过程中，发现这个 example 不太友好，大家可以参考下面的 example

```
package main

import (
    "database/sql"
    "fmt"

    "github.com/Akegarasu/qmsg-unpacker/qqmsg"
    _ "github.com/mattn/go-sqlite3"
)

func main() {
    db, err := sql.Open("sqlite3", "./Msg3.0.db_0_xxxxx.db")
    if err != nil {
        panic(err)
    }

    
    rows, err := db.Query("SELECT * FROM group_xxxxxx")
    if err != nil {
        panic(err)
    }
    for rows.Next() {
        var time int
        var rand int
        var senduin int
        var msgcontent []byte
        var msgDecode string
        var info []byte
        err := rows.Scan(&time, &rand, &senduin, &msgcontent, &info)
        if err != nil {
            panic(err)
        }
        fmt.Printf("time: %+v, rand: %+v,senduin: %+v,msgcontent: %+v,info: %+v\n", time, rand, senduin, msgcontent, info)

        msg := qqmsg.Unpack(msgcontent)
        fmt.Printf("msg %+v\n", msg)
        fmt.Printf("SenderNickname： %s\n", msg.SenderNickname)
        fmt.Printf("msg.Header.Time:  %+v\n", msg.Header.Time)
        msgPrintable := qqmsg.EncodeMsg(msg)
        
        fmt.Printf("msgPrintable:  %+v\n", msgPrintable)
        break
    }
    rows.Close()
    db.Close()

}


```

或者可以使用我编写的 python 脚本，项目地址为：[https://github.com/saucer-man/qq_msg_decode](https://github.com/saucer-man/qq_msg_decode)，解码原理就是参考了上面的 golang 项目，在代码中修改下数据库地址，然后直接运行就行，这个脚本会自动将所有的`msgContent`字段解密，并且增加一列`DecodedMsg`用于存放明文，代码用 gpt 转的，有点丑陋，对于数据库比较大的情况下，转换过程会比较缓慢。

下面是 [https://github.com/saucer-man/qq_msg_decode](https://github.com/saucer-man/qq_msg_decode) 解码后的数据库：

[![](https://saucer-man.com/usr/uploads/2023/06/3456895206.png)](https://saucer-man.com/usr/uploads/2023/06/3456895206.png "2023-06-01T06:01:20.png")