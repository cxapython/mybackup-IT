> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jdh5202.tistory.com](https://jdh5202.tistory.com/908)

> native 모든 context 조회 console.log(JSON.stringify(this.context) ) 특정 라이브러리 로드 후 주소값 후킹(android) Interc......

#### **native 모든 context 조회**

```
console.log( JSON.stringify(this.context) )

```

#### **특정 라이브러리 로드 후 주소값 후킹(android)**

```
Interceptor.attach(Module.getExportByName(null, "android_dlopen_ext"), {
  onEnter(args) {
    this.path = Memory.readUtf8String(args[0]);
  },
  onLeave(retval) {
    if ( /libfoo.so/i.test(this.path) ) {
      const target = this.path.split("/").pop();
      hook(target);
    }
  },
});

function hook(lib) {
  const base_adr = Module.findBaseAddress(lib);
  console.log("lib address => ", base_adr);

  Interceptor.attach(base_adr.add(0x100000), {
    onEnter(args) {
      console.log(this.context.x0);
    },
    onLeave(retval) {},
  });
}

```

#### **로딩되는 라이브러리 확인(android)**

```
Interceptor.attach(Module.findExportByName(null, 'android_dlopen_ext'), {
  onEnter: function (args) {
      this.path = Memory.readUtf8String(args[0]);
      console.log(" [+] android_dlopen_ext path : " + this.path);
  },
  onLeave: function (retval) {
  }
});



Java.perform(function() {
  const System = Java.use('java.lang.System');
  const Runtime = Java.use('java.lang.Runtime');
  const VMStack = Java.use('dalvik.system.VMStack');
  System.loadLibrary.implementation = function(library) {
      try {
          console.log('System.loadLibrary("' + library + '")');
          Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), library);
      } catch(ex) {
          console.log(ex);
      }
  };
});

```

#### **특정 라이브러리 내 로딩되는 함수 확인(android)**

```
Interceptor.attach(Module.getExportByName(null, 'android_dlopen_ext'), {
    onEnter (args) {
      this.lib = Memory.readCString(args[0]);
      console.log(">>>>>>>>>>>>>>>>>");
    },
    onLeave(retval){
        var lib_new = this.lib.toString().split('/').pop();
        if (/libso/i.test(lib_new)){
            const libc = Module.findBaseAddress(this.lib);

            var app_lib = Process.findModuleByName(lib_new).enumerateExports();
            app_lib.forEach(func => {
                Interceptor.attach(func.address, {
                    onEnter(a) {
                        console.warn(func.name);
                    }
                })
            })
                
        }
        
    }
});

```

#### **후킹 여부 확인**

```
const base_adr = Module.findBaseAddress('app');
let svcAddr = [0x100001, 0x100002];
  svcAddr.forEach((currentElement, index, array) => {
      Interceptor.attach(base_adr.add(currentElement), {
          onEnter(a) {   
              console.log('svc call addr:',currentElement.toString(16));
          }
      });
});

```

#### **현재 실행 중인 클래스 객체 및 변수 접근  
**

```
Java.perform(function () {
  var SpannableStringBuilder = Java.use("android.text.SpannableStringBuilder");
  var String = Java.use("java.lang.String");
  Java.choose("android.widget.EditText", {
    onMatch: function (instance) {
      console.log("Found instance: " + instance);
      console.log("[*] EditText RootView: " + instance.getRootView());
      console.log("[*] EditText InputType: " + instance.getInputType());
      console.log(
        "[*] EditText Text: " +
          Java.cast(instance.getText(), SpannableStringBuilder)
      );
      console.log(
        "[*] EditText Hint: " + Java.cast(instance.getHint(), String)
      );
    },
    onComplete: function () {
      console.log("[*] Finished heap search");
    },
  });
});

```

#### **루팅&탈옥 탐지 파일명 변경**

```
const base_adr = Module.findBaseAddress('app');
const address = [0x100001,0x100002,0x100003,0x100004];
address.filter((addr) => { 
    Interceptor.attach(base_adr.add(addr), { 
        onEnter(arg) {
            if ( /cydia/i.test(this.context.x0.readCString()) )
            {
                console.log("before: ", this.context.x0.readCString());
                Memory.writeUtf8String(this.context.x0,'/Applications/xxx.app');
                console.log("After: ", this.context.x0.readCString());
            }
        }
    });
});

```

#### **안티디버깅 우회**

```
Interceptor.attach(Module.findExportByName(null, "ptrace"), {

  onEnter: function(args) {
    var content = Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\\n') + '\\n';
    console.log("[ptrace]"+args[0]);
    console.log(content);

    if(args[0] == 0x1f){
      args[0]=ptr("0x0");
      console.log("Changed Argument value! Bypassing ptrace");
    }
  },
  onLeave: function(retval) {
    if (this.flag) // passed from onEnter
      console.log("\\nretval: " + retval);
  }
});

Interceptor.attach(Module.findExportByName(null, "sysctl"), {

  onEnter: function(args) {
    //this.a = Memory.readInt(args[0]);
    //this.b = Memory.readInt(args[0].add(ptr(4)));
    //this.c = Memory.readInt(args[0].add(ptr(8)));
    this.d = Memory.readInt(args[0].add(ptr(12))); // get pid

    var pid = Process.id;
    //console.log("pid: "+pid);
    
    if(this.d == pid){
      this.kp_proc = args[2];
      console.log("[sysctl] pid: "+this.d);
    }
    else
      this.kp_proc = null;
  },
  onLeave: function(retval){

    if(this.kp_proc != null){
      console.log(Memory.readByteArray(this.kp_proc.add(30), 16));
      Memory.writeByteArray(this.kp_proc.add(33), [0x40]);
      console.log(Memory.readByteArray(this.kp_proc.add(30), 16));
      console.log('Changed Memory value! Bypassing sysctl');
    }
  }
});

Interceptor.attach(Module.findExportByName(null, "getppid"), {

  onEnter: function(args) {
    var content = Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\\n') + '\\n';
    console.log("[getppid]");
    console.log(content);
  },
  onLeave: function(retval){
    if(retval!=0x01){
      var new_retval = prt("0x01");
      retval.replace(new_retval);
      console.log("Changed return value! Bypassing getppid")
    }
  }
});

```

#### **trace 및 Backtrace**

```
> frida-trace -U -f com.co.packages -m "*[classname *]"
ex) frida-trace -U -f com.direct -m "*[* *Jailbroken]"


console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n\t'));


var logContentArray = new Array();
var singlePrefix = "|-";
function uniqBy(array, key) {
  var seen = {};
  return array.filter(function (item) {
    var k = key(item);
    return seen.hasOwnProperty(k) ? false : (seen[k] = true);
  });
}

function traceClass(targetClass) {
  var hook = Java.use(targetClass);
  var methods = hook.class.getDeclaredMethods();
  hook.$dispose;

  var parsedMethods = [];
  methods.forEach(function (method) {
    parsedMethods.push(
      method
        .toString()
        .replace(targetClass + ".", "TOKEN")
        .match(/\sTOKEN(.*)\(/)[1]
    );
  });

  var targets = uniqBy(parsedMethods, JSON.stringify);
  targets.forEach(function (targetMethod) {
    traceMethod(targetClass + "." + targetMethod);
  });
}


function traceMethod(targetClassMethod) {
  var delim = targetClassMethod.lastIndexOf(".");
  if (delim === -1) return;

  var targetClass = targetClassMethod.slice(0, delim);
  var targetMethod = targetClassMethod.slice(
    delim + 1,
    targetClassMethod.length
  );
  var hook = Java.use(targetClass);
  var overloadCount = hook[targetMethod].overloads.length;
  console.log(
    "Tracing " + targetClassMethod + " [" + overloadCount + " overload(s)]"
  );
  for (var i = 0; i < overloadCount; i++) {
    hook[targetMethod].overloads[i].implementation = function () {
      var logContent_1 = "entered--" + targetClassMethod;
      var prefixStr = gainLogPrefix(logContentArray);
      logContentArray.push(prefixStr + logContent_1);
      console.log(prefixStr + logContent_1);

      for (var j = 0; j < arguments.length; j++) {
        var tmpLogStr = prefixStr + "arg[" + j + "]: " + arguments[j];
        console.log(tmpLogStr);
      }

      var retval = this[targetMethod].apply(this, arguments);
      var tmpReturnStr = prefixStr + "retval: " + retval;
      console.log(tmpReturnStr);
      var logContent_ex = "exiting--" + targetClassMethod;
      console.log(prefixStr + logContent_ex);
      return retval;
    };
  }
}

function gainLogPrefix(theArray) {
  var lastIndex = theArray.length - 1;
  if (lastIndex < 0) {
    return singlePrefix;
  }

  for (var i = lastIndex; i >= 0; i--) {
    var tmpLogContent = theArray[i];
    var cIndex = tmpLogContent.indexOf("entered--");
    if (cIndex == -1) {
      var cIndex2 = tmpLogContent.indexOf("exiting--");
      if (cIndex2 == -1) {
        continue;
      } else {
        var resultStr = tmpLogContent.slice(0, cIndex2);
        return resultStr;
      }
    } else {
      var resultStr = singlePrefix + tmpLogContent.slice(0, cIndex); 
      return resultStr;
    }
  }
  return "";
}

setTimeout(function () {
  Java.perform(function () {
    
    traceClass("com.xxx.Application");
  });
}, 0);



function trace(pattern)
{
var type = (pattern.indexOf(" ") === -1) ? "module" : "objc"; 
var res = new ApiResolver(type);
var matches = res.enumerateMatchesSync(pattern);
var targets = uniqBy(matches, JSON.stringify);

targets.forEach(function(target) {
if (type === "objc")
traceObjC(target.address, target.name);
else if (type === "module")
traceModule(target.address, target.name);
});
}


function uniqBy(array, key)
{
var seen = {};
return array.filter(function(item) {
var k = key(item);
return seen.hasOwnProperty(k) ? false : (seen[k] = true);
});
}


function traceObjC(impl, name)
{
console.log("Tracing " + name);

Interceptor.attach(impl, {

onEnter: function(args) {


this.flag = 0;

this.flag = 1;

if (this.flag) {
console.warn("\n[+] entered " + name);

console.log("\x1b[31mCaller:\x1b[0m \x1b[34m" + DebugSymbol.fromAddress(this.returnAddress) + "\x1b[0m\n");


console.log("\x1b[31margs[2]:\x1b[0m \x1b[34m" + args[2] + ", \x1b[32m" + ObjC.Object(args[2]) + "\x1b[0m")
console.log("\x1b[31margs[3]:\x1b[0m \x1b[34m" + args[3] + ", \x1b[32m" + ObjC.Object(args[3]) + "\x1b[0m")





}
},

onLeave: function(retval) {

if (this.flag) {

console.log("\n\x1b[31mretval:\x1b[0m \x1b[34m" + retval + "\x1b[0m");
console.warn("[-] exiting " + name);
}
}

});
}


function traceModule(impl, name)
{
console.log("Tracing " + name);

Interceptor.attach(impl, {

onEnter: function(args) {


this.flag = 0;



this.flag = 1;

if (this.flag) {
console.warn("\n*** entered " + name);


console.log("\nBacktrace:\n" + Thread.backtrace(this.context, Backtracer.ACCURATE)
.map(DebugSymbol.fromAddress).join("\n"));
}
},

onLeave: function(retval) {

if (this.flag) {

console.log("\nretval: " + retval);
console.warn("\n*** exiting " + name);
}
}

});
}


if (ObjC.available) {
trace("*[* *jail*]");







} else {
send("error: Objective-C Runtime is not available!");
}

```

#### **IOS logtrace**

```
function logtrace(ctx) {
  var content =
    Thread.backtrace(ctx.context, Backtracer.ACCURATE)
      .map(DebugSymbol.fromAddress)
      .join("\n") + "\n";
  if (
    content.indexOf("SubstrateLoader") == -1 &&
    content.indexOf("JavaScriptCore") == -1 &&
    content.indexOf("FLEXing.dylib") == -1 &&
    content.indexOf("NSResolveSymlinksInPathUsingCache") == -1 &&
    content.indexOf("MediaServices") == -1 &&
    content.indexOf("bundleWithPath") == -1 &&
    content.indexOf("CoreMotion") == -1 &&
    content.indexOf("infoDictionary") == -1 &&
    content.indexOf("objectForInfoDictionaryKey") == -1
  ) {
    console.log(content);
    return true;
  }
  return false;
}
function iswhite(path) {
  if (path == null) return true;
  if (path.startsWith("/var/mobile/Containers")) return true;
  if (path.startsWith("/var/containers")) return true;
  if (path.startsWith("/var/mobile/Library")) return true;
  if (path.startsWith("/var/db")) return true;
  if (path.startsWith("/private/var/mobile")) return true;
  if (path.startsWith("/private/var/containers")) return true;
  if (path.startsWith("/private/var/mobile/Library")) return true;
  if (path.startsWith("/private/var/db")) return true;
  if (path.startsWith("/System")) return true;
  if (path.startsWith("/Library/Preferences")) return true;
  if (path.startsWith("/Library/Managed")) return true;
  if (path.startsWith("/usr")) return true;
  if (path.startsWith("/dev")) return true;
  if (path == "/AppleInternal") return true;
  if (path == "/etc/hosts") return true;
  if (path == "/Library") return true;
  if (path == "/var") return true;
  if (path == "/private/var") return true;
  if (path == "/private") return true;
  if (path == "/") return true;
  if (path == "/var/mobile") return true;
  if (path.indexOf("/containers/Bundle/Application") != -1) return true;
  return false;
}
Interceptor.attach(Module.findExportByName(null, "access"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    console.log("access " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "creat"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("creat " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "dlopen"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (!iswhite(path)) console.log("dlopen " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "dlopen_preflight"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (!iswhite(path)) console.log("dlopen_preflight " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "faccessat"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[1].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("faccessat " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "getxattr"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("getxattr " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "link"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("link " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "listxattr"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("listxattr " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "lstat"), {
  block: false,
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("lstat " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "open"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = Memory.readUtf8String(args[0]);
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("open " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "opendir"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("opendir " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "__opendir2"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("opendir2 " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "readlink"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("readlink " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "realpath"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("realpath " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "realpath$DARWIN_EXTSN"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("realpath$DARWIN_EXTSN " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "stat"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("stat " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "statfs"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("statfs " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "symlink"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var path = args[0].readUtf8String();
    if (iswhite(path)) return;
    if (logtrace(this)) console.log("symlink " + path);
  },
});
Interceptor.attach(Module.findExportByName(null, "syscall"), {
  onEnter: function (args) {
    if (args[0].isNull()) return;
    var callnum = args[0].toInt32();
    if (callnum == 180) return;
    console.log("syscall " + args[0].toInt32());
  },
});

```

#### **IOS 값 조회**

```
-[ff01234a classname] →→→ ff01234a['- classname:'];


var className = "classname";
var methodName = "+ isJailBreak";
var hook_name = eval("ObjC.classes." + className + '["' + methodName + '"]');


var checkjail = ObjC.classes.classname;
var checkjail2 = checkjail['+ isJailBreak'];
Interceptor.attach(checkjail2.implementation, {  onEnter(a){ } });
  

const ret = new ObjC.Object( this.context.x0 );


ret = ObjC.classes.NSString.stringWithString_('abcd1234');

```

#### **NSString isEqualToString 후킹(IOS)** 

```
Interceptor.attach(ObjC.classes.NSString['- isEqualToString:'].implementation, {
    onEnter: function (args) {
        var str = new ObjC.Object(ptr(args[2])).toString()
        console.log('[+] Hooked -[NSString isEqualToString:] ->'+str);
    },onLeave: function (retval){
        console.log(retval)
    }
});

Interceptor.attach(ObjC.classes.__NSCFString['- isEqualToString:'].implementation, {
    onEnter: function (args) {
      var str = new ObjC.Object(ptr(args[2])).toString()
      console.log('[+] Hooked __NSCFString[- isEqualToString:] ->' , str);
    }
});

Interceptor.attach(ObjC.classes.NSTaggedPointerString['- isEqualToString:'].implementation, {
    onEnter: function (args) {
      var str = new ObjC.Object(ptr(args[2])).toString()
      console.log('[+] Hooked NSTaggedPointerString[- isEqualToString:] ->' , str);
}});

```

#### **JailBreak Bypass(ios)** 

```
var fileExistsAtPath = ObjC.classes.NSFileManager["- fileExistsAtPath:"];
var hideFile = 0;
var paths = [
	"/Applications/blackra1n.app",
    "/Applications/crackerxi.app",
    "/Applications/Cydia.app",
    "/Applications/FakeCarrier.app",
    "/Applications/Icy.app",
    "/Applications/IntelliScreen.app",
    "/Applications/MxTube.app",
    "/Applications/RockApp.app",
    "/Applications/SBSettings.app",
    "/Applications/WinterBoard.app",
    "/bin/bash",
    "/bin/sh",
    "/bin/su",
    "/etc/alternatives/sh",
    "/etc/apt/",
    "/etc/apt",
    "/etc/apt/sources.list.d/cydia.list",
    "/etc/apt/sources.list.d/electra.list",
    "/etc/apt/sources.list.d/sileo.sourcs",
    "/etc/apt/undecimus/undecimus.list",
    "/etc/ssh/sshd_config",
    "/jb/amfid_payload.dylib",
    "/jb/jailbreakd.plist",
    "/jb/libjailbreak.dylib",
    "/jb/lzma",
    "/jb/offsets.plists",
    "/Library/MobileSubstrate/CydiaSubstrate.dylib",
    "/Library/MobileSubstrate/DynamicLibraries/*",
    "/Library/MobileSubstrate/DynamicLibraries/LiveClock.plist",
    "/Library/MobileSubstrate/DynamicLibraries/Veency.plist",
    "/Library/MobileSubstrate/MobileSubstrate.dylib",
    "/pguntether",
    "/private/var/cache/apt",
    "/private/var/lib/apt",
    "/private/var/lib/cydia",
    "/private/var/log/syslog",
    "/private/var/mobile/Library/SBSettings/Themes",
    "/private/var/stash",
    "/private/var/tmp/cydia.log",
    "/private/var/tmp/frida-*.dylib",
    "/private/var/Users",
    "/System/Library/LaunchDaemons/com.ikey.bbot.plist",
    "/System/Library/LaunchDaemons/com.saurik.Cydia.Startup.plist",
    "/usr/bin/cycript",
    "/usr/bin/ssh",
    "/usr/bin/sshd",
    "/usr/lib/libjailbreak.dylib",
    "/usr/libexec/cydia/firmware.sh",
    "/usr/libexec/sftp-server",
    "/usr/libexec/sshd-keygen-wrapper",
    "/usr/libexec/ssh-keysign",
    "/usr/sbin/frida-server",
    "/usr/sbin/sshd",
    "/usr/share/jailbreak/injectme.plist",
    "/var/cache/apt",
    "/var/lib/apt",
    "/var/lib/cydia",
    "/var/lib/dpkg/info/mobilesubstrate.dylib",
    "/var/log/apt",
    "/var/mobile/Library/Caches/com.saurik.Cydia/sources.list",
    "/var/mobile/Media/.evasi0n7_installed",
    "/var/tmp/cydia.log",
    "/.bootstrapped_electra",
    "/.cydia_no_stash",
    "/.installed_unc0ver"
	];

Interceptor.attach(fileExistsAtPath.implementation, {
    onEnter: function(args) {
	var path = ObjC.Object(args[2]);

	if (paths.indexOf(path.toString()) > -1) {
	    console.log("Found Jailbreak Check: " + path.toString());
	    hideFile = 1;
	}},

    onLeave: function(retval) {
	if (hideFile) {
	    console.log("Return Value!!(0x0)");
	    retval.replace(0);
	    hideFile = 0;
	}}
});




var paths = [
     	"/Applications/Cydia.app",
     	"/Applications/blackra1n.app",
     	"/Applications/FakeCarrier.app",
     	"/Applications/Icy.app",
	"/Applications/IntelliScreen.app",
	"/Applications/MxTube.app",
	"/Applications/RockApp.app",
	"/Applications/SBSettings.app",
	"/Applications/WinterBoard.app",
	"/Library/MobileSubstrate/DynamicLibraries/LiveClock.plist",
	"/Library/MobileSubstrate/DynamicLibraries/Veency.plist",
	"/private/var/lib/apt",
	"/private/var/lib/apt/",
	"/private/var/lib/cydia",
	"/private/var/mobile/Library/SBSettings/Themes",
	"/private/var/stash",
	"/private/var/tmp/cydia.log",
	"/System/Library/LaunchDaemons/com.ikey.bbot.plist",
	"/System/Library/LaunchDaemons/com.saurik.Cydia.Startup.plist",
	"/usr/bin/sshd",
	"/usr/libexec/sftp-server",
	"/usr/sbin/sshd",
	"/etc/apt",
	"/bin/bash",
	"/Library/MobileSubstrate/MobileSubstrate.dylib"
];

try {
    var resolver = new ApiResolver('objc');

    resolver.enumerateMatches('*[* fileExistsAtPath*]', {
        onMatch: function(match) {
            var ptr = match["address"];
            Interceptor.attach(ptr, {
                onEnter: function(args) {
                    var path = ObjC.Object(args[2]).toString();
                    this.jailbreakCall = false;
                    for (var i = 0; i < paths.length; i++) {
                        if (paths[i] == path) {
                            this.jailbreakCall = true;
                        }
                    }
                },
                onLeave: function(retval) {
                    if (this.jailbreakCall) {
                        retval.replace(0x0);
                    }
                }
            });
        },
        onComplete: function() {}
    });

    resolver.enumerateMatches('*[* canOpenURL*]', {
        onMatch: function(match) {
            var ptr = match["address"];
            Interceptor.attach(ptr, {
                onEnter: function(args) {
                    var url = ObjC.Object(args[2]).toString();
                    this.jailbreakCall = false;
                    if (url.indexOf("cydia") >= 0) {
                        this.jailbreakCall = true;
                    }
                },
                onLeave: function(retval) {
                    if (this.jailbreakCall) {
                        retval.replace(0x0);
                    }
                }
            });
        },
        onComplete: function() {}
    });

    var response = {
        type: 'sucess',
        data: {
            message: "[!] Jailbreak Bypass success"
        }
    };
    send(response);
} catch (e) {
    var message = {
        type: 'exception',
        data: {
            message: '[!] Jailbreak bypass script error: '
        }
    };
    send(message);
}

```

#### **소켓 통신 확인**

```
setTimeout(function(){
    Java.perform(function(){
    
    var sock = Java.use("java.net.Socket");
    
    
    sock.bind.implementation = function(localAddress){
    console.log("Socket.bind("+localAddress.toString()+")");
    sock.bind.call(this, localAddress);
    }
    
    
    sock.connect.overload("java.net.SocketAddress").implementation = function(endPoint){
    console.log("Socket.connect("+endPoint.toString()+")");
    sock.connect.overload("java.net.SocketAddress").call(this, endPoint);
    }
    
    
    sock.connect.overload("java.net.SocketAddress", "int").implementation = function(endPoint, tmout){
    console.log("Socket.connect("+endPoint.toString()+", Timeout: "+tmout+")");
    sock.connect.overload("java.net.SocketAddress", "int").call(this, endPoint, tmout);
    }
    
    
    sock.getInetAddress.implementation = function(){
    ret = sock.getInetAddress.call(this);
    console.log(ret.toString()+" Socket.getInetAddress()");
    return ret;
    }
    
    
    sock.getInputStream.implementation = function(){
    console.log("Socket.getInputStream()");
    return sock.getInputStream.call(this);
    }
    
    
    sock.getOutputStream.implementation = function(){
    console.log("Socket.getOutputStream()");
    return sock.getOutputStream.call(this);
    }
    
    });
    },0);
    
    
    
    setTimeout(function(){
    Java.perform(function(){
    
    var sock = Java.use("java.net.Socket");
    
    
    
    
    sock.$init.overload().implementation = function(){
    console.log("new Socket() called");
    this.$init.overload().call(this);
    }
    
    
    sock.$init.overload("java.net.InetAddress", "int").implementation = function(inetAddress, port){
    console.log("new Socket('"+inetAddress.toString()+"', "+port+") called");
    this.$init.overload("java.net.InetAddress", "int").call(this, inetAddress, port);
    }
    
    
    sock.$init.overload("java.net.InetAddress", "int","java.net.InetAddress", "int").implementation = function(inetAddress, port, localInet, localPort){
    console.log("new Socket(RemoteInet: '"+inetAddress.toString()+"', RemotePort"+port+", LocalInet: '"+localInet+"', LocalPort: "+localPort+") called");
    this.$init.overload("java.net.InetAddress", "int","java.net.InetAddress", "int").call(this, inetAddress, port);
    }
    
    
    sock.$init.overload("java.net.Proxy").implementation = function(proxy){
    console.log("new Socket(Proxy: '"+proxy.toString()+"') called");
    this.$init.overload("java.net.Proxy").call(this, proxy);
    }
    
    
    sock.$init.overload("java.net.SocketImpl").implementation = function(si){
    console.log("new Socket(SocketImpl: '"+si.toString()+"') called");
    this.$init.overload("java.net.SocketImpl").call(this, si);
    }
    
    
    sock.$init.overload("java.lang.String", "int", "java.net.InetAddress", "int").implementation = function(host,port, localInetAddress, localPort){
    console.log("new Socket(Host: '"+host+"', RemPort: "+port+", LocalInet: '"+localInetAddress+"', localPort: "+localPort+") called");
    this.$init.overload("java.lang.String", "int", "java.net.InetAddress", "int").call(this, si);
    }
    
    
    });
},0);




"use strict";

var connect = new NativeFunction(
  Module.findExportByName(null, "connect"),
  "int",
  ["int", "pointer", "int"]
);

Interceptor.replace(
  connect,
  new NativeCallback(
    function (sockfd, addr, addrlen) {
      console.log(sockfd, addr, addrlen);
      var buf = Memory.readByteArray(addr, addrlen);
      console.log(
        hexdump(buf, { offset: 0, length: addrlen, header: true, ansi: true })
      );
      var result = connect(sockfd, addr, addrlen);
      console.log("connect()");
      var buf = Memory.readByteArray(addr, addrlen);
      console.log(
        hexdump(buf, { offset: 0, length: addrlen, header: true, ansi: true })
      );
      return result;
    },
    "int",
    ["int", "pointer", "int"]
  )
);

function hook_HttpUrlConnection() {
  Java.perform(function () {
    
    Java.use("java.net.URL").$init.overload("java.lang.String").implementation =
      function (str) {
        var result = this.$init(str);
        console.log("result , str => ", result, str);
        return result;
      };

    
    Java.use(
      "com.android.okhttp.internal.huc.HttpURLConnectionImpl"
    ).setRequestProperty.implementation = function (str1, str2) {
      
      var result = this.setRequestProperty(str1, str2);
      console.log(".setRequestProperty result,str1,str2->", result, str1, str2);

      return result;
    };

    Java.use(
      "com.android.okhttp.internal.huc.HttpURLConnectionImpl"
    ).setRequestMethod.implementation = function (str1) {
      var result = this.setRequestMethod(str1);
      console.log(".setRequestMethod result,str1,str2->", result, str1);
      return result;
    };
  });
}
setImmediate(hook_HttpUrlConnection);




var sendAddress = Module.findExportByName(null, "send");
console.log("send address: " + sendAddress.toString());
Interceptor.attach(sendAddress, {
  onEnter: function (args) {
    console.log(
      Thread.backtrace(this.context, Backtracer.ACCURATE)
        .map(DebugSymbol.fromAddress)
        .join("\n\t")
    );
    console.log("buf : " + Memory.readCString(args[1]));
    
    
    
    
  },
  onLeave: function (ret) {
    console.log(ret);
  },
});

```

#### **지문인증 우회(ios)** 

```
var lacontext = ObjC.classes['LAContext']
var localized = lacontext['- evaluatePolicy:localizedReason:reply:']

Interceptor.attach(localized.implementation, {
    onEnter: function(args) {
        var block = new ObjC.Block(args[4]);
        var callback = block.implementation;
        block.implementation = function (error, value)  {
            console.log("Touch ID Bypass")
            const result = callback(1, null);
            return result;
        }
    }
});

```

#### **클릭이벤트 후킹(android)** 

```
var jclazz = null;
var jobj = null;

function getObjClassName(obj) {
  if (!jclazz) {
    var jclazz = Java.use("java.lang.Class");
  }
  if (!jobj) {
    var jobj = Java.use("java.lang.Object");
  }
  return jclazz.getName.call(jobj.getClass.call(obj));
}

function watch(obj, mtdName) {
  var listener_name = getObjClassName(obj);
  var target = Java.use(listener_name);
  if (!target || !mtdName in target) {
    return;
  }

  target[mtdName].overloads.forEach(function (overload) {
    overload.implementation = function () {
      console.log("[WatchEvent] " + mtdName + ": " + getObjClassName(this));
      return this[mtdName].apply(this, arguments);
    };
  });
}

function OnClickListener() {
  Java.perform(function () {
    Java.use("android.view.View").setOnClickListener.implementation = function (
      listener
    ) {
      if (listener != null) {
        watch(listener, "onClick");
      }
      return this.setOnClickListener(listener);
    };

    Java.choose("android.view.View$ListenerInfo", {
      onMatch: function (instance) {
        instance = instance.mOnClickListener.value;
        if (instance) {
          console.log("mOnClickListener name is :" + getObjClassName(instance));
          watch(instance, "onClick");
        }
      },
      onComplete: function () {},
    });
  });
}
setImmediate(OnClickListener);

```

#### **화면 캡처 방지 비활성화(android)** 

```
var surface_view = Java.use('android.view.SurfaceView');

var set_secure = surface_view.setSecure.overload('boolean');

set_secure.implementation = function(flag){ 
    set_secure.call(false);
};

var window = Java.use('android.view.Window');
var set_flags = window.setFlags.overload('int', 'int');

var window_manager = Java.use('android.view.WindowManager');
var layout_params = Java.use('android.view.WindowManager$LayoutParams');

set_flags.implementation = function(flags, mask){
    flags =(flags.value & ~layout_params.FLAG_SECURE.value);
    set_flags.call(this, flags, mask);
};

```

#### **Java LOG 메서드 후킹(android)** 

```
var Log = Java.use("android.util.Log");

Log.d.overload('java.lang.String', 'java.lang.String').implementation = function (a, b) {
   console.log("Log.d() : "+a.toString()+" / "+b.toString());
   return this.d.overload('java.lang.String', 'java.lang.String').call(this, a, b);
};
Log.v.overload('java.lang.String', 'java.lang.String').implementation = function (a, b) {
   console.log("Log.v() : "+a.toString()+" / "+b.toString());
   return this.v.overload('java.lang.String', 'java.lang.String').call(this, a, b);
};

Log.i.overload('java.lang.String', 'java.lang.String').implementation = function (a, b) {
   console.log("Log.i() : "+a.toString()+" / "+b.toString());
   return this.i.overload('java.lang.String', 'java.lang.String').call(this, a, b);
};
Log.e.overload('java.lang.String', 'java.lang.String').implementation = function (a, b) {
   console.log("Log.e() : "+a.toString()+" / "+b.toString());
   return this.e.overload('java.lang.String', 'java.lang.String').call(this, a, b);
};
Log.w.overload('java.lang.String', 'java.lang.String').implementation = function (a, b) {
   console.log("Log.w() : "+a.toString()+" / "+b.toString());
   return this.w.overload('java.lang.String', 'java.lang.String').call(this, a, b);
};

```

#### **무결성 검증 우회(Android)**

```
Interceptor.attach(Module.findExportByName(null, "open" ), {
    onEnter: function(args) {
        this.hooked = Boolean(0);
        this.a = Memory.readCString(ptr(args[0])).toLowerCase();  
        console.log("[+] args[0] : " + this.a);

        if (this.a.indexOf("/data/app/com.app.app") !== -1) {
            var base_path = this.a.split("/");
            var prev = "/data/app/" + base_path[3] + "/base.apk";
            if(this.a == prev) {
                const file = "/data/local/tmp/base.apk"; 
                Memory.writeUtf8String(args[0], file); 
                console.log("[+] args[0] : " + this.a);
                console.log("integrity bypass") 
            }            
        }
    
    },
    onLeave: function(retval) {
        console.log("retval : " + retval)
        return retval;
    }
});

```

#### **Java String Build 메서드 후킹(android)** 

```
const StringBuilder = Java.use('java.lang.StringBuilder');
StringBuilder.toString.implementation = function () {
	var retVal = this.toString();
	console.log(retVal);
	return retVal;
}

```

#### **디렉터리 조회 함수 후킹(android)** 

```
Interceptor.attach(Module.findExportByName(null, "opendir"), {
    onEnter: function (args) {
        console.log("* dir : "+args[0].readCString());
    },
   onLeave: function (retval) {
   }
})

Interceptor.attach(Module.findExportByName(null, "readdir"), {
    onEnter: function (args) {
    
    },
   onLeave: function (retval) {
        var str =retval.add(19);
        try{
            console.log(str.readCString());
        }
        catch(err){
            console.log(" END");
        }
   }
})

```

#### **파일/문자열 관련 함수를 후킹하여 루팅 탐지 문자열이 존재하는지 확인(android)**

```
let funcs = [
    { name: 'open', arg: 0},
	{ name: 'fopen', arg: 1},
    { name: 'access', arg: 0},
    { name: 'strstr', arg: 1},
    { name: 'strcmp', arg: 1},
    { name: 'strcpy', arg: 1},
    { name: 'strtok', arg: 1},
	{ name: 'system', arg: 1},
    { name: 'opendir', arg: 0},   
    { name: 'scandir', arg: 0}   
  ]; 
let dup_chk;
  
funcs.filter( (func) => {  

    Interceptor.attach(Module.findExportByName(null, func['name']), {
        onEnter: function(args) {

          this.file_name = args[ func['arg'] ].readCString();
          if ( /frida|magisk|sbin\/su|\/proc\/self\/maps/i.test(this.file_name) )
          {
            const btc= Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n\t');

            if ( btc != dup_chk )
            {
                dup_chk = btc;
                console.warn("[*] ", func['name'], "-", this.file_name);
                console.log(btc);
            }
          }
        },
        onLeave: function(retval) { }
    });
});

```

![](https://blog.kakaocdn.net/dn/kW5Tq/btrrQspiXZS/1Sqecn9kYRhulsu0fqdQKK/img.png)

#### **로딩되는 파일 경로 후킹(ios)**

```
var fileExistsAtPath = ObjC.classes.NSFileManager["- fileExistsAtPath:"];
var hideFile = 0;

var jailbreakFiles = ["/Applications/Cydia.app",
		      "/bin/bash",
		      "/bin/sh",
		      "/etc/apt/sources.list.d/sileo.sources",
		      "/etc/apt/sillyo/sileo.sources",
		      "/Library/MobileSubstrate/MobileSubstrate.dylib",
		      "/usr/sbin/sshd",
		      "/etc/apt",
		      "/usr/bin/ssh"];

Interceptor.attach(fileExistsAtPath.implementation, {
    onEnter: function(args) {
	var path = ObjC.Object(args[2]);
      console.log(path);
	if (jailbreakFiles.indexOf(path.toString()) > -1) {
	    console.log("Checking jailbreak file: " + path.toString());
	    hideFile = 1;
	} 

    },
    onLeave: function(retval) {
	if (hideFile) {
	    console.log("Hiding jailbreak file...");
	    retval.replace(0);
	    hideFile = 0;
	}
    }
});

```

#### **앱 실행 시 로딩되는 모든 문자열 추출(ios)**

```
Interceptor.attach(ObjC.classes.NSString['+ stringWithUTF8String:'].implementation, {
onEnter: function (args) {
console.log('[+] Hooked +[NSString stringWithUTF8String:] ');
},
onLeave: function (retval) {
var str = new ObjC.Object(ptr(retval)).toString()

console.log("\nBacktrace:\n" + Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join("\n"));
console.log('[+] Returning [NSString stringWithUTF8String:] -> ', str);
return retval;
}
});

```

#### **리소스 함수 후킹(android)**

```
Java.perform(function () {
  const Resources = Java.use("android.content.res.Resources");
  const getStringMethod = Resources.getString.overload("int");
  getStringMethod.implementation = function (resId) {
    console.log("getString('" + resId.toString() + "') called");
    const ret = getStringMethod.call(this, resId);
    console.log("Value is: '" + ret + "'");
    console.log(
      Java.use("android.util.Log").getStackTraceString(
        Java.use("java.lang.Exception").$new()
      )
    );
    return ret;
  };
});

```

#### **native 함수 재정의**

```
 const base_adr = Module.findBaseAddress(lib);
 Interceptor.replace( base_adr.add(0x12345), new NativeCallback(
      function () {
        console.log("Bypass");
        return 0;
      }, "void", ["pointer"] ) 
);

```

#### **특정 클래스의 모든 메소드 확인(ios)**

```
console.log("[*] Started: Find All Methods of a Specific Class");
if (ObjC.available) {
  try {
    var className = "JailbreakDetectionVC";
    var methods = eval("ObjC.classes." + className + ".$methods");
    for (var i = 0; i < methods.length; i++) {
      try {
        console.log("[-] " + methods[i]);
      } catch (err) {
        console.log("[!] Exception1: " + err.message);
      }
    }
  } catch (err) {
    console.log("[!] Exception2: " + err.message);
  }
} else {
  console.log("Objective-C Runtime is not available!");
}
console.log("[*] Completed: Find All Methods of a Specific Class");

```

#### **open / exit native 함수 후킹(ios)**

```
var openPtr = Module.getExportByName(null, 'open');
var open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
Interceptor.replace(openPtr, new NativeCallback(function (pathPtr, flags) {
  var path = pathPtr.readUtf8String();
  console.log('Opening "' + path + '"');
  var fd = open(pathPtr, flags);
  console.log('Got fd: ' + fd);
  return fd;
}, 'int', ['pointer', 'int']));


var openPtr = Module.getExportByName(null, 'exit');
  var open = new NativeFunction(openPtr, 'void', ['int']);
  Interceptor.replace(openPtr, new NativeCallback(function (flags) {
    console.log('Got fd: ');
  }, 'void', ['int']));

```

#### **exit 함수 후킹(ios)**

```
if (ObjC.available)
{
    try
    {
        var example = ObjC.classes.example;
        var systemExit = example['- systemExit:'];
        var systemExitImpl = systemExit.implementation;
        systemExit.implementation = ObjC.implement(systemExit, function(handle, selector){
            console.log('muffin!');
        });
    }
    catch(err)
    {
        console.log("[!] Exception2: " + err.message);
    }
}
else
{
    console.log("Objective-C Runtime is not available!");
}


var exit = Module.findExportByName('libSystem.B.dylib', 'exit');

Interceptor.attach(exit, {
        onEnter: function (args) {
            console.log("[*] exit Call");
            console.log('\tBacktrace:\n\t' + Thread.backtrace(this.context,
                Backtracer.ACCURATE).map(DebugSymbol.fromAddress)
                .join('\n\t'));
        },
        onLeave: function (retval) {
        }
});


if (ObjC.available)
{
  const base_adr = Module.findBaseAddress('app');
  let svcAddr = [0xB329CA]; 
    svcAddr.forEach((currentElement, index, array) => {
        Interceptor.attach(base_adr.add(currentElement), {
            onEnter(a) {   
            console.log('svc call addr:',currentElement.toString(16));
            console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n\t'));
            }
        });
  });
}

```

#### **현재 앱에서 로드된 className 및 Method Name 출력(ios)**

```
if(ObjC.available){
    Object.keys(ObjC.classes).forEach(function(className){
        console.log(className);
        var methods = eval("ObjC.classes['"+className+"'].$methods");
        for(var i = 0; i<methods.length; i++){
            console.log(methods[i]);
        }
    });
}

```

#### **특정 메서드 실행 시 후킹(ios)**

```
var arguType = ObjC.classes['Class Name']['Method Name'].argumentTypes;
 

var retType = ObjC.classes['Class Name']['Method Name'].returnType;



if(ObjC.available){
    console.log("Running Hooking");
    var rooting = ObjC.classes['ANSMetadata'];
    var meta = rooting['- setValueInDictionary:withKey:toObject:'];
    var metafunc = meta.implementation;
    meta.implementation = ObjC.implement(meta,function(handler, selector, arg1, arg2, arg3){
        if(ObjC.Object(arg2).toString()=="jailbroken"){
            var arg3 = ObjC.classes.NSString.stringWithString_("0");
        }
        metafunc(handler,selector,arg1, arg2, arg3);
    })
}

 



if(ObjC.available){
    console.log("Running Hooking");
    var meta = ObjC.classes['ANSMetadata']['- setValueInDictionary:withKey:toObject:'];
    Interceptor.attach(meta.implementation,{
        onEnter: function(args){
            console.log("onEnter");
        },
        onLeave: function(retval){
        	retval.replace(0x1)
        }
    })    
}
var baseAddr = Module.findBaseAddress('file_name');
var sub1 = baseAddr.add(0x145214);
Interceptor.attach(sub1, {
    onEnter: function (args) {
    	console.log("sub_145214 IN");
    },
    onLeave: function (retval) {
        console.log("sub_145214 OUT");
    }
})
 



var base = Module.findBaseAddress("Jailbreak");
var check = base.add(0x70EC)

Interceptor.replace(check, new NativeCallback(function(){
    console.log("Check IN");
    return 1;
},'int',[]));
 



var base = Module.findBaseAddress("file Name")


var sub3 = base.add(0xEFF3C)
Interceptor.attach(sub3,{
    onEnter:function(args){
        
        
        console.log("Correct Password : "+this.context.x0)
        
    }
})


var sub4 = base.add(0xEFF40)
Interceptor.attach(sub4,{
    onEnter:function(args){
        this.context.x0 = 0x01
        console.log("Bypass Lock")
  }
})

```

#### **libc.so 우회(android)**

```


var RootPackages = ["com.noshufou.android.su", "com.noshufou.android.su.elite", "eu.chainfire.supersu", "com.koushikdutta.superuser", "com.thirdparty.superuser", "com.yellowes.su", "com.koushikdutta.rommanager", "com.koushikdutta.rommanager.license", "com.dimonvideo.luckypatcher", "com.chelpus.lackypatch", "com.ramdroid.appquarantine", "com.ramdroid.appquarantinepro", "com.devadvance.rootcloak", "com.devadvance.rootcloakplus", "de.robv.android.xposed.installer", "com.saurik.substrate", "com.zachspong.temprootremovejb", "com.amphoras.hidemyroot", "com.amphoras.hidemyrootadfree", "com.formyhm.hiderootPremium", "com.formyhm.hideroot", "me.phh.superuser", "eu.chainfire.supersu.pro", "com.kingouser.com", "com.android.rooting.apk" ]; var RootBinaries = ["su", "busybox", "supersu", "Superuser.apk", "KingoUser.apk", "SuperSu.apk", "rooting.apk", "Magisk", "temprootremovejb", "su-backup"];

Interceptor.attach(Module.findExportByName("libc.so", "fopen"), {
  onEnter: function (args) {
    var path = Memory.readCString(args[0]);
    console.log("fopen >> " + path.toString());
    path = path.split("/");
    var executable = path[path.length - 1];
    var shouldFakeReturn = RootBinaries.indexOf(executable) > -1;
    if (shouldFakeReturn && path.indexOf("ahnlab") === -1) {
      Memory.writeUtf8String(args[0], "/notexists");
      send("Bypass native fopen");
    }
  },
  onLeave: function (retval) {},
});
Interceptor.attach(Module.findExportByName("libc.so", "system"), {
  onEnter: function (args) {
    var cmd = Memory.readCString(args[0]);
    if (
      cmd.indexOf("getprop") != -1 ||
      cmd == "mount" ||
      cmd.indexOf("build.prop") != -1 ||
      cmd == "id"
    ) {
      send("Bypass native system: " + cmd);
      Memory.writeUtf8String(args[0], "grep");
    }
    if (cmd == "su") {
      send("Bypass native system: " + cmd);
      Memory.writeUtf8String(
        args[0],
        "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled"
      );
    }
  },
  onLeave: function (retval) {},
});
Interceptor.attach(Module.findExportByName("libc.so", "strstr"), {
  onEnter: function (args) {
    var cstr = Memory.readCString(args[1]);
    console.log("strstr >> " + cstr.toString());
    var shouldFakeReturn = RootBinaries.indexOf(cstr) > -1;
    if (shouldFakeReturn) {
      Memory.writeUtf8String(args[1], "/notexists");
      send("Bypass native strstr");
    }
  },
  onLeave: function (retval) {},
});
Interceptor.attach(Module.findExportByName("libc.so", "strtok"), {
  onEnter: function (args) {
    var cstr = Memory.readCString(args[1]);
    console.log("strtok >> " + cstr.toString());
    send(cstr);
  },
  onLeave: function (retval) {},
});
Interceptor.attach(Module.findExportByName("libc.so", "open"), {
  onEnter: function (args) {
    
    var cstr = Memory.readCString(args[0]);
    console.log("fopen >> " + cstr);
  },
  onLeave: function (retval) {
    
  },
});

Interceptor.attach(Module.findExportByName("libc.so", "access"), {
  onEnter: function (args) {
    
    var cstr = Memory.readCString(args[0]);
    console.log("access >> " + cstr);
    
    var shouldFakeReturn = RootAccess.indexOf(cstr) > -1;
    if (shouldFakeReturn) {
      var buf = Memory.allocUtf8String("/notexists");
      
      this.buf = buf;
      args[0] = buf;
      
    }
  },
  onLeave: function (retval) {
    
    
  },
});

```

#### **환경변수를 이용한 공유라이브러리 후킹**

```
Interceptor.attach(Module.findExportByName("libc.so", "dlsym"), {
    onEnter: function(args) {
        console.log(Memory.readUtf8String(args[1]));
    },
});

```

#### **메모리 덤프**

```
function hexDump(dump_addr) {
    const module_base = Module.findBaseAddress("libfoo.so");
    const dump_realaddr = module_base.add(dump_addr);
 
    console.warn("\nhexdump: " + ptr(dump_addr));
    console.log(hexdump(dump_realaddr, { offset: 0, length: 32 }));
}
hexDump(0x15038);

```