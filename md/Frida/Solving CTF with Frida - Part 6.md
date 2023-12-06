> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cmrodriguez.me](https://cmrodriguez.me/blog/hpandro-6/)

> frida ctf challenge emulator detection

In this series of posts I’ll be solving some persistence challenges from [hpandro ctf challenges](http://ctf.hpandro.raviramesh.info/). hpAndro created an Android application with multiple vulnerabilities, following the [MSTG](https://github.com/OWASP/owasp-mstg).

In this series I’ll be reviewing the Emulator Detection techniques implemented in the CTF. I’ll explain how each validation is executed and how to bypass them with Frida. I’ll show generic validations that could be used in any application regardless the type or level of obfuscation it has.

Emulation Detection techniques basically tries to find whether the application is running on an emulator by checking some properties the OS has, and comparing them with known emulation values (like generic phone numbers used by emulators or values on some system properties). In order to bypass the validations the scripts created need to intercept the functions that return the values from the phone and change them for ones that are not generic.

Note: In order to solve the challenges in a faster are easier way, you can follow the guide from [Solving CTF with Frida - Part 5](https://cmrodriguez.me/blog/hpandro-5/).

We have nine different challenges related to emulation detection mechanisms:

*   Virtual Phone Number validation.
*   Device ID validation.
*   Hardware Specifications.
*   QEMU check.
*   Emulator Files Check.
*   Emulator Default IP Check.
*   Package name Check.
*   Debug Flag.
*   Network Operator Name.

### Virtual Phone Number validation

This validation is executed in the **com.hpandro.androidsecurity.ui.activity.task.emulatorDetection.VirtualPhoneNumberActivity$init$2.onClick** method:

```
public final void onClick(View view) {
    //validates Shared Preference value
    String string = MainApp.Companion.getSharedPrefEmulatorDetection().getString("VirtualPhoneNumberStr", "00");
    //check is being executed here
    boolean checkPhoneNumber = new EmulatorDetector(this.this$0).checkPhoneNumber();
    Intrinsics.checkNotNull(string);
    if (!StringsKt.contains$default((CharSequence) string, (CharSequence) "F", false, 2, (Object) null) || checkPhoneNumber) {
        //shows error and returns
        ...
        return;
    }
    ...
    //retrieves virtual phone number flag
    VirtualPhoneNumberActivity.access$getPresenter$p(this.this$0).getEmulatorDetectFlag("vpn");
}


```

This method calls the EmulatorDetector.checkPhoneNumber, which gets the TelephonyManager, and gets the line number one, and then it checks if it is in a list of phone numbers recognized as emulators’ ones. The phone numbers are hardcoded in the variable PHONE_NUMBERS:

```
public final boolean checkPhoneNumber() {
    Object systemService = this.mContext.getSystemService("phone");
    if (systemService != null) {
        try {
            String line1Number = ((TelephonyManager) systemService).getLine1Number();
            for (String str : PHONE_NUMBERS) {
                if (StringsKt.equals(str, line1Number, true)) {
                    log(" check phone number is detected");
                    return true;
                }
            }
        } catch (Exception unused) {
            log("No permission to detect access of Line1Number");
        }
        return false;
    }
    throw new NullPointerException("null cannot be cast to non-null type android.telephony.TelephonyManager");
}


```

In this case in order to bypass this control I need to change the way the TelephonyManager returns the phone number. This can be done with the following script:

```
Java.perform( function () {
    var String = Java.use("java.lang.String");
    var TelephonyManager = Java.use("android.telephony.TelephonyManager");
    TelephonyManager.getLine1Number.overload().implementation = function () {
        var result = this.getLine1Number();
        console.log("entra a telephony manager " + result);
        //set a valid phone number
        return String.$new("152945677");
    }
}); 


```

### Device ID

In this case I could not get to the activity. So I went through the source code to see if it was implemented, and found that there were all the classes needed to execute it. I then called the activity directly from Frida:

```
Java.perform(  function () {
    
    var Intent = Java.use("android.content.Intent");
    var String = Java.use("java.lang.String");
    var Integer = Java.use("java.lang.Integer");
    var startIntent = Intent.$new();
    startIntent.setClassName(String.$new("com.hpandro.androidsecurity"),String.$new("com.hpandro.androidsecurity.ui.activity.task.emulatorDetection.DeviceIDsActivity"));
    startIntent.setFlags(0x10000000);
    Java.use('android.app.ActivityThread').currentApplication().startActivity(startIntent);
});


```

Whenever I did that the activity was opened, but whenever I pressed the button, nothing happened. I saw the content of the class and checked the onclickin the DeviceIDsActivity$init$2 class. In the onClick method from the DeviceIDsActivity$init$2 class I have the following code:

```
public final void onClick(View view) {
    String string = MainApp.Companion.getSharedPrefEmulatorDetection().getString("DeviceIDsStr", "00");
    boolean checkImsi = new EmulatorDetector(this.this$0).checkImsi();
    boolean checkDeviceId = new EmulatorDetector(this.this$0).checkDeviceId();
    Intrinsics.checkNotNull(string);
    if (!StringsKt.contains$default((CharSequence) string, (CharSequence) "F", false, 2, (Object) null) || (checkDeviceId && checkImsi)) {
        TextView textView = (TextView) this.this$0._$_findCachedViewById(R.id.tvDetails);
        Intrinsics.checkNotNullExpressionValue(textView, "tvDetails");
        textView.setVisibility(0);
        if (checkImsi) {
            TextView textView2 = (TextView) this.this$0._$_findCachedViewById(R.id.tvDetails);
            Intrinsics.checkNotNullExpressionValue(textView2, "tvDetails");
            textView2.setText(this.this$0.getString(R.string.device_id) + " detected as\n\n" + new EmulatorDetector(this.this$0).getDetectedImsi());
        }
        if (checkDeviceId) {
            TextView textView3 = (TextView) this.this$0._$_findCachedViewById(R.id.tvDetails);
            Intrinsics.checkNotNullExpressionValue(textView3, "tvDetails");
            textView3.setText(this.this$0.getString(R.string.device_id) + " detected as\n\n" + new EmulatorDetector(this.this$0).getDeviceId());
            return;
        }
        return;
    }


```

I checked the values that the “DeviceIDsStr”, checkImsi and checkDeviceId returned to see what happens:

```
Java.perform( function () {

    var String = Java.use("java.lang.String");
    var MainApp = Java.use("com.hpandro.androidsecurity.MainApp");
    console.log( MainApp.Companion.value.getSharedPrefEmulatorDetection().getString(String.$new("DeviceIDsStr"), String.$new("00")) );     

    //I need to get the context to call EmulatorDetector
    var context = Java.use('android.app.ActivityThread').currentApplication().getApplicationContext();

    var EmulatorDetector = Java.use("com.hpandro.androidsecurity.utils.emulatorDetection.EmulatorDetector");
    var emulatorDetector = EmulatorDetector.$new(context);
    console.log(emulatorDetector.checkImsi());
    console.log(emulatorDetector.checkDeviceId());
});


```

That script returned:

```
00000000000000
false
false


```

So based on the responses, what happens in the onClick method is that the app gets in the “if”, because the “DeviceIDsStr” does not have any “F” on the content, and it does not print any message because the two conditions of root detection are not fulfilled. So in this case to show the result we can change the method that validates the “F” in the Shared Preference file:

```
Java.perform( function () {
    
    var StringKt = Java.use("kotlin.text.StringsKt__StringsKt");
    StringKt.contains$default.overload('java.lang.CharSequence', 'java.lang.CharSequence', 'boolean', 'int', 'java.lang.Object').implementation = function (charSeq, charSeq2,boolZ, intI, objO) {
        if (charSeq2.toString().indexOf("F") >= 0) {
            console.log("entra");
            return true;
        }
        return this.contains$default(charSeq, charSeq2,boolZ, intI, objO);
    }
});


```

And after that I received the flag: **hpandro{emudid.FNvq8tpbMtDweNn7gsgazfz5FuO5rxrg}**

If you have an scenario where any of the two validations shown (imsi_ids and device_id) returns true, the following script can be used:

```
Java.perform( function () {
    var String = Java.use("java.lang.String");
    var TelephonyManager = Java.use("android.telephony.TelephonyManager");
    TelephonyManager.getDeviceId.overload().implementation = function () {
        //set a valid device id
        return String.$new("000000000000001");
    }

    TelephonyManager.getSubscriberId.overload().implementation = function () {
        //set a valid suscriber id
        return String.$new("310260000000001");
    }
});


```

You can change the values returned in each overloaded function to return a valid value taken from a cellphone.

### Hardware Specifications

In this case the values checked are the ones from the class Build. In order to bypass them we can change the values from generic ones related to the VM to valid ones from a phone, so there is no way to detect it.

The validation with genymotion threw three errors:

```
HARDWARE: vbox86
PRODUCT: vbox86p
BOARD:(has) unknown


```

We need to patch them. We could patch all the values as well, but in this case we’ll do it just with the ones that are detected by the application:

```
Java.perform( function () {
    var String = Java.use("java.lang.String");
    var Build = Java.use("android.os.Build");
    Build.HARDWARE.value = String.$new("anything");
    Build.PRODUCT.value = String.$new("anything");
    Build.BOARD.value = String.$new("anything");
});


```

Note that for doing this, the best moment is at the start of the execution of the application with early instrumentation.

### QEMU detection

This is similar to the case of DEVICEID. I could not get to the Activity, so I had to navigate with Frida to the component and then force the change of the validation of StringKt to true.

```
Java.perform(  function () {
    
    var Intent = Java.use("android.content.Intent");
    var String = Java.use("java.lang.String");
    var Integer = Java.use("java.lang.Integer");
    var startIntent = Intent.$new();
    startIntent.setClassName(String.$new("com.hpandro.androidsecurity"),String.$new("com.hpandro.androidsecurity.ui.activity.task.emulatorDetection.QEMUDetectionActivity"));
    startIntent.setFlags(0x10000000);
    Java.use('android.app.ActivityThread').currentApplication().startActivity(startIntent);
});


```

This detection has multiple validations:

```
public final void onClick(View view) {
    String string = MainApp.Companion.getSharedPrefEmulatorDetection().getString("QEMUStr", "00");
    boolean checkQEmuDrivers = new EmulatorDetector(this.this$0).checkQEmuDrivers();
    boolean checkQEmuProps = new EmulatorDetector(this.this$0).checkQEmuProps();
    boolean checkFiles = new EmulatorDetector(this.this$0).checkFiles(EmulatorDetector.Companion.getX86_FILES(), "X86");
    boolean checkFiles2 = new EmulatorDetector(this.this$0).checkFiles(EmulatorDetector.Companion.getPIPES(), "Pipes");
    Intrinsics.checkNotNull(string);
    if ((!StringsKt.contains$default((CharSequence) string, (CharSequence) "F", false, 2, (Object) null) || (checkQEmuDrivers && (checkQEmuProps || checkFiles))) && checkFiles2) {
        //show message error
    } else {
        //retrieve the flag
    }
}


```

**a) Check QEMU Drivers**

This control executes the validation by opening the **/proc/tty/drivers** and **/proc/cpuinfo** files and checking in the content if they have the word _goldfish_ on them (stored in the variable QEMU_DRIVERS):

```
public final boolean checkQEmuDrivers() {
    File[] fileArr = {new File("/proc/tty/drivers"), new File("/proc/cpuinfo")};
    for (int i = 0; i < 2; i++) {
        File file = fileArr[i];
        if (file.exists() && file.canRead()) {
            byte[] bArr = new byte[1024];
            try {
                FileInputStream fileInputStream = new FileInputStream(file);
                fileInputStream.read(bArr);
                fileInputStream.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
            String str = new String(bArr, Charsets.UTF_8);
            for (String str2 : QEMU_DRIVERS) {
                if (StringsKt.contains$default((CharSequence) str, (CharSequence) str2, false, 2, (Object) null)) {
                    log("Check QEmuDrivers is detected");
                    return true;
                }
            }
            continue;
        }
    }
    return false;
}


```

In this case I decided to modify the _FileInputStream_ in order to avoid returning a file wih _goldfish_ as content. Another strategy could be to change the File constructor and checking if the file being opened by the application is **/proc/tty/drivers** or **/proc/cpuinfo**. In those cases you could retrieve the content of the files returned by a real cellphone (to do this you could upload the file examples to /tmp).

```
Java.perform( function () {
    var FileIS = Java.use("java.io.FileInputStream");
    FileIS.read.overload('[B').implementation = function(byteArray) {
        console.log("entra");
        var retByteArray = this.read(byteArray);
        var str = String.fromCharCode.apply(String, byteArray);
        if (str.indexOf("goldfish") >= 0) {
            var index = str.indexOf("goldfish");
            byteArray[index] = "a".charCodeAt(0);
            byteArray[index+1] = "a".charCodeAt(0);
            byteArray[index+2] = "a".charCodeAt(0);
            byteArray[index+3] = "a".charCodeAt(0);
        }
        return retByteArray;
    }
});


```

To confirm the change you could intercept the following function:

```
Java.perform( function () {
    
    var StringKt = Java.use("kotlin.text.StringsKt__StringsKt");
    StringKt.contains$default.overload('java.lang.CharSequence', 'java.lang.CharSequence', 'boolean', 'int', 'java.lang.Object').implementation = function (charSeq, charSeq2,boolZ, intI, objO) {
        console.log("charSeq1: " + charSeq + " charSeq2: " + charSeq2);
        return this.contains$default(charSeq, charSeq2,boolZ, intI, objO);
    }
});


```

**b) checkQEmuProps**

In this case the validation is the following one:

```
public final boolean checkQEmuProps() {
    Property[] propertyArr = PROPERTIES;
    int i = 0;
    for (Property property : propertyArr) {
        String prop = getProp(this.mContext, property.getName());
        if (property.getSeek_value() == null && prop != null) {
            i++;
        }
        if (property.getSeek_value() != null) {
            Intrinsics.checkNotNull(prop);
            String seek_value = property.getSeek_value();
            Intrinsics.checkNotNull(seek_value);
            if (StringsKt.contains$default((CharSequence) prop, (CharSequence) seek_value, false, 2, (Object) null)) {
                i++;
            }
        }
    }
    if (i < 5) {
        return false;
    }
    log("Check QEmuProps is detected");
    return true;
}


```

which calls the following validation:

```
private final String getProp(Context context, String str) {
try {
    Class<?> loadClass = context.getClassLoader().loadClass("android.os.SystemProperties");
    Object invoke = loadClass.getMethod("get", String.class).invoke(loadClass, Arrays.copyOf(new Object[]{str}, 1));
    if (invoke != null) {
        return (String) invoke;
    }
    throw new NullPointerException("null cannot be cast to non-null type kotlin.String");
} catch (Exception unused) {
    return null;
}


```

In this case I reviewed the values retrieved by the SystemProperties in Genymotion and the only one I had to change was **ro.hardware**. Whenever you face this validation you might need to change another values as well:

```
Java.perform ( function () {
    var SystemProperties = Java.use("android.os.SystemProperties");
    var String = Java.use("java.lang.String");
    SystemProperties.get.overload('java.lang.String').implementation = function (str) {
        console.log(str + "= " + this.get(str));
        if (str == "ro.hardware") {
            return String.$new("vbox86");
        } else {
            return this.get(str);
        }
    }    
});


```

**c) checkFiles**

