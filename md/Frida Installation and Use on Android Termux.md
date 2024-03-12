> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.com](https://github.com/frida/frida/discussions/2411)

---------------------------------------------------Installation Part-----------------------------------------------------------------

Step 1 : Update your termux to latest version. Don't install from play store.  
use from f-droid ( [https://f-droid.org/repo/com.termux_118.apk](https://f-droid.org/repo/com.termux_118.apk)) or  
github release ( [https://github.com/termux/termux-app/releases/download/v0.118.0/termux-app_v0.118.0+github-debug_universal.apk](https://github.com/termux/termux-app/releases/download/v0.118.0/termux-app_v0.118.0+github-debug_universal.apk) )

Step 2 : Give storage permission to termux manually  
long click on icon -> select App Info -> permission and give storage permission.  
Alternatively go to android settings -> installed apps -> search for termux -> permission and give storage permission.

Step 3 : Update dependency in termux

```
pkg update
pkg upgrade

```

Step 3: Install necessary tools needed for frida or (radare2 if you want) installation.

```
pkg install build-essential python python-pip git wget binutils openssl

```

Step 4 : Download Frida Core DevKit according to device architecture from  
[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)  
As of writing this, 16.0.19 is the latest frida version -  
for arm device - [https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-arm.tar.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-arm.tar.xz)  
for arm64 device - [https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-arm64.tar.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-arm64.tar.xz)  
for 32 bit emulator/device - [https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-x86.tar.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-x86.tar.xz)  
for 64 bit emulators/device - [https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-x86_64.tar.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-core-devkit-16.0.19-android-x86_64.tar.xz)

Step 5 : Extract it on internal storage (don't extract it in /data/local/tmp). for demo purpose I extracted it into

```
/sdcard/devkit 

```

folder. It should have 4 files -  
frida-core-example.c  
frida-core.h  
frida-core.gir  
libfrida-core.a  
[![](https://user-images.githubusercontent.com/27184655/218310554-cafe1b24-f369-4397-9674-efeb18a886f6.jpg)](https://user-images.githubusercontent.com/27184655/218310554-cafe1b24-f369-4397-9674-efeb18a886f6.jpg)

Step 6: Set proper environment variables, in this case type

```
export FRIDA_CORE_DEVKIT=/sdcard/devkit/

```

if you extracted it on some other location, use that path.  
make sure you run above command without root/su.

Step 7: Install frida and its commandline tools from pip

```
pip install frida frida-tools

```

[![](https://user-images.githubusercontent.com/27184655/218310575-89d7d2c0-028d-4942-a5ea-edc96461d55f.jpg)](https://user-images.githubusercontent.com/27184655/218310575-89d7d2c0-028d-4942-a5ea-edc96461d55f.jpg)

Now Frida is available to use from commandline

---------------------------------------------------Installation Part End-----------------------------------------------------------------

---------------------------------------------------Frida-Server Setup-----------------------------------------------------------------

Step 1: Download appropriate frida server according to your device architecture.  
[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)  
As of writing this, 16.0.19 is the latest frida version -  
for arm device - [https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-arm.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-arm.xz)  
for arm64 device - [https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-arm64.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-arm64.xz)  
for 32 bit emulator/device - [https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-x86.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-x86.xz)  
for 64 bit emulator/device - [https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-x86_64.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-server-16.0.19-android-x86_64.xz)

Step 2: Extract this archive into /data/local/tmp/ (**Remember - Extract it , Don't copy/move it there**, Some guys just move it to tmp folder and try to execute it so they get error like **Not executable: Magic FD37**)

Step 3: Rename it to **frida-server** without any extension. (You can pick any name but some guys later complain that ./frida-server command is not working so remeber the name whatever you choosen).

Step 4: Give executable permission to frida-server. Follow below step if you don't know how to do it.

1.  Open MT Manager and navigate to /data/local/tmp
2.  Long press on frida-server and select property

[![](https://user-images.githubusercontent.com/27184655/234478891-7e761fa5-c364-4e4c-9da3-2863f4bd94c2.png)](https://user-images.githubusercontent.com/27184655/234478891-7e761fa5-c364-4e4c-9da3-2863f4bd94c2.png)  
3. Now click on modify in front of Permissions

[![](https://user-images.githubusercontent.com/27184655/234479468-d026cacd-3895-4f73-8fa7-4943878d2e48.png)](https://user-images.githubusercontent.com/27184655/234479468-d026cacd-3895-4f73-8fa7-4943878d2e48.png)

4.  Now select all the checkbox and click on OK

[![](https://user-images.githubusercontent.com/27184655/234479151-8260ffb5-5bcb-4d2e-b727-dae68f02842c.png)](https://user-images.githubusercontent.com/27184655/234479151-8260ffb5-5bcb-4d2e-b727-dae68f02842c.png)  
---------------------------------------------------Frida-Server Setup End-----------------------------------------------------------------

**How to use Frida in Termux**

Step 1: Start Termux and type

```
su

```

Step 2: type

```
cd /data/local/tmp

```

Step 3: type

```
./frida-server -l 127.0.0.1

```

like this picture  
[![](https://user-images.githubusercontent.com/27184655/234481843-0d8c16a7-a006-4d47-b1a7-6e43d4362610.png)](https://user-images.githubusercontent.com/27184655/234481843-0d8c16a7-a006-4d47-b1a7-6e43d4362610.png)

Step 4: Left swipe on termux and start a new session  
[![](https://user-images.githubusercontent.com/27184655/234482110-620ad448-c525-4857-8837-92db009e6bdc.png)](https://user-images.githubusercontent.com/27184655/234482110-620ad448-c525-4857-8837-92db009e6bdc.png)

Step 5: type

```
cd /data/local/tmp

```

Step 6: type following command to spawn app with frida with your script

```
frida -H 127.0.0.1 -f com.your.app -l script.js

```

[![](https://user-images.githubusercontent.com/27184655/234482447-ed3e755d-ff72-488b-accb-9f77f1ac88f0.png)](https://user-images.githubusercontent.com/27184655/234482447-ed3e755d-ff72-488b-accb-9f77f1ac88f0.png)

If you want to run **frida-trace** then also similar command will work.  
Note - **Don't try to run frida-trace when you are in tmp directory as it will create handler folder and due to permission it will throw error. So we navigate to home directory to run it**

```
cd ~
frida-trace -H 127.0.0.1 -f com.your.app -i open

```

[![](https://user-images.githubusercontent.com/27184655/234516596-260ac9f4-3426-4290-9b4a-305c42f59af0.png)](https://user-images.githubusercontent.com/27184655/234516596-260ac9f4-3426-4290-9b4a-305c42f59af0.png)

**How to use Frida Inject in Termux**

Step 1: Download appropriate Frida Inject Binary according to your device architecture.  
[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)  
As of writing this, 16.0.19 is the latest frida version -  
for arm device - [https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-arm.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-arm.xz)  
for arm64 device - [https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-arm64.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-arm64.xz)  
for 32 bit emulator/device - [https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-x86.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-x86.xz)  
for 64 bit emulator/device - [https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-x86_64.xz](https://github.com/frida/frida/releases/download/16.0.19/frida-inject-16.0.19-android-x86_64.xz)

Step 2:  
Extract this archive into /data/local/tmp/ (**Remember - Extract it , Don't copy/move it there**, Some guys just move it to tmp folder and try to execute it so they get error like **Not executable: Magic FD37**)

Step 3: Rename it to **frida** without any extension. (You can pick any name but some guys later complain that ./frida- command is not working so remeber the name whatever you choosen).  
Here is a screenshot of device when a guy try to run **frida** without downloading anything(come on man, You are a frida user , have some common sense).  
[![](https://user-images.githubusercontent.com/27184655/236178067-a488ec16-6b57-4551-988e-e75ec9c2894b.png)](https://user-images.githubusercontent.com/27184655/236178067-a488ec16-6b57-4551-988e-e75ec9c2894b.png)  
To avoid such issue, make sure you download frida inject first and keep following steps.

Step 4: Make it executable, read this whole discussion to find out how to do it.

Step 5: Copy/Move your script to /data/local/tmp. It is not mandatory to have this path but then some guy will come and complain that script.js is not found

Step 6: Use frida as usual

```
su
cd /data/local/tmp
./frida -f com.app -s script.js

```

------------------------------------------------FAQ------------------------------------------------------------------------  
Q1. I am getting error with **--no-pause** command with frida  
A1. **--no-pause** command is removed from frida in 16.x so don't use it.

Q2. What is alternative to **--no-pause**  
A2. as **--no-pause** is removed and new command **--pause** is introduced, don't mix it uses.  
--no-pause == don't pause the process and directly run the script.  
--pause == pause the process and let you decide when to resume with **%resume** command

Q3. frida command is not found  
A3. You not have frida installed from pip. Follow above guide to install it properly

Q4. frida-ps command is not working.  
A4. frida-ps require frida-server so start the server in tmp folder with

```
su
cd /data/local/tmp
./frida-server -l 127.0.0.1

```

and then

```
frida-ps -ai -H 127.0.0.1
frida-ps -H 127.0.0.1

```

etc commands will work.

 

I managed to follow all the steps and installed the latest Frida in Termux.

How is it supposed to be used? I get

```
$ frida-ps
Failed to enumerate processes: unable to find process with name 'system_server'.


```

 

> I managed to follow all the steps and installed the latest Frida in Termux.
> 
> How is it supposed to be used? I get
> 
> ```
> $ frida-ps
> 
> Failed to enumerate processes: unable to find process with name 'system_server'.
> 
> 
> 
> ```

You probably forgot to run frida-server as root

 

Hi. I'm trying to run Frida on Termux on my device. I have managed to install it, but when I try to run it I get the following error:

Failed to load the Frida native extension: dlopen failed: cannot locate symbol "__atomic_is_lock_free" referenced by "data/data/com. termux/files/usr/lib/python3.11/site-packages/_frida.abi3.so... Please ensure that the extension was compiled correctly.

I'm running it on a Samsung Galaxy Tab 3 10.1 with custom Lineage OS 14.1 on Android 7.1.2. It's x86 system.

Any suggestions please?

 

Try To Use Old Versions of python like Version 3.10 or 3.9 OR Use Pc to install it on Termux

[…](#) On Sun, Jul 23, 2023, 12:01 AM black8855 ***@***.***> wrote: Hi. I'm trying to run Frida on Termux on my device. I have managed to install it, but when I try to run it I get the following error: Failed to load the Frida native extension: dlopen failed: cannot locate symbol "__atomic_is_lock_free" referenced by "data/data/com. termux/files/usr/lib/python3.11/site-packages/_frida.abi3.so... Please ensure that the extension was compiled correctly. I'm running it on a Samsung Galaxy Tab 3 10.1 with custom Lineage OS 14.1 on Android 7.1.2. It's x86 system. Any suggestions please? — Reply to this email directly, view it on GitHub <[#2411 (comment)](https://github.com/frida/frida/discussions/2411#discussioncomment-6518351)>, or unsubscribe <[https://github.com/notifications/unsubscribe-auth/A3TZGBUZPRZ2TGMQFXNOE4DXRQ5UBANCNFSM6AAAAAAUZIYL4A](https://github.com/notifications/unsubscribe-auth/A3TZGBUZPRZ2TGMQFXNOE4DXRQ5UBANCNFSM6AAAAAAUZIYL4A)> . You are receiving this because you commented.Message ID: ***@***.***>

 

can someone tell me how to fix this

[![](https://private-user-images.githubusercontent.com/13099434/289311768-0587b922-f8cc-4afa-89ef-abb4abdd6742.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTAyNDQ3OTcsIm5iZiI6MTcxMDI0NDQ5NywicGF0aCI6Ii8xMzA5OTQzNC8yODkzMTE3NjgtMDU4N2I5MjItZjhjYy00YWZhLTg5ZWYtYWJiNGFiZGQ2NzQyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMTIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzEyVDExNTQ1N1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTE1M2Q3Zjc5MTUxNTk0YjgyMzg3YTc2Y2M4ODJiYzg0NmJjYzQ4MzEyNDRjODA0Njk3MWEyMTYwNzc3MWI1ZjgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.7uJ9WWAfUQx8Rdt_Wl2nwrRoZmWnxbudKsIel4rejLQ)](https://private-user-images.githubusercontent.com/13099434/289311768-0587b922-f8cc-4afa-89ef-abb4abdd6742.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTAyNDQ3OTcsIm5iZiI6MTcxMDI0NDQ5NywicGF0aCI6Ii8xMzA5OTQzNC8yODkzMTE3NjgtMDU4N2I5MjItZjhjYy00YWZhLTg5ZWYtYWJiNGFiZGQ2NzQyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMTIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzEyVDExNTQ1N1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTE1M2Q3Zjc5MTUxNTk0YjgyMzg3YTc2Y2M4ODJiYzg0NmJjYzQ4MzEyNDRjODA0Njk3MWEyMTYwNzc3MWI1ZjgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.7uJ9WWAfUQx8Rdt_Wl2nwrRoZmWnxbudKsIel4rejLQ)

 

Can you make a tutorial about how to install it on an emulator, like Bluestacks or Nox?