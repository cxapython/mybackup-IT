> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/pureszgd/article/details/89478574)

xposed 开发 13 - 微信消息防撤回
======================

```
try {
    val clazz = XposedHelpers.findClass("com.tencent.wcdb.database.SQLiteDatabase", lpparam.classLoader)
    XposedHelpers.findAndHookMethod(clazz, "updateWithOnConflict", String::class.java, ContentValues::class.java,
        String::class.java, Array<String>::class.java, Int::class.java, object : XC_MethodHook() {
            override fun beforeHookedMethod(param: MethodHookParam) {
                if (param.args[0] == "message") {
                    val contentValues = param.args[1] as ContentValues
                    val content = contentValues.getAsString("content")
                    if (contentValues.getAsInteger("type") == 10000 && content != "你撤回了一条消息") {
                        val msgId = contentValues.getAsLong("msgId")
                        val msg = msgContainer[msgId]
                        XposedHelpers.setObjectField(msg, "field_content", "已拦截 " + content.substring(1, content.indexOf("撤") - 2) + " 的撤回")
                        XposedHelpers.setIntField(msg, "field_type", contentValues.getAsInteger("type"))
                        XposedHelpers.setLongField(msg, "field_createTime", XposedHelpers.getLongField(msg, "field_createTime") + 1L)
                        XposedHelpers.callMethod(insertAny, "c", msg, false)
                        param.result = 1
                    }
                }
                super.beforeHookedMethod(param)
            }
        })

    // Hook 删除资源文件数据库记录的方法
    XposedHelpers.findAndHookMethod(clazz, "delete", String::class.java, String::class.java, Array<String>::class.java, object : XC_MethodHook() {
        override fun beforeHookedMethod(param: MethodHookParam) {
            val tableName = param.args[0].toString()
            if (tableName.contains("voiceinfo") || tableName.contains("ImgInfo2") ||
                tableName.contains("videoinfo2") || tableName.contains("WxFileIndex2")) {
                param.result = 1
            }
            super.beforeHookedMethod(param)
        }
    })

    // Hook 删除文件的方法
    XposedHelpers.findAndHookMethod(File::class.java, "delete", object : XC_MethodHook() {
        override fun beforeHookedMethod(param: MethodHookParam) {
            val filePath = (param.thisObject as File).absolutePath
            if (filePath.contains("/image2/") || filePath.contains("/voice2/") || filePath.contains("/video/")) {
                param.result = true
            }
            super.afterHookedMethod(param)
        }
    })

    // Hook 插入信息的方法
    val cla = XposedHelpers.findClass("com.tencent.mm.storage.bj", lpparam.classLoader)
    XposedBridge.hookAllMethods(cla, "c", object : XC_MethodHook() {
        override fun afterHookedMethod(param: MethodHookParam) {
            insertAny = param.thisObject
            val bd = param.args[0]
            val field_msgId = XposedHelpers.getLongField(bd, "field_msgId")
            msgContainer[field_msgId] = bd
            super.afterHookedMethod(param)
        }
    })

} catch (e: Error) {
    e.printStackTrace()
} catch (e: Exception) {
    e.printStackTrace()
}
```

![](https://img-blog.csdnimg.cn/20190423184107181.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3B1cmVzemdk,size_16,color_FFFFFF,t_70)