In this validation, the application checks if some files exist on the cellphone which only exist on some emulators. This validation is done with two different list of files:

```
boolean checkFiles = new EmulatorDetector(this.this$0).checkFiles(EmulatorDetector.Companion.getX86_FILES(), "X86");
boolean checkFiles2 = new EmulatorDetector(this.this$0).checkFiles(EmulatorDetector.Companion.getPIPES(), "Pipes");


```

The validation executed is the following one:

```
public final boolean checkFiles(String[] strArr, String str) {
    Intrinsics.checkNotNullParameter(strArr, "targets");
    Intrinsics.checkNotNullParameter(str, "type");
    for (String str2 : strArr) {
        if (new File(str2).exists()) {
            log("Check " + str + " is detected");
            return true;
        }
    }
    return false;
}


```

So the script generated modifies the behavior of the **exists** method on the class File:

```
Java.perform ( function () {
    var File = Java.use("java.io.File");
    File.exists.implementation = function () {
        var path_file = this.path.value; 
        if (path_file == "ueventd.android_x86.rc" || path_file == "x86.prop" || path_file == "ueventd.ttVM_x86.rc" || path_file == "init.ttVM_x86.rc" || path_file == "fstab.ttVM_x86" || path_file == "fstab.vbox86" || path_file == "init.vbox86.rc" || path_file == "ueventd.vbox86.rc" || path_file == "/dev/socket/qemud" || path_file == "/dev/goldfish_pipe" || path_file == "/dev/qemu_pipe") {
            return false;
        } else {
            console.log("DID NOT WORK: " + path_file);
        }
        return this.exists();
    }    
});


```

