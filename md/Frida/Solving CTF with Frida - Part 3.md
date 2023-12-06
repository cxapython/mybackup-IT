> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cmrodriguez.me](https://cmrodriguez.me/blog/hpandro-3/)

> frida ctf challenge sqlite

In this series of posts I’ll be solving some persistence challenges from [hpandro ctf challenges](http://ctf.hpandro.raviramesh.info/). hpAndro created an Android application with multiple vulnerabilities, following the [MSTG](https://github.com/OWASP/owasp-mstg).

We have two different challenges to solve:

*   SQLite database storage.
*   SQLite encrypted storage.

### SQLite database storage

I start by searching the AndroidManifest.xml to find the Activity that could be called in order to solve the task:

```
<activity android:/>


```

by checking the com.hpandro.androidsecurity.ui.activity.task.datastorage.SQLiteDataBaseActivity class. The first method to see is **onCreate** where I found the OnClickListener event for a button with if btnCheckStorageFlag:

```
((Button) y(R.id.btnCheckStorageFlag)).setOnClickListener(new j0(2, this));


```

I went to the class j0, where we had to focus on the following code (the important code is i == 2 because of the first parameter sent in the constructor to the class:

```
} else if (i == 2) {
    ProgressBar progressBar = (ProgressBar) ((SQLiteDataBaseActivity) this.f).y(R.id.progress);
    g.d(progressBar, "progress");
    progressBar.setVisibility(0);
    a aVar = ((SQLiteDataBaseActivity) this.f).s;
    if (aVar != null) {
        aVar.b("db");
    } else {
        g.k("presenter");
        throw null;
    }
} 


```

I had to check what type of variable is “s”, which is of type “v0.d.a.c.a.d.b.e.a”. It is being called with the this variable, which is a SQLiteDataBaseActivity:

In the constructor of a, the SQLiteDataBaseActivity is casted to b and set to the “b” attribute:

```
    public a(b bVar) {
        g.e(bVar, "activity");
        this.b = bVar;
    }


```

I searched where this variable was being used, and I found it on the following method:

```
public void a(r rVar) {
    r rVar2 = rVar;
    b bVar = this.a.b;
    g.d(rVar2, "result");
    bVar.b(rVar2);
}


```

which is being used in the following method (from v0.d.a.c.a.d.b.e.a class):

```
public final void b(String str) {
    g.e(str, "flag");
    a().c(v0.d.a.b.a.b.a().b(str).e(y0.a.o.a.a).a(y0.a.i.a.a.a()).b(new C0142a(this), new b(this)));
}


```

In the a method the application calls bVar.b(rVar2); which is the b method from SQLiteBaseActivity.

Whenever I went to the method, I found that at the end a toast is being generated with the following message: “Successfully added flag to SQLite.”. I saw that toast on the application when I press the button to retrieve the flag:

```
public void b(r rVar) {
    g.e(rVar, "response");
    ProgressBar progressBar = (ProgressBar) y(R.id.progress);
    g.d(progressBar, "progress");
    progressBar.setVisibility(8);
    v0.d.a.d.b.a aVar = new v0.d.a.d.b.a(this);
    String f = v0.a.a.a.a.f(rVar, "flag", "response.asJsonObject.get(\"flag\")", "response.asJsonObject.get(\"flag\").asString");
    o j = rVar.d().j("flag");
    g.d(j, "response.asJsonObject.get(\"flag\")");
    String substring = f.substring(1, j.g().length() - 1);
    g.d(substring, "(this as java.lang.Strin…ing(startIndex, endIndex)");
    g.e(substring, "data");
    StringBuilder sb = new StringBuilder("");
    int i = 0;
    while (i < substring.length()) {
        int i2 = i + 2;
        sb.append((char) v0.a.a.a.a.x(substring, i, i2, "(this as java.lang.Strin…ing(startIndex, endIndex)", 16, 16));
        i = i2;
    }
    String sb2 = sb.toString();
    g.d(sb2, "output.toString()");
    g.e(sb2, "flag");
    SQLiteDatabase writableDatabase = aVar.getWritableDatabase();
    ContentValues contentValues = new ContentValues();
    String str = v0.d.a.d.b.b.a;
    contentValues.put("flag", sb2);
    Toast.makeText(aVar.e, writableDatabase.insert("Flags", null, contentValues) == 0 ? "Failed to add flag to SQLite." : "Successfully added flag to SQLite.", 0).show();
    Toast.makeText(this, "Storage flag received successfully.", 1).show();
}


```

I did not try to analyze all the content on the method, just the instructions related to the database:

a. A databse instance is retrieved

```
SQLiteDatabase writableDatabase = aVar.getWritableDatabase();


```

b. A ContentValue instance is set with a key-pair “flag”, sb2. This class is used to execute updates and inserts. The keys are the name of the columns on the tables, and the values are set in the second parameter:

```
contentValues.put("flag", sb2);


```

c. Then the data is stored in the database:

```
Toast.makeText(aVar.e, writableDatabase.insert("Flags", null, contentValues) == 0 ? "Failed to add flag to SQLite." : "Successfully added flag to SQLite.", 0).show();


```

In order to find which database is being updated I needed to find the name of the file with the db. I found it in:

```
v0.d.a.d.b.a aVar = new v0.d.a.d.b.a(this);


```

I checked the constructor of “v0.d.a.d.b.a”, and I found the following:

```
public a(Context context) {
    super(context, "AndroidSecurity.db", (SQLiteDatabase.CursorFactory) null, 1);
    g.e(context, "context");
    String str = b.a;
    this.e = context;
}


```

In that method the AndroidSecurity.db file is the one being opened, so after finding that informations I had the name of the file AndroidSecurity.db, the name of the table (Flags) and the name of column (flag).

I had two alternatives now to solve the challenge. The first one was checking the content of the folder of the application, and the second one is using Frida. I started with the first one:

a- Search where was the file located (generally it is in the folder databases of the private folder of the application):

```
> find /data/data/com.hpandro.androidsecurity -name AndroidSecurity.db


```

which returns just the following result:

```
/data/data/com.hpandro.androidsecurity/databases/AndroidSecurity.db


```

I opened the database and read the table Flags:

```
sqlite3 /data/data/com.hpandro.androidsecurity/databases/AndroidSecurity.db


```

and then execute:

which returns:

```
1|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
2|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
3|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
4|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
5|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
6|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
7|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}
8|hpandro{sqlite.5xQcqhGNCVoXinSqZWJau1SXgdfX3oYp}


```

So each time I press the button that retrieves the flag, it adds a new row to the table.

At this point I had the challenge solved, but I got curious about where the flag was retrieved from. By checking the traffic sent and received from the servers I got the following content:

```
{
    "flag": "16870616e64726f7b73716c6974652e357851637168474e43566f58696e53715a574a617531535867646658336f59707d1"
}


```

I wanted to know how that hexa String was converted to the flag in the database. In the b method from SQLiteDataBaseActivity:

The first thing the application does is remove the first and the last numbers from the flag:

```
String substring = f.substring(1, j.g().length() - 1);


```

and then it iterates the String:

```
int i = 0;
while (i < substring.length()) {
    int i2 = i + 2;
    sb.append((char) v0.a.a.a.a.x(substring, i, i2, "(this as java.lang.Strin…ing(startIndex, endIndex)", 16, 16));
    i = i2;
}


```

In the while code the following method is called:

```
v0.a.a.a.a.x(substring, i, i2, "(this as java.lang.Strin…ing(startIndex, endIndex)", 16, 16)


```

where i is an even number (it starts with the value 0 and it is being incremented 2) and i2 = i + 2. Let’s check the following examples:

*   first time application gets in while: i = 0, i2 = 2,
*   second time application gets in while: i = 2, i2 = 4,

X method does the following:

```
public static int x(String str, int i, int i2, String str2, int i3, int i4) {
    String substring = str.substring(i, i2); <-- takes two digits of String (0,1 - 2,3 - 4,5)
    g.d(substring, str2);
    y0.a.n.a.h(i3);
    return Integer.parseInt(substring, i4); <-- converts it to a decimal number.
}


```

and then it casts it to a char. Let’s test this with Frida:

```
Java.perform( function () {
    var aClass = Java.use("v0.a.a.a.a");
    aClass.x.implementation = function (str, i, i2, str2, i3, i4) {
        console.log("input str="+str+" i=" + i + " i2=" + i2 + " i4=" + i4);
        var res = this.x(str, i, i2, str2, i3, i4);
        console.log("result= " + res);
        console.log("charat= " + String.fromCharCode(res));
        return res;
    }
});


```

which will return the characters from the flag. Then I created a script that retrieves the full flag by hooking the **SQLiteDatabase.insert** method:

```
Java.perform( function () {
    
    var SQLiteDatabase = Java.use("android.database.sqlite.SQLiteDatabase");
    var ContentValues = Java.use("android.content.ContentValues");
    var SetClass = Java.use("java.util.Set");
    var MapEntryClass = Java.use("java.util.Map$Entry");

    SQLiteDatabase.insert.implementation = function (table, otherVal, contentValues) {
        console.log("Valor insertado en tabla: " + table);
        var valueSetGeneric = contentValues.valueSet();
        var valueSet = Java.cast(valueSetGeneric,SetClass);
        var iterator = valueSet.iterator();
        while (iterator.hasNext()) {
            var mapEntry = Java.cast(iterator.next(),MapEntryClass);
            console.log(mapEntry.getKey() + "=" + mapEntry.getValue().toString());
        }
        return this.insert(table, otherVal, contentValues);
    }
});


```

### EncryptedSqliteDB

In this challenge I could not get to te task after pressing it:

![](https://cmrodriguez.me/images/featured-post/sqlite-encrypted.png)

What was weird initially, is that in the application whenever a task is not ready a Toast shows an error pointing that. I searched for the place where the SQLiteDataBaseActivity was being called (it had to be through an Intent), and in order to compare what was being done with it. I found the class “v0.d.a.c.b.c” which is a Fragment and it has the following code in the **onClick** method:

```
...
//this if with multiple or corresponds to root check exercises, which were not ready in this version (now they are ready though)
} else if (g.a(str2, y(R.string.root_management)) || g.a(str2, y(R.string.potentially_dangerous)) || g.a(str2, y(R.string.root_clocking)) || g.a(str2, y(R.string.text_keys)) || g.a(str2, y(R.string.dangerous_props)) || g.a(str2, y(R.string.busybox_binaries)) || g.a(str2, y(R.string.su_binary)) || g.a(str2, y(R.string.su_exists)) || g.a(str2, y(R.string.rw_system)) || g.a(str2, y(R.string.emulator_detection)) || g.a(str2, y(R.string.debugger_detection))) {
    return;
} else {
    //here the sqlite databse activity is called
    if (g.a(str2, y(R.string.sqlite_db))) {
        intent = new Intent(c(), SQLiteDataBaseActivity.class);
    //and here  is the issue for any other activity that is not the EncryptedDatabaseActivity navigate to an activity.
    } else if (!g.a(str2, y(R.string.sqlite_edb))) {
        if (g.a(str2, y(R.string.realm_db))) {
        ...
    } else {
        //click on EncryptedDatabaseActivity ends up here always, so the Activity is never called
        return;
    }


```

That method is the behavior from the Task button that evaluates the Activity to call based on the parameter sent to it when the onClick is executed. As the comments on the code explains there is no way to call the activity from the UI even when the Activity seems to be functional.

Here we have two alternatives:

a- Change the smali code and recompile the application in order to navigate to the Activity

b- Navigate to the activity with Frida. Again this could be done with two main strategies:

b.1- Change the behavior of the onClick method to create an Intent and send the application to the desired activity. b.2- Create some code that will not hook anything, but will force the application to navigate to the Activity.

Here is the code for the second alternative:

```
Java.perform(  function () {
    
    var Intent = Java.use("android.content.Intent");
    var String = Java.use("java.lang.String");
    

    var startIntent = Intent.$new();
    startIntent.setClassName(String.$new("com.hpandro.androidsecurity"),String.$new("com.hpandro.androidsecurity.ui.activity.task.datastorage.EncryptSQLiteDBActivity"));
    //myIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    startIntent.setFlags(0x10000000);
    Java.use('android.app.ActivityThread').currentApplication().startActivity(startIntent);
});


```

After we do that, the application shows the EncryptSQLiteDBActivity. The behavior of the Activity is like the one on the last challenge. The application gets the flag from the following URL:

GET [http://hpandro.raviramesh.info/flagstore.php?flag=edb](http://hpandro.raviramesh.info/flagstore.php?flag=edb)

which is obfuscated with the same mechanism as in the SQLite activity:

```
//este es el mismo script que en caso anterior para desencriptar. Se guarda en sb el resultado.
while (i < substring.length()) {
    int i2 = i + 2;
    sb.append((char) v0.a.a.a.a.x(substring, i, i2, "(this as java.lang.Strin…ing(startIndex, endIndex)", 16, 16));
    i = i2;
}

//no se hace nada con sb (g.d es una validación que se agrega por el uso de Kotlin)
g.d(sb.toString(), "output.toString()");
Toast.makeText(this, "Insecure Storage flag received successfully.", 1).show();
ProgressBar progressBar = (ProgressBar) y(R.id.progress);
g.d(progressBar, "progress");
progressBar.setVisibility(8);


```

We still want to solve this with Frida. It is a bit harder because we do not have a clear point where the output is being used (in the previous example it was the insert on the db. We’ll do the following:

hook the function that returns the flag:

```
String f = v0.a.a.a.a.f(rVar, "flag", "response.asJsonObject.get(\"flag\")", "response.asJsonObject.get(\"flag\").asString");


```

and overload it in order to decode the hexa String:

```
Java.perform( function () {
    
    var ClassToHook = Java.use("v0.a.a.a.a");
    var StringJava = Java.use("java.lang.String");
    
    var strFlag = StringJava.$new("flag"); 
    var strCommand = StringJava.$new("(this as java.lang.Strin…ing(startIndex, endIndex)");
            
    ClassToHook.f.implementation = function (rVar, str, str2, str3) {
        var returnValue = this.f(rVar, str, str2, str3);
        if (strFlag.equals(str)) {
            //procesdo el contenido
            var strResponse = returnValue.substring(1);
            var res = "";
            var i = 0;
            while (i < strResponse.length - 1) {
                var i2 = i + 2;
                var resInterno = ClassToHook.x(strResponse, i, i2, strCommand, 16, 16);
                res = res + String.fromCharCode(resInterno);
                i = i2;
            }
            console.log(res);
        }
        return returnValue;
    }
    
});        


```

During the stream people suggested to hook the following instruction:

```
g.d(sb.toString(), "output.toString()");


```

So the script that was created was the following one:

```
Java.perform( function () {
   var StringBuilder = Java.use("java.lang.StringBuilder");
   StringBuilder.toString.implementation = function () {
        var resultado = this.toString();
        //this if is just to show the content for the application when it has a flag 
        if (resultado.indexOf("hpandro{") >= 0) {
            console.log("resultado: " + resultado);
        }
        return resultado;
   } 
});


```

In this post we didn’t go too deep on SQLite Databases, and we played a bit more with Frida and reversing the application.

In the following link [https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%201](https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%201) you can find the application we used for this post

In the following repo you’ll have the link to the script used on the sqlite task: [https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%204](https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%204)

And in the following link you’ll have the link to the scripts used for the encrypted sqlite task: [https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%204](https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%204)