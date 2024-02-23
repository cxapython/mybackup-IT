> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.yllhwa.com](https://blog.yllhwa.com/2023/09/29/Android%20QQ%20NT%20%E7%89%88%E6%95%B0%E6%8D%AE%E5%BA%93%E8%A7%A3%E5%AF%86/)

> 以下分析均基于当前最新版本（8.9.78.12275）进行，不保证跨版本兼容性，但是分析思路应该相似。

以下分析均基于当前最新版本（8.9.78.12275）进行，不保证跨版本兼容性，但是分析思路应该相似。

首先通过简单的定位（文件检测、字符串查找等等）可以得知 QQ NT 版的数据库存放在`/data/data/com.tencent.mobileqq/databases/nt_db/nt_qq_{账号相关的hash}/nt_msg.db`中，具体的实现代码在`libkernel.so`中。

根据我们的经验和简单的观察可以发现，QQ NT 版还是采用了 SQLCipher 的加密方案。

### [](#SQLCipher-版本定位 "SQLCipher 版本定位")SQLCipher 版本定位

由于 SQLCipher 是开源库，为了我们接下来分析逆向代码的方便性，我们可以先定位到其版本，将对应的代码下载下来对照观察。

参考：[对解密某国产聊天软件聊天数据库的分析](https://www.cjovi.icu/software-testing/1650.html)

根据上文内容，可以通过搜索`misuse`字符串，快速定位到`sqlite3_log`函数，同时可以得到版本信息`872ba256cbf61d9290b571c0e6d82a20c224ca3ad82971edc46b29818d5dalt1`。通过搜索这个信息我们可以得到 SQLCipher 的版本为`4.5.1`（其实在字符串中直接搜索`4.`也搜得到）。

在 GitHub 上将对应版本的代码下载备用。

### [](#解密方式思考 "解密方式思考")解密方式思考

最佳的解密方式肯定是直接将数据库拿出来，自己用 SQLCipher 解密，但是说实话，我折腾了半天都解不出来（其他平台可以很轻松地用原版 SQLCipher 解密）。

于是只能退而求其次，用 Hook 的方式在程序执行中进行解密。此处我们选择 Frida 作为 Hook 工具。

### [](#sqlite3-exec-定位 "sqlite3_exec 定位")sqlite3_exec 定位

考虑到 QQ 执行过程中肯定会打开聊天消息数据库，我们只需要 Hook 到`sqlite3_exec`即可。

由于我们手中有源码，定位相当简单，只需要从该函数中挑选一个字符串搜索后查找引用即可。

此处我随意选择了一个调用了`sqlite3_exec`的函数`sqlcipher_check_connection`。

```
static int sqlcipher_check_connection(const char *filename, char *key, int key_sz, char *sql, int *user_version, char** journal_mode) {
  int rc;
  sqlite3 *db = NULL;
  sqlite3_stmt *statement = NULL;
  char *query_journal_mode = "PRAGMA journal_mode;";
  char *query_user_version = "PRAGMA user_version;";

  rc = sqlite3_open(filename, &db);
  if(rc != SQLITE_OK) goto cleanup;

  rc = sqlite3_key(db, key, key_sz);
  if(rc != SQLITE_OK) goto cleanup;

  rc = sqlite3_exec(db, sql, NULL, NULL, NULL);
  if(rc != SQLITE_OK) goto cleanup;
  ...

```

在 IDA 中查找`PRAGMA journal_mode;`即可交叉定位到`sqlcipher_check_connection`函数（0x1CE8634），进而定位到`sqlite3_exec`函数（0x1CE8DFC）。此处也能定位到`sqlite3_key`函数，需要寻找 Key 时也可以从这里入手。

### [](#通过连接句柄定位文件名 "通过连接句柄定位文件名")通过连接句柄定位文件名

此时还剩下最后一个问题，`sqlite3_exec`是供全体数据库连接调用的，我们要定位到我们需要的数据库的连接，所以要从句柄中定位到文件名。说实话我还真没找到参考，只能大概糊了一个路线：`struct sqlite3->Db *aDb->Btree *pBt->BtShared *pBt->Pager *pPager->char *zFilename`。

这个路线如何寻找？由于该项目的代码质量较高，我们全局搜索`zFilename`，同时这些数据结构中子元素往往对上层元素有反向引用，我们可以一层层上升到句柄结构体（其实我感觉这个活儿肯定有个函数专门处理，但是我没找到）。

### [](#Frida-脚本 "Frida 脚本")Frida 脚本

完成逆向工作后我们就可以编写代码进行 Hook 了，参考网上的资料，我们可以得知在数据库中运行以下指令即可导出没有加密的数据库：

```
ATTACH DATABASE 'plaintext.db' AS plaintext KEY '';
SELECT sqlcipher_export('plaintext');
DETACH DATABASE plaintext;

```

事实上这样在安卓上面跑会出错，我猜测是奇怪的文件权限问题，将明文数据库放在权限较低的位置（如`/storage/emulated/0/Download/plaintext.db`）即可。最终形成的 Frida 脚本如下：

**注意，直接对数据库进行操作有可能损坏数据库，请务必做好备份。**

```
const DATABASE_URI =  "/data/user/0/com.tencent.mobileqq/databases/nt_db/nt_qq_{CHNAGE_THIS_TO_YOURS}/nt_msg.db";


let SQLITE3_EXEC_CALLBACK_LOG = true;
let index1 = 0;
let xCallback = new NativeCallback(
  (para, nColumn, colValue, colName) => {
    if (!SQLITE3_EXEC_CALLBACK_LOG) {
      return 0;
    }
    console.log();
    console.log(
      "------------------------" + index1++ + "------------------------"
    );
    for (let index = 0; index < nColumn; index++) {
      let c_name = colName
        .add(index * 8)
        .readPointer()
        .readUtf8String();
      let c_value = "";
      try {
        c_value =
          colValue
            .add(index * 8)
            .readPointer()
            .readUtf8String() ?? "";
      } catch {}
      console.log(c_name, "\t", c_value);
    }
    return 0;
  },
  "int",
  ["pointer", "int", "pointer", "pointer"]
);


let get_filename_from_sqlite3_handle = function (sqlite3_db) {
  
  let zFilename = "";
  try {
    let db_pointer = sqlite3_db.add(0x8 * 5).readPointer();
    let pBt = db_pointer.add(0x8).readPointer();
    let pBt2 = pBt.add(0x8).readPointer();
    let pPager = pBt2.add(0x0).readPointer();
    zFilename = pPager.add(208).readPointer().readCString();
  } catch (e) {}
  return zFilename;
};

setTimeout(function () {
  let base_addr = Module.findBaseAddress("libkernel.so");
  console.log("libkernel.so base address: " + base_addr);

  
  let sqlite3_exec_addr = base_addr.add(0x1cfb9c0);
  console.log("sqlite3_exec_addr: " + sqlite3_exec_addr);

  let sqlite3_exec = new NativeFunction(sqlite3_exec_addr, "int", [
    "pointer",
    "pointer",
    "pointer",
    "int",
    "int",
  ]);

  let target_db_handle = null;
  let js_sqlite3_exec = function (sql) {
    if (target_db_handle == null) {
      return -1;
    }
    let sql_pointer = Memory.allocUtf8String(sql);
    return sqlite3_exec(target_db_handle, sql_pointer, xCallback, 0, 0);
  };

  
  Interceptor.attach(sqlite3_exec_addr, {
    onEnter: function (args) {
      
      let sqlite3_db = ptr(args[0]);
      let sql = Memory.readCString(args[1]);
      let callback_addr = ptr(args[2]);
      let callback_arg = ptr(args[3]);
      let errmsg = ptr(args[4]);
      let databasae_name = get_filename_from_sqlite3_handle(sqlite3_db);
      if (databasae_name == DATABASE_URI) {
        console.log("sqlite3_db: " + sqlite3_db);
        console.log("sql: " + sql);
        target_db_handle = sqlite3_db;
      }
    },
  });
  setTimeout(function () {
    let ret = js_sqlite3_exec(
      `ATTACH DATABASE '/storage/emulated/0/Download/plaintext.db' AS plaintext KEY '';SELECT sqlcipher_export('plaintext');DETACH DATABASE plaintext;`
    );
    console.log("js_sqlite3_exec ret: " + ret);
  }, 4000);
}, 1200);

```