### Emulator Files Check Task

This validation is the same as the one used in the previous validation, but the files to control are other ones. I updated the same script, but extended it for the new files:

```
Java.perform ( function () {
    var File = Java.use("java.io.File");
    File.exists.implementation = function () {
        var path_file = this.path.value; 
        if (path_file == "ueventd.android_x86.rc" || path_file == "x86.prop" || path_file == "ueventd.ttVM_x86.rc" || path_file == "init.ttVM_x86.rc" || path_file == "fstab.ttVM_x86" || path_file == "fstab.vbox86" || path_file == "init.vbox86.rc" || path_file == "ueventd.vbox86.rc" || path_file == "/dev/socket/qemud" || path_file == "/dev/goldfish_pipe" || path_file == "/dev/qemu_pipe" || path_file == "/dev/socket/genyd" || path_file == "/dev/socket/baseband_genyd" || path_file == "/system/genymotion/wallpapers/genymotion.png" || path_file == "/system/bin/genybaseband" || path_file == "/system/vendor/bin/hw/android.hardware.health@2.0-service.genymotion") {
            return false;
        }
        return this.exists();
    }    
});


```

### Emulator Default Ip Check

In this case the application works on the netcfg command and searches for the ip address assigned to the app. We’ll change the way the content is returned in order to avoid the detection of the validation. The code of the validation is the following one:

