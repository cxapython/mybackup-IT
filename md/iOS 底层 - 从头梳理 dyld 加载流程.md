> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844904040149729294)

前言
--

了解 `dyld` 的加载流程可以帮我们更系统的了解 `iOS` 应用的本质 . 无论是在逆向方向或者在底层研究方面 , `dyld` 都是必不可少的领域 . 对流程梳理清楚可以帮助我们更好地了解一些基础原理 . 例如我们之前讲 [分类底层原理详细研究流程](https://juejin.cn/post/6844903935002558472 "https://juejin.cn/post/6844903935002558472") , [load 方法调用机制解析](https://juejin.cn/post/6844903936088866823 "https://juejin.cn/post/6844903936088866823") , 都不可避免的提到 `dyld` .

本篇文章就整个加载流程进行梳理分析 , 并不会特别细 , 毕竟整个流程太多 , 需要提点的都会有所介绍 .

提示 : 了解本文前先请对 [Mach-O](https://juejin.cn/post/6844903983841214472 "https://juejin.cn/post/6844903983841214472") 文件有所了解 .

1、dyld
------

### 1.1 简介

`dyld` 全名 **The dynamic link editor** . 它是苹果的动态链接器，是苹果操作系统一个重要组成部分 ，在应用被编译打包成可执行文件格式的 `Mach-O` 文件之后 ，交由 `dyld` 负责链接 , 加载程序 。

`dyld` 是开源的，我们可以通过 [官网](https://link.juejin.cn?target=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fdyld%2F "https://opensource.apple.com/tarballs/dyld/") 下载它的源码来阅读理解它的运作方式，了解系统加载动态库的细节 。

我这里下载的是 `dyld-635.2` .

### 1.2 共享缓存

解读 `dyld` 有一个必不可少的东西 - 共享缓存 .

由于 `iOS` 系统中 `UIKit` / `Foundation` 等库每个应用都会通过 `dyld` 加载到内存中 , 因此 , 为了节约空间 , 苹果将这些系统库放在了一个地方 : **动态库共享缓存区 (dyld shared cache)** . (Mac OS 一样有) .

因此 , 类似 `NSLog` 的函数实现地址 , 并不会也不可能会在我们自己的工程的 `Mach-O` 中 , 那么我们的工程想要调用 `NSLog` 方法 , 如何能找到其真实的实现地址呢 ?

其流程如下 :

> *   **在工程编译时 , 所产生的 `Mach-O` 可执行文件中会预留出一段空间 , 这个空间其实就是符号表 , 存放在 `_DATA` 数据段中 ( 因为 `_DATA` 段在运行时是可读可写的 )**
>     
> *   编译时 : **工程中所有引用了共享缓存区中的系统库方法 , 其指向的地址设置成符号地址 , ( 例如工程中有一个 `NSLog` , 那么编译时就会在 `Mach-O` 中创建一个 `NSLog` 的符号 , 工程中的 `NSLog` 就指向这个符号 )**
>     
> *   运行时 : **当 `dyld`将应用进程加载到内存中时 , 根据 `load commands` 中列出的需要加载哪些库文件 , 去做绑定的操作 ( 以 `NSLog` 为例 , `dyld` 就会去找到 `Foundation` 中 `NSLog` 的真实地址写到 `_DATA` 段的符号表中 `NSLog` 的符号上面 )**
>     

这个过程被称为 `PIC` 技术 . (Position Independent Code : 位置代码独立)

了解了系统函数的整个加载过程 , 我们来看 `fishhook` 的函数名称 :

`rebind_symbols :: 重绑定符号` 也就简单明了了.

`fishhook` 原理就是 :

> **将编译后系统库函数所指向的符号 , 在运行时重绑定到用户指定的函数地址 , 然后将原系统函数的真实地址赋值到用户指定的指针上.**

2、dyld 加载流程
-----------

新建一个空 `app` 工程 , 在 `ViewController` 中添加 `load` 方法 .

```
+ (void)load{
    NSLog(@"load 来了");
}


```

`load` 方法添加断点 . 运行程序 . 查看函数调用栈 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/6/16f7a11b3f395b1b~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

通过 `lldb` : `bt` + `up` / `down` 指令来到入口 `_dyld_start` 处 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/6/16f7a150052181e3~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

### 2.1 _dyld_start

上图第 `11` 行 : `call` 就是调用函数的指令 , (同 bl) . 这个函数也就是我们 `app` 开始的地方 .

当我们点开一个应用 , 系统内核会开启一个进程 , 然后由 `dyld` 开始加载这个可执行文件 .

#### 2.1.1 dyldbootstrap :: start

`dyldbootstrap::start` 就是指 `dyldbootstrap` 这个命名空间作用域里的 `start` 函数 .

来到源码中 , 搜索 `dyldbootstrap` , 然后找到 `start` 函数 .

`cmd + shift + j` 可以定位文件位置

```
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
				intptr_t slide, const struct macho_header* dyldsMachHeader,
				uintptr_t* startGlue)
{
    slide = slideOfMainExecutable(dyldsMachHeader);
    bool shouldRebase = slide != 0;
#if __has_feature(ptrauth_calls)
    shouldRebase = true;
#endif
    if ( shouldRebase ) {
        rebaseDyld(dyldsMachHeader, slide);
    }

	mach_init();
	const char** envp = &argv[argc+1];
	const char** apple = envp;
	while(*apple != NULL) { ++apple; }
	++apple;

	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	runDyldInitializers(dyldsMachHeader, slide, argc, argv, envp, apple);
#endif
	uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
	return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}


```

这个函数首先有两个参数我们要说明一下 :

> *   1️⃣、`const struct macho_header* appsMachHeader` , 这个参数就是 `Mach-O` 的 `header` . 关于这个 `header` , [Mach-O 文件](https://juejin.cn/post/6844903983841214472 "https://juejin.cn/post/6844903983841214472") 这篇文章中 `Mach-O 文件结构` 里有详细描述 .
>     
> *   2️⃣、`intptr_t slide` , 这个其实就是 [ALSR](https://link.juejin.cn?target=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E5%259C%25B0%25E5%259D%2580%25E7%25A9%25BA%25E9%2597%25B4%25E9%2585%258D%25E7%25BD%25AE%25E9%259A%258F%25E6%259C%25BA%25E5%258A%25A0%25E8%25BD%25BD%2F22785938%3Ffr%3Daladdin%26fromtitle%3Daslr%26fromid%3D5779647 "https://baike.baidu.com/item/%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4%E9%85%8D%E7%BD%AE%E9%9A%8F%E6%9C%BA%E5%8A%A0%E8%BD%BD/22785938?fr=aladdin&fromtitle=aslr&fromid=5779647") , 说白了就是通过一个随机值 (也就是我们这里的 slide) 来实现地址空间配置随机加载 .
>     
>     *   当某个特定进程，在存储器中所能够使用与控制的地址空间在运行时随机进行分配 , 可以使某些攻击者无法事先获知地址 ，令攻击者难以通过固定地址获取函数或者内存值进行攻击 .
>         
>     *   `Mac OS X Lion10.7` 开始所有的应用程序均提供了 `ASLR` 支持 .
>         
> *   3️⃣、 物理地址 = `ALSR` + 虚拟地址 (偏移) .
>     

那么接下来 , 这个函数到底做了什么呢 ?

流程如下 :

*   首先 , 根据计算出来的 `ASLR` 的 `slide` 来重定向 `macho` .
    
*   初始化 , 允许 `dyld` 使用 `mach` 消息传递 .
    
*   栈溢出保护 .
    
*   初始化完成后调用 `dyld` 的 `main` 函数 ,`dyld::_main` .
    

#### 2.1.2 dyld::_main

直接点击跳转到 `dyld` - `main` 函数中 . 该函数是加载 `app` 的主要函数.

```
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
    // *函数太长 , 这里就不贴了.*/
}


```

这个函数主要流程如下 :

##### 2.1.2.1 准备工作

*   1️⃣ : 配置相关环境变量 .
*   2️⃣ : 设置上下文信息 `setContext` .
*   3️⃣ : 检测进程是否受限 , 在上下文中做出对应处理 `configureProcessRestrictions` , 检测环境变量 `checkEnvironmentVariables`.
    
    > *   熟悉越狱插件的同学应该都很清楚 , 某些环境变量会直接影响该库是否会被加载 , 有些防护操作就是基于这个原理来做的 . (后续更新越狱篇章攻防会详细讲述和演示)
    
*   4️⃣ : 根据环境变量配置打印信息 , `DYLD_PRINT_OPTS` 与 `DYLD_PRINT_ENV` , 大家可以在如下图中配置玩一玩 . ![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/6/16f7a559a2c21344~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)
*   5️⃣ : 获取程序架构 `getHostInfo` .

##### 2.1.2.2 加载共享缓存库

该流程主要步骤如下 :

*   1️⃣ : 检测共享缓存禁用状态 `checkSharedRegionDisable` . (iOS 下不会被禁用) .
    
*   2️⃣ : 加载共享缓存库 , `mapSharedCache` -> `loadDyldCache` . 这里加载共享缓存有几种情况 :
    
    *   1、仅加载到当前进程 `mapCachePrivate` , (模拟器仅支持加载到当前进程) .
    *   2、共享缓存是第一次被加载 , 就去做加载操作 `mapCacheSystemWide` .
    *   3、共享缓存不是第一次被加载 , 那么就不做任何处理 . ![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7e8623bcaa509~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

##### 2.1.2.3 reloadAllImages

```
sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);


```

实例化主程序 , 检测可执行程序格式 .

```
static ImageLoaderMachO* instantiateFromLoadedImage(const macho_header* mh, uintptr_t slide, const char* path)
{
	// try mach-o loader
	if ( isCompatibleMachO((const uint8_t*)mh, path) ) {
		ImageLoader* image = ImageLoaderMachO::instantiateMainExecutable(mh, slide, path, gLinkContext);
		addImage(image);
		return (ImageLoaderMachO*)image;
	}
	
	throw "main executable not a known format";
}


```

`isCompatibleMachO` 里就会通过 `header` 里的 `magic` , `cputype` , `cpusubtype` 去检测是否兼容 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7e98897cda0c2~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

检测通过 , 就会通过

`instantiateMainExecutable`

实例化这个 image , 并添加到

`static std::vector<ImageLoader*> sAllImages;`

这个全局的镜像列表中去 , 设置好上下文 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7ed39868ea8f1~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

`instantiateMainExecutable` 里 , 真正实例化主程序是用 `sniffLoadCommands` 这个函数去做的 . 有的同学可能对这个函数比较熟悉了 . 我们来稍微看一下 .

还是 `ImageLoaderMachO` 这个作用域里的 `sniffLoadCommands` 函数 .

```
void ImageLoaderMachO::sniffLoadCommands(const macho_header* mh, const char* path, bool inCache, bool* compressed,
											unsigned int* segCount, unsigned int* libCount, const LinkContext& context,
											const linkedit_data_command** codeSigCmd,
											const encryption_info_command** encryptCmd)
{
    *compressed = false;
	*segCount = 0;
	*libCount = 0;
	*codeSigCmd = NULL;
	*encryptCmd = NULL;
	/*
	...省略掉.
	*/
	// fSegmentsArrayCount is only 8-bits
	if ( *segCount > 255 )
		dyld::throwf("malformed mach-o image: more than 255 segments in %s", path);

	// fSegmentsArrayCount is only 8-bits
	if ( *libCount > 4095 )
		dyld::throwf("malformed mach-o image: more than 4095 dependent libraries in %s", path);

	if ( needsAddedLibSystemDepency(*libCount, mh) )
		*libCount = 1;
}


```

这个函数就是根据 `Load Commands` 来加载主程序 .

这里几个参数我们稍微说明下 :

*   `compressed` -> 根据 `LC_DYLD_INFO_ONYL` 来决定 .
*   `segCount` 段命令数量 , 最大不能超过 `255` 个.
*   `libCount` 依赖库数量 , `LC_LOAD_DYLIB (Foundation / UIKit ..)` , 最大不能超过 `4095` 个.
*   `codeSigCmd` , 应用签名 , 在 [应用签名原理及重签名 (重签微信应用实战)](https://juejin.cn/post/6844903969811070990 "https://juejin.cn/post/6844903969811070990") 这篇文章中有非常详细的讲述 , 建议读一读 .
*   `encryptCmd` , 应用加密信息 , (我们俗称的应用加壳 , 我们非越狱环境重签名都是需要砸过壳的应用才能调试 , 关于应用的砸壳 , 后续逆向文章越狱篇里会实际操作演练) .

经过以上步骤 , 主程序的实例化就已经完成了 .

##### 2.1.2.4 加载插入动态库

```
if ( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
	for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
		loadInsertedDylib(*lib);
}


```

熟悉越狱插件的同学应该很清楚这个机制了 . 根据 `DYLD_INSERT_LIBRARIES` 环境变量来决定是否需要加载插入的动态库 .

越狱的插件就是基于这个原理来实现只需要下载插件 , 就可以影响到应用 . 有部分防护手段就用到了这个环境变量 (后续逆向文章会带着大家自己写一个越狱插件 , 这个很简单 , 然后会讲一讲越狱环境插件如何防护 .) .

`sInsertedDylibCount = sAllImages.size()-1;`

记录插入动态库的数量 .

##### 2.1.2.5 链接主程序

```
// link main executable
gLinkContext.linkingMainExecutable = true;

link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL), -1);
sMainExecutable->setNeverUnloadRecursive();
if ( sMainExecutable->forceFlat() ) {
	gLinkContext.bindFlat = true;
	gLinkContext.prebindUsage = ImageLoader::kUseNoPrebinding;
}
if ( sInsertedDylibCount > 0 ) {
	for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
		ImageLoader* image = sAllImages[i+1];
		link(image, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL), -1);
		image->setNeverUnloadRecursive();
	}
	
	for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
		ImageLoader* image = sAllImages[i+1];
		image->registerInterposing(gLinkContext);
	}
}


```

点击进入 `link` 函数 , `link` 函数中有一系列 `recursiveLoadLibraries` , `recursiveBindWithAccounting -> recursiveBind` , 也就是递归进行符号绑定的过程 .

`link` 函数执行完毕之后 , `dyld :: main` 会调用 `sMainExecutable->weakBind(gLinkContext);` 进行弱绑定 , 懒加载绑定 , 也就是说弱绑定一定发生在 其他库链接绑定完成之后 .

绑定的过程就是我们上述 `1.2` 章节中所讲的共享缓存绑定的过程 .

> 走到了这里 , 主程序已经实例化完毕 , 但还没有加载 , `framework` 已经加载完毕了 , 那讲到这插一句题外话 , 不同 `framework` , 谁先会被加载 ? 其实根据二进制顺序有关 , `Xcode` 中可以自由调整 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7f107cd58197d~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

拖动就可以自己调整顺序了 , 编译顺序就会根据这个顺序来 , 同样你可以使用 `MachOView` 来查看二进制顺序 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7f1331e00bea7~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

至此 , 配置环境变量 -> 加载共享缓存 -> 实例化主程序 -> 加载动态库 -> 链接动态库 就已经完成了 .

继续往 `dyld :: main` 下面找 , 我们会看到

```
initializeMainExecutable();


```

那么我们回到函数调用栈看下 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7f1dddb364f02~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

#### 2.1.3 运行主程序

通过查看源码查看 , 结合函数调用栈 , 我们跟进去调用流程 . `initializeMainExecutable` -> `runInitializers` -> `processInitializers` -> **递归**调用 `recursiveInitialization` .

到了这里 , 直接点击 进不去了 , 同理 , `cmd` + `shift` + `o`, 搜索 `recursiveInitialization` . 来到函数实现 , 找到如下代码 :

```
// let objc know we are about to initialize this image
uint64_t t1 = mach_absolute_time();
fState = dyld_image_state_dependents_initialized;
oldState = fState;
context.notifySingle(dyld_image_state_dependents_initialized, this, &timingInfo);

// initialize this image
bool hasInitializers = this->doInitialization(context);

// let anyone know we finished initializing this image
fState = dyld_image_state_initialized;
oldState = fState;
context.notifySingle(dyld_image_state_initialized, this, NULL);


```

调用 `notifySingle` 函数 .

⚠️ : 重头戏来了 . 根据函数调用栈我们发现 , 下一步是调用 `load_images` , 可是这个 `notifySingle` 里并没有找到 `load_images` 的影子 . 但是我们看到了这么个东西 :

```
(*sNotifyObjCInit)(image->getRealPath(), image->machHeader());


```

> 这是个回调函数的调用 , `sNotifyObjCInit` 上面判断了并不会为空 , 那就代表一定是有值的 . 那我们搜索一下 `sNotifyObjCInit` , 看看什么时候被赋的值 .

直接本文件搜索 , 看到如下 :

```
void registerObjCNotifiers(_dyld_objc_notify_mapped mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped unmapped)
{
	// record functions to call
	sNotifyObjCMapped	= mapped;
	sNotifyObjCInit		= init;
	sNotifyObjCUnmapped = unmapped;

	// call 'mapped' function with all images mapped so far
	try {
		notifyBatchPartial(dyld_image_state_bound, true, NULL, false, true);
	}
	catch (const char* msg) {
		// ignore request to abort during registration
	}

	// <rdar://problem/32209809> call 'init' function on all images already init'ed (below libSystem)
	for (std::vector<ImageLoader*>::iterator it=sAllImages.begin(); it != sAllImages.end(); it++) {
		ImageLoader* image = *it;
		if ( (image->getState() == dyld_image_state_initialized) && image->notifyObjC() ) {
			dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);
			(*sNotifyObjCInit)(image->getRealPath(), image->machHeader());
		}
	}
}



```

也就是说 , 这个函数调用 , 其第二个参数赋值给了 `sNotifyObjCInit` , 然后在 `notifySingle` 里被执行 .

那么我们搜索一下 `registerObjCNotifiers` , 看看其在什么时候被调用的 , 搜索发现 :

```
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped)
{
	dyld::registerObjCNotifiers(mapped, init, unmapped);
}


```

再继续搜索 , 没啥结果了 . 那么怎么办 , 不着急 , 我们来到测试工程里下一个符号断点 `_dyld_objc_notify_register` , 运行来到断点 , 看函数调用栈 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7f32d53238ea3~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

. 至此 , 我们看到的就是

`runtime`

被加载的整个流程 , 来到

`objc 750`

的代码中直接搜索

`_objc_init`

.

#### 2.1.4 _objc_init

```
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}


```

来到这里 , 我们就看到了 `_dyld_objc_notify_register` 被调用 , 传递了三个参数 , 这三个分别代表 在 [分类底层原理详细研究](https://juejin.cn/post/6844903935002558472#heading-7 "https://juejin.cn/post/6844903935002558472#heading-7") 中我们也有详细讲述过 .

> *   `map_images` : `dyld` 将 `image` 加载进内存时 , 会触发该函数.
> *   `load_images` : `dyld` 初始化 `image` 会触发该方法. ( 我们所熟知的 `load` 方法也是在此处调用 ) .
> *   `unmap_image` : `dyld` 将 `image` 移除时 , 会触发该函数 .

当然 , 你可以通过 `lldb` 验证一下 .

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7f47bb972a40d~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

那么这个 `load_images` , 就调用了各个类的 `load` 方法 ( `call_load_methods` ) . 关于这个请看 [分类底层原理详细研究](https://juejin.cn/post/6844903935002558472#heading-7 "https://juejin.cn/post/6844903935002558472#heading-7") 与 [load 方法调用机制解析](https://juejin.cn/post/6844903936088866823 "https://juejin.cn/post/6844903936088866823") 这两篇文章 .

要声明一下的是 :

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7fbb7b92b7a91~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

那么也就是说 :

> *   1️⃣、 当 `dyld` 加载到开始链接主程序的时候 , 递归调用 `recursiveInitialization` 函数 .
> *   2️⃣、 这个函数第一次执行 , 进行 `libsystem` 的初始化 . 会走到 `doInitialization` -> `doModInitFunctions` -> `libSystemInitialized` .
> *   3️⃣、 `Libsystem` 的初始化 , 它会调用起 `libdispatch_init` , `libdispatch` 的 `init` 会调用 `_os_object_init` , 这个函数里面调用了 `_objc_init` .
> *   4️⃣、`_objc_init` 中注册并保存了 `map_images` , `load_images` , `unmap_image` 函数地址.
> *   5️⃣ : 注册完毕继续回到 `recursiveInitialization` 递归下一次调用 , 例如 `libobjc` , 当 `libobjc` 来到 `recursiveInitialization` 调用时 , 会触发 `libsystem` 调用到 `_objc_init` 里注册好的回调函数进行调用 . 就来到了 `libobjc` , 调用 `load_images`.

跟我们上面截图的函数调用栈一模一样 .

#### 2.1.5 doInitialization

`dyld` 来到 `doInitialization` 时 ,

```
bool ImageLoaderMachO::doInitialization(const LinkContext& context)
{
	CRSetCrashLogMessage2(this->getPath());

	// mach-o has -init and static initializers
	doImageInit(context);
	doModInitFunctions(context);
	
	CRSetCrashLogMessage2(NULL);
	
	return (fHasDashInit || fHasInitializers);
}


```

在 `doModInitFunctions` 中 , 值得一提的是会调用 `c++` 的构造方法 .

演示如下 :

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7fc4146b88c4b~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

打印结果 :

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7fc4809c3499b~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

这种 `c++` 构造方法存储在 `__DATA` 段 , `__mod_init_func` 节中.

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7fc7c130206c6~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

#### 2.1.6 找到主程序的入口

```
// find entry point for main executable
result = (uintptr_t)sMainExecutable->getEntryFromLC_MAIN();


```

找到真正 `main` 函数入口 并返回.

总结 :
----

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/1/7/16f7fceeeacaa3da~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

以上便是 `dyld` 加载应用的完整流程 . 建议大家仔细探索 .