```
public final boolean checkIp() {
    if (ContextCompat.checkSelfPermission(this.mContext, "android.permission.INTERNET") == 0) {
        String[] strArr = {"/system/bin/netcfg"};
        StringBuilder sb = new StringBuilder();
        try {
            ProcessBuilder processBuilder = new ProcessBuilder((String[]) Arrays.copyOf(strArr, 1));
            processBuilder.directory(new File("/system/bin/"));
            processBuilder.redirectErrorStream(true);
            Process start = processBuilder.start();
            Intrinsics.checkNotNullExpressionValue(start, "process");
            InputStream inputStream = start.getInputStream();
            byte[] bArr = new byte[1024];
            while (inputStream.read(bArr) != -1) {
                sb.append(new String(bArr, Charsets.UTF_8));
            }
            inputStream.close();
        } catch (Exception unused) {
        }
        String sb2 = sb.toString();
        Intrinsics.checkNotNullExpressionValue(sb2, "stringBuilder.toString()");
        log("netcfg data -> " + sb2);
        String str = sb2;
        if (!TextUtils.isEmpty(str)) {
            Object[] array = new Regex("\n").split(str, 0).toArray(new String[0]);
            if (array != null) {
                String[] strArr2 = (String[]) array;
                for (String str2 : strArr2) {
                    String str3 = str2;
                    if ((StringsKt.contains$default((CharSequence) str3, (CharSequence) "wlan0", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) str3, (CharSequence) "tunl0", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) str3, (CharSequence) "eth0", false, 2, (Object) null)) && (StringsKt.contains$default((CharSequence) str3, (CharSequence) IP, false, 2, (Object) null) || StringsKt.contains$default((CharSequence) str3, (CharSequence) IP1, false, 2, (Object) null))) {
                        log("Check IP is detected " + str2);
                        return true;
                    }
                }
            } else {
                throw new NullPointerException("null cannot be cast to non-null type kotlin.Array<T>");
            }
        }
    }
    return false;
}


```

The script created to bypass it is the following. I took an example of netcfg output, and changed it in order to avoid the validation. Then I uploaded the file to /tmp/test.txt:

```
Java.perform( function () {
    var ProcessBuilder = Java.use("java.lang.ProcessBuilder");
    ProcessBuilder.$init.overload('[Ljava.lang.String;').implementation = function (array) {
        if (array[0] == "/system/bin/netcfg") {
            var params = new Array();
            params[0] = "cat";
            params[1] = "/tmp/test.txt";
            return this.$init(params);
        } else {
            return this.$init(array);
        }
    }
});


```

### Check Package Name

In this case the validation executed is the following one:

```
public final boolean checkPackageNameDetect(Context context, String[] strArr) {
    Intrinsics.checkNotNullParameter(context, "mContext");
    Intrinsics.checkNotNullParameter(strArr, "additionalRootManagementApps");
    boolean z = false;
    ArrayList arrayList = new ArrayList(CollectionsKt.listOf(new String[]{"com.google.android.launcher.layouts.genymotion", "com.bluestacks", "com.bignox.app", "com.genymotion.genyd", "com.genymotion.systempatcher", "com.genymotion.tasklocker"}.toString()));
    if (strArr.length == 0) {
        z = true;
    }
    if (!z) {
        arrayList.addAll(Arrays.asList((String[]) Arrays.copyOf(strArr, strArr.length)));
    }
    return isAnyPackageFromListInstalled(context, arrayList);
}


```

which calls the following method:

```
private final boolean isAnyPackageFromListInstalled(Context context, List<String> list) {
    Intrinsics.checkNotNull(context);
    PackageManager packageManager = context.getPackageManager();
    Intrinsics.checkNotNullExpressionValue(packageManager, "mContext!!.packageManager");
    boolean z = false;
    for (String str : list) {
        try {
            packageManager.getPackageInfo(str, 0);
            System.out.println((Object) (">>>>>> " + str + " ROOT management app detected!"));
            z = true;
        } catch (PackageManager.NameNotFoundException unused) {
        }
    }
    return z;
}


```

If the application were not optimized the following script should work:

```
Java.perform( function () {

   var PackageManagerNotFound = Java.use("android.content.pm.PackageManager$NameNotFoundException");
   console.log(PackageManagerNotFound);
   var PackageManager = Java.use("android.content.pm.PackageManager");
   PackageManager.getPackageInfo.overload('java.lang.String', 'int').implementation = function (str, valInt) {
        if (str == "com.google.android.launcher.layouts.genymotion" || str == "com.bluestacks" || str == "com.bignox.app" || str == "com.genymotion.genyd" || str == "com.genymotion.systempatcher" || str == "com.genymotion.tasklocker") {
            throw PackageManagerNotFound.$new();
        } else {
            return this.getPackageInfo(str, valInt);
        }
   } 
});


```

But as in my case the application was installed in an OS with optimizations, the resolution of the PackageManager isn’t in the Java layer. I still do not know how to intercept this, so I leave the standard resolution used in the root detection blogpost:

```
Java.perform( function () {
    var EmulatorDetector = Java.use("com.hpandro.androidsecurity.utils.emulatorDetection.EmulatorDetector");
    EmulatorDetector.checkPackageNameDetect.implementation = function(context, strArr) {
        return false;
    }
});


```

### Debug Flag

This is an atypical validation, because it checks a flag that is in the **EmulatorDetector** class, called **isDebug**. This value does not seem to be set anywhere in the application, so I just created an script like the one in the previous example:

```
Java.perform( function () {
    var EmulatorDetector = Java.use("com.hpandro.androidsecurity.utils.emulatorDetection.EmulatorDetector");
    EmulatorDetector.isDebug.implementation = function() {
        return false;
    }
});


```

### Network Operator Name

In this case the validation is the following one:

```
public final boolean checkOperatorNameAndroid() {
    Object systemService = this.mContext.getSystemService("phone");
    if (systemService == null) {
        throw new NullPointerException("null cannot be cast to non-null type android.telephony.TelephonyManager");
    } else if (!StringsKt.equals(((TelephonyManager) systemService).getNetworkOperatorName(), "android", true)) {
        return false;
    } else {
        log("Check operator name android is detected");
        return true;
    }
}


```

This method uses the TelephonyManager as well, but it validates whether the NetworkOperatorName is **android**. As I already created scripts patching the TelephonyManager, I’ll not explain the solution:

```
Java.perform( function () {
    var String = Java.use("java.lang.String");
    var TelephonyManager = Java.use("android.telephony.TelephonyManager");
    TelephonyManager.getNetworkOperatorName.overload().implementation = function () {
        //set a valid device id
        return String.$new("android");
    }
});


```

### Conclusion

The controls shown in this application are commonly seen in production applications. As pentesters, it is important to understand them in order to find patterns in the applications we are analyzing to try to detect these controls and patch them in order to continue with the security analysis of the target. As a developer it is good to know how the controls work and how an attacker could try to bypass them.

In the blog I created several generic scripts that could work on any phone, so in case you find any validation related to emulation, you can use these in order to bypass them.