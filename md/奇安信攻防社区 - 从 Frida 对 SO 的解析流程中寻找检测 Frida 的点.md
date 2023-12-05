> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [forum.butian.net](https://forum.butian.net/share/1389)

> 奇安信攻防社区 - 从 Frida 对 SO 的解析流程中寻找检测 Frida 的点

# 0x01 frida 寻找符号地址 frida hook Native 的基础就是如何寻找到符号在内存中的地址，在 frida 中就仅仅是一句话 findExportByName 或者通过枚举符号遍历，今天我们就一起看一看他是如何实现...

frida hook Native 的基础就是如何寻找到符号在内存中的地址，在 frida 中就仅仅是一句话 findExportByName 或者通过枚举符号遍历，今天我们就一起看一看他是如何实现的。

寻找模块基址
------

首先就要找到他是哪里实现的，frida 源码这么大而我们之前有没有阅读 ts 的经验，所以选择了直接 grep 寻找函数名称的方法，在 frida 目录输入下面命令

```
grep -rl findExportByName


```

发现了如下文件中有我们要的函数名称

```
frida-core/tests/test-host-session.vala
frida-core/src/darwin/agent/xpcproxy.js
frida-core/src/darwin/agent/launchd.js
frida-gum/tests/gumjs/script.c
frida-gum/bindings/gumjs/runtime/core.js
frida-gum/bindings/gumjs/gumv8module.cpp
frida-gum/bindings/gumjs/gumquickmodule.c


```

在这之中我们发现了 2 个文件中都有 findExportByName 的实现，那么如何分辨呢，我们选择了加一条日志重新编译来看一看到底是哪一个

```
GUMJS_DEFINE_FUNCTION (gumjs_module_find_export_by_name)
{
    __android_log_print(6,"r0ysue","i am from v8")
......

}

GUMJS_DEFINE_FUNCTION (gumjs_module_find_export_by_name)
{

  __android_log_print(6,"r0ysue","i am from qucik");
 .......
}


```

![](https://shs3.b.qianxin.com/attack_forum/2022/03/attach-4004f0036812eddb127276ddaafeb4d3e42afab4.png)

可以看到是 quick 中的代码，那么我们只要阅读这当中的代码即可，首先进入了 gum_module_find_export_by_name 函数中，地址就是它的返回值，

```
GUMJS_DEFINE_FUNCTION (gumjs_module_find_export_by_name)
{
...
  address = gum_module_find_export_by_name (module_name, symbol_name);
...
}

GumAddress
gum_module_find_export_by_name (const gchar * module_name,
                                const gchar * symbol_name)
{
  GumAddress result;
  void * module;
#ifdef HAVE_ANDROID
  if (gum_android_get_linker_flavor () == GUM_ANDROID_LINKER_NATIVE &&
      gum_android_try_resolve_magic_export (module_name, symbol_name, &result))
    return result;
#endif
  if (module_name != NULL)
  {
    module = gum_module_get_handle (module_name);/获得模块基址
    if (module == NULL)
      return 0;
  }
  else
  {
    module = RTLD_DEFAULT;
  }
  result = GUM_ADDRESS (gum_module_get_symbol (module, symbol_name));
  if (module != RTLD_DEFAULT)
    dlclose (module);
  return result;
}



```

接着跟进 gum_module_get_handle 继续分析如何得到的模块地址，这里它分了 2 种形式，正如我们之前写的文章，高版本的 android 不能打开系统白名单之外的 so 所以上面是修改的 dlopen，下面是 linux 的 dlopen

```
gum_module_get_handle (const gchar * module_name)
{
#ifdef HAVE_ANDROID
  if (gum_android_get_linker_flavor () == GUM_ANDROID_LINKER_NATIVE)
    return gum_android_get_module_handle (module_name);
#endif
  return dlopen (module_name, RTLD_LAZY | RTLD_NOLOAD);
}


```

寻找 linker 相关的信息
---------------

普通的 dlopen 之前的文章领着大家看过这里就只分析无限制的 dlopen 是如何实现的了，跟进 gum_android_get_module_handle，这里有 2 个函数一个是 gum_enumerate_soinfo，一个是 gum_store_module_handle_if_name_matches，我们分开看，先看一个他是如何枚举 soinfo 的，跟进去发现 gum_linker_api_get 函数来获得 linker 中的 api，跟进去看看如何实现的，这里首先获得了 linker 的首地址如下，最终到了 gum_try_parse_linker_proc_maps_line 函数中，这个函数就是遍历了 maps 来获得 linker（目前好像只有这一种获得 linker 首地址的方法），虽然这种方法很准确但是 frida 太谨慎了，又校验了魔术字段，又校验了只读权限，所以这是一个 antifrida 的关键点

```
void *
gum_android_get_module_handle (const gchar * name)
{
  GumGetModuleHandleContext ctx;
  ctx.name = name;
  ctx.module = NULL;
  gum_enumerate_soinfo (
      (GumFoundSoinfoFunc)  gum_store_module_handle_if_name_matches, &ctx);
  return ctx.module;
}

static void
gum_enumerate_soinfo (GumFoundSoinfoFunc func,
                      gpointer user_data)
{

  api = gum_linker_api_get ();
  .......
}
static GumLinkerApi *
gum_linker_api_get (void)
{
 .....

  g_once (&once, (GThreadFunc) gum_linker_api_try_init, NULL);
.......
}
static GumLinkerApi *
gum_linker_api_try_init (void)
{
....
  linker = gum_android_open_linker_module ();
  ....
}
GumElfModule *
gum_android_open_linker_module (void)
{
  const GumModuleDetails * linker;
  linker = gum_android_get_linker_module_details ();
  return gum_elf_module_new_from_memory (linker->path,
      linker->range->base_address);
}
const GumModuleDetails *
gum_android_get_linker_module_details (void)
{
  static GOnce once = G_ONCE_INIT;

  g_once (&once, (GThreadFunc) gum_try_init_linker_details, NULL);

  if (once.retval == NULL)
  {
    g_critical ("Unable to locate the Android linker; please file a bug");
    g_abort ();
  }

  return once.retval;
}

static const GumModuleDetails *
gum_try_init_linker_details (void)
{
  const GumModuleDetails * result = NULL;
  gchar * linker_path;
  GRegex * linker_path_pattern;
  gchar * maps, ** lines;
  gint num_lines, vdso_index, i;
  linker_path = gum_find_linker_path ();
  linker_path_pattern = gum_find_linker_path_pattern ();
  g_file_get_contents ("/proc/self/maps", &maps, NULL, NULL);
  lines = g_strsplit (maps, "\n", 0);
  num_lines = g_strv_length (lines);

  vdso_index = -1;
  for (i = 0; i != num_lines; i++)
  {
    const gchar * line = lines[i];

    if (g_str_has_suffix (line, " [vdso]"))
    
    {
      vdso_index = i;
      break;
    }
  }
  if (vdso_index == -1)
    goto no_vdso;

  for (i = vdso_index + 1; i != num_lines; i++)
  {
    if (gum_try_parse_linker_proc_maps_line (lines[i], linker_path,
        linker_path_pattern, &gum_dl_module, &gum_dl_range))
        
    {
      result = &gum_dl_module;
      goto beach;
    }
  }

  for (i = vdso_index - 1; i >= 0; i--)
  {
    if (gum_try_parse_linker_proc_maps_line (lines[i], linker_path,
        linker_path_pattern, &gum_dl_module, &gum_dl_range))
    {
      result = &gum_dl_module;
      goto beach;
    }
  }

  goto beach;

no_vdso:
  for (i = num_lines - 1; i >= 0; i--)
  {
    if (gum_try_parse_linker_proc_maps_line (lines[i], linker_path,
        linker_path_pattern, &gum_dl_module, &gum_dl_range))
    {
      result = &gum_dl_module;
      goto beach;
    }
  }

.......
}

static gboolean
gum_try_parse_linker_proc_maps_line (const gchar * line,
                                     const gchar * linker_path,
                                     const GRegex * linker_path_pattern,
                                     GumModuleDetails * module,
                                     GumMemoryRange * range)
{
  GumAddress start, end;
  gchar perms[5] = { 0, };
  gchar path[PATH_MAX];
  gint n;
  const guint8 elf_magic[] = { 0x7f, 'E', 'L', 'F' };

  n = sscanf (line,
      "%" G_GINT64_MODIFIER "x-%" G_GINT64_MODIFIER "x "
      "%4c "
      "%*x %*s %*d "
      "%s",
      &start, &end,
      perms,
      path);
  if (n != 4)
    return FALSE;

  if (!g_regex_match (linker_path_pattern, path, 0, NULL))
    return FALSE;

  if (perms[0] != 'r')
    return FALSE;

  if (memcmp (GSIZE_TO_POINTER (start), elf_magic, sizeof (elf_magic)) != 0)
    return FALSE;

  module->name = strrchr (linker_path, '/') + 1;
  module->range = range;
  module->path = linker_path;

  range->base_address = start;
  range->size = end - start;

  return TRUE;
}



```

之后我们一起看一看，他是如何寻找 linker 中函数的地址，也就是如何初始化的 api，可以看到和我们之前的方法差不多都是从节头表索引，也只有遍历节头表这一种方式能够得到 linker 中的 dlopen 这种符号了因为 linker 没有导出符号，最终到了 gum_store_linker_symbol_if_needed 函数中，保存需要的符号类似 do_dlopen 等, 经此之后我们就有了直接从 maps 中得到的 linker 中的 do_dlopen 和 do_dlsym 等, 保存到了 api 中

```
static GumLinkerApi *
gum_linker_api_try_init (void)
{
.....
  api_level = gum_android_get_api_level ();
gum_elf_module_enumerate_symbols (linker,
      (GumElfFoundSymbolFunc) gum_store_linker_symbol_if_needed, &pending);
.....
}
void
gum_elf_module_enumerate_symbols (GumElfModule * self,
                                  GumElfFoundSymbolFunc func,
                                  gpointer user_data)
{
  gum_elf_module_enumerate_symbols_in_section (self, SHT_SYMTAB, func,
      user_data);
}
static void
gum_elf_module_enumerate_symbols_in_section (GumElfModule * self,
                                             GumElfSectionHeaderType section,
                                             GumElfFoundSymbolFunc func,
                                             gpointer user_data)
{
......
  if (!gum_elf_module_find_section_header_by_type (self, section, &scn, &shdr))
.......

  for (symbol_index = 0;
      symbol_index != symbol_count && carry_on;
      symbol_index++)
  {
    ......

    carry_on = func (&details, user_data);
  }
}

static gboolean
gum_store_linker_symbol_if_needed (const GumElfSymbolDetails * details,
                                   guint * pending)
{
  
  GUM_TRY_ASSIGN (dlopen, "__dl___loader_dlopen");       
  GUM_TRY_ASSIGN (dlsym, "__dl___loader_dlvsym");        
  GUM_TRY_ASSIGN (dlopen, "__dl__Z8__dlopenPKciPKv");    
  GUM_TRY_ASSIGN (dlsym, "__dl__Z8__dlvsymPvPKcS1_PKv"); 
  
  GUM_TRY_ASSIGN_OPTIONAL (do_dlopen,
      "__dl__Z9do_dlopenPKciPK17android_dlextinfoPv");
  GUM_TRY_ASSIGN_OPTIONAL (do_dlsym, "__dl__Z8do_dlsymPvPKcS1_S_PS_");

  GUM_TRY_ASSIGN (dl_mutex, "__dl__ZL10g_dl_mutex"); 
  GUM_TRY_ASSIGN (dl_mutex, "__dl__ZL8gDlMutex");    
  GUM_TRY_ASSIGN (solist_get_head, "__dl__Z15solist_get_headv"); 
  GUM_TRY_ASSIGN_OPTIONAL (solist, "__dl__ZL6solist");           
  GUM_TRY_ASSIGN_OPTIONAL (libdl_info, "__dl_libdl_info");       
  GUM_TRY_ASSIGN (solist_get_somain, "__dl__Z17solist_get_somainv"); 
  GUM_TRY_ASSIGN_OPTIONAL (somain, "__dl__ZL6somain");               

  GUM_TRY_ASSIGN (soinfo_get_path, "__dl__ZNK6soinfo12get_realpathEv");

beach:
  return *pending != 0;
}



```

处理我们要寻找的符号
----------

那么这里我们就可以通过 linker 中的 soinfo 链表来遍历所有的 soinfo，然后再通过 **dl**Z17solist_get_somainv 函数来匹配 so 的名字，最后通过我们之前得到的 dlopen 函数调用来，得到该 so 的 handle

```
static void
gum_enumerate_soinfo (GumFoundSoinfoFunc func,
                      gpointer user_data)
{

    ......
 somain = api->solist_get_somain ();
  gum_init_soinfo_details (&details, somain, api, &ranges);
  carry_on = func (&details, user_data);
  for (si = api->solist_get_head (); carry_on && si != NULL; si = next)
  {
     carry_on = func (&details, user_data);
     ....

}
.....
}

static gboolean
gum_store_module_handle_if_name_matches (const GumSoinfoDetails * details,
                                         GumGetModuleHandleContext * ctx)
{
  GumLinkerApi * api = details->api;

  if (gum_linux_module_path_matches (details->path, ctx->name))
  {
    GumSoinfoBody * sb = details->body;
    int flags = RTLD_LAZY;
    void * caller_addr = GSIZE_TO_POINTER (sb->base);

    if (gum_android_is_vdso_module_name (details->path))
      return FALSE;

    if ((sb->flags & GUM_SOINFO_NEW_FORMAT) != 0)
    {
      GumSoinfo * parent;

      parent = (sb->parents.head != NULL)
          ? sb->parents.head->element
          : NULL;
      if (parent != NULL)
      {
        caller_addr = GSIZE_TO_POINTER (gum_soinfo_get_body (parent)->base);
      }

      if (sb->version >= 1)
      {
        flags = sb->rtld_flags;
      }
    }

    if (gum_android_get_api_level () >= 21)
    {
      flags |= RTLD_NOLOAD;
    }

    if (api->dlopen != NULL)
    {
      
      ctx->module = api->dlopen (details->path, flags, caller_addr);
    }
    else if (api->do_dlopen != NULL)
    {
      
      ctx->module = api->do_dlopen (details->path, flags, NULL, caller_addr);
    }
    else
    {
      ctx->module = dlopen (details->path, flags);
    }

    return FALSE;
  }

  return TRUE;
}



```

至此我们解析完了 frida 构造没有限制的 dlopen 的过程，那么接下来就看看他是如何找到 dlsym 的

```
GumAddress
gum_module_find_export_by_name (const gchar * module_name,
                                const gchar * symbol_name)
{

....
  result = GUM_ADDRESS (gum_module_get_symbol (module, symbol_name));
....
}
static void *
gum_module_get_symbol (void * module,
                       const gchar * symbol)
{
  GumGenericDlsymImpl dlsym_impl = dlsym;
#ifdef HAVE_ANDROID
  if (gum_android_get_linker_flavor () == GUM_ANDROID_LINKER_NATIVE)
    gum_android_find_unrestricted_dlsym (&dlsym_impl);
#endif
  return dlsym_impl (module, symbol);
}

gboolean
gum_android_find_unrestricted_dlsym (GumGenericDlsymImpl * generic_dlsym)
{
  if (!gum_android_find_unrestricted_linker_api (NULL))
    return FALSE;
  *generic_dlsym = gum_call_inner_dlsym;
  return TRUE;
}
static void *
gum_call_inner_dlsym (void * handle,
                      const char * symbol)
{
  return gum_dl_api.dlsym (handle, symbol, NULL, gum_dl_api.trusted_caller);
}



```

到此为止我们就分析完了，frida 是如何找到高版本的 dlopen 和 dlsym，其实就是从节头表找到几个函数的地址，期间还意外的发现了 anti frida 的方法（其实具体逻辑不在这里，因为即使 dlopen 为空也能正常找到符号地址，但是 attach 需要验头，下文再说）

frida 发展了这么久，发展出了 2 种 anti 方式，一种是以字符串为基准的旧检测方式，一种是以 frida 代码为基准的各种各样的崩溃基址，旧方案现在很多地方还在使用，但是旧方案的绕过方式就太多了，比如 hook fgets 函数，甚至出现了 hluda 这种傻瓜式的绕过方式，所以有必要开发新的 anti 方式，这就是这篇文章的主旨希望能找到一个新的，难以被发现的 antifrida 的方式

旧方案之`maps`检测法
-------------

在字符串的检测方案中，大部分用的都是这种，但是这种也很容易被感知，它的代码结构如下, 主要就是检测 maps 文件种是否有 frida-agent 字符串，当然这种取自 maps 的方式太容易被感知了，随便 hook 一下就知道我们遍历了 maps，所以有以下的改进版本，通过遍历链表的方式来获得 so 的名称，查看是否有 frida 字样的 so。

```
void anti3(){
while (1) {
    sleep(1);
    char line[1024];

    FILE *fp = fopen("/proc/self/maps", "r");
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, "frida-agent")) {
            __android_log_print(6, "r0ysue", "i find frida from anti3");
        }
    }
}
}


```

改进

```
 void fridafind(){
    char line[1024];
    int *start;
    int *end;
    int n=1;
    int m=1;
    int *start1;
    FILE *fp=fopen("/proc/self/maps","r");
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, "linker64") ) {
            __android_log_print(6,"r0ysue","%s", line);
            if(n==1){
                start = reinterpret_cast<int *>(strtoul(strtok(line, "-"), NULL, 16));
                end = reinterpret_cast<int *>(strtoul(strtok(NULL, " "), NULL, 16));

            }
            else{
                strtok(line, "-");
                end = reinterpret_cast<int *>(strtoul(strtok(NULL, " "), NULL, 16));
            }
            n++;

        }
        if (strstr(line, "libopenjdkjvm.so") ) {
            __android_log_print(6,"r0ysue","%s", line);
            if(m==1){
                start1 = reinterpret_cast<int *>(strtoul(strtok(line, "-"), NULL, 16));

            }
            m++;
        }

    }
int dlopenoff=findsym("/system/bin/linker64","__dl__Z8__dlopenPKciPKv");
    int headeroff=findsym("/system/bin/linker64","_dl__ZL6solist");
    long header= *(long *) ((char *) start + headeroff);
    for ( _QWORD *result = (_QWORD *)header; result; result = (_QWORD *)result[5] )
     if(strstr((const char*)*(_QWORD *)((__int64)result + 408),"frida"))
        __android_log_print(6,"r0ysue","%s",*(_QWORD *)((__int64)result + 408));

    }

}


```

这种还有一个版本，就是检查 / data/local/tmp 目录下面有没有 frida 依赖 so 所组成的文件夹，就是说 frida-server 在启动的时候会将依赖的 so 放在  
/data/local/tmp 这个文件夹下面，所以我们可以扫描有没有这个文件夹，类似与下面这样的代码，当然上面提到的两种方法都能被简单的绕过，比如典型了 hluda，就可以轻松的绕过，或者直接 hook strstr 函数也能发现校验的关键点，就不是很好，所以其实也可以改成逐比特用等号对比，当然也是很好绕过就对了

```
void anti4(){
    int a=   access("/data/local/tmp/re.frida.server",0);
    if(a ==0)
    __android_log_print(6,"r0ysue","i find frida from anti4");

}


```

当然这里还有一些原理性的检测方法，比如和 xposed 一样检测 ArtMethod 的 AccessFlags 值来判断一个确定为 Java 的函数是否变成了 Native 函数，这个和 java hook 的原理有关, 这个是 frida 绕不开的，就是想 hookjava 函数就一定要将 java 函数改成 native 函数，但是这种方式如果不 hook java 函数直接搞 Native 层就拉了，所以这种方案也不太行。

```
void anti7(__int64 a1){
    while (1) {
        sleep(1);

        __android_log_print(6,"r0ysue","i find frida %x", (~*(_DWORD *)(a1 + 4) & 0x80000) );

        if((~*(_DWORD *)(a1 + 4) & 0x80000) !=0)
            __android_log_print(6,"r0ysue","i find frida %x", (~*(_DWORD *)(a1 + 4) & 0x80000) );
    }

}


```

当然后来又有大佬搞出了一个方案, 见贴`https://bbs.pediy.com/thread-268586.htm`, 这种方式提供了一个新思路，就是从 frida 的变化入手，例如检测 frida 的 inline hook，这种方法就相当的好用了，因为 frida 作者也说了异常处理有一个 bug 必须要 hook PrettyMethod 函数，所以这种 script boy 就是无论如何都绕不开的

```
function fixupArtQuickDeliverExceptionBug (api) {                                                            
  const prettyMethod = api['art::ArtMethod::PrettyMethod'];
  if (prettyMethod === undefined) {
    return;
  }
  
  Interceptor.attach(prettyMethod.impl, artController.hooks.ArtMethod.prettyMethod);
  Interceptor.flush();
}


```

所以说这种的 anti 代码就如下, 这种就靠谱多了，但是有可能会误杀，现在很多 inline hook 都会采用 x16 跳转这种形式

```
void anti6(long * as){
        while (1) {
            pthread_mutex_lock(&mutex);


                if (*as == 0xd61f020058000050) {
                    __android_log_print(6, "r0ysue", "i find frida from anti6 ");
                }
sleep(2);
            pthread_mutex_unlock(&mutex);
            }
        }


```

所以说旧方式都是形式上的 anti frida，都是可见的，都是在 frida 对系统的更改，那么新方式就是从 frida 的 bug 出发，要寻找 frida 在做寻找符号过程中容易发出异常的点，来主动抛出这些异常。

新方案之 attach 流程中寻找 anti 点
------------------------

在上篇文章中我们发现了 frida 调用了一个函数`gum_android_open_linker_module`来获取 linker 的地址，但是有一个缺陷，导致我们可以根据这一点反制 frida，接下来我们就来一起看一下这个问题。

首先写一个简单的 demo，逻辑很简单就是在主函数里面加一个 anti7 函数，从 maps 里面遍历 linker64，然后把它开头的魔术字随便段改一个，比如我们这里就是将 0x7f 改成了 0，最后看一下结果

```
void anti7(){
    char line[1024];
    int *start;
    int *end;
    int n=1;
    FILE *fp=fopen("/proc/self/maps","r");
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, "linker64") ) {
            __android_log_print(6,"r0ysue","%s", line);
            if(n==1){
                start = reinterpret_cast<int *>(strtoul(strtok(line, "-"), NULL, 16));
                end = reinterpret_cast<int *>(strtoul(strtok(NULL, " "), NULL, 16));
            }
            else{
                strtok(line, "-");
                end = reinterpret_cast<int *>(strtoul(strtok(NULL, " "), NULL, 16));
            }
            n++;
        }
    }
long* sr= reinterpret_cast<long *>(start);
    mprotect(start,PAGE_SIZE,PROT_WRITE|PROT_READ|PROT_EXEC);
    *sr=*sr^0x7f;
    __android_log_print(6, "r0ysue", "i find frida %p",*sr);
    void* tt=dlopen("libc.so",RTLD_NOW);
    void* ts=dlsym(tt,"strstr");
    __android_log_print(6,"r0ysue","%p",ts);
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_roysue_anti_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject ) {
    std::string hello = "Hello from C++";
anti7();
    return env->NewStringUTF(hello.c_str());
}


```

![](https://shs3.b.qianxin.com/attack_forum/2022/03/attach-20cbf329c330bdd1f40ed653e2486b1ae139ed8b.png)

可以看到以 attach 的方式启动，可以成功的使我们的 frida 崩溃掉，我们来一起看一下他为什么会有这种结果。

可以直接从 frida-core/src/linux/frida-helper-backend-glue.c 目录下的_frida_linux_helper_backend_do_inject 函数入手（还是 c 代码比较易读其他语言不太易读），发现这里面调用了 frida_resolve_linker_address 函数来获得 dlclose 函数，我们跟进去看一下

```
void
_frida_linux_helper_backend_do_inject (FridaLinuxHelperBackend * self, guint pid, const gchar * path, const gchar * entrypoint, const gchar * data, const gchar * temp_path, guint id, GError ** error)
{
    .....
#elif defined (HAVE_ANDROID)
  params.dlopen_impl = frida_resolve_android_dlopen (pid);
  params.dlclose_impl = frida_resolve_linker_address (pid, dlclose);
  params.dlsym_impl = frida_resolve_linker_address (pid, dlsym);
....
}



```

有两个 frida_resolve_linker_address 函数我们只需要看这个 ANDROID 就好了，它之中调用了 gum_android_get_linker_module_details 函数来获得 linker 地址

```
#ifdef HAVE_ANDROID
static GumAddress
frida_resolve_linker_address (pid_t pid, gpointer func)
{
......
  else
    local_base = gum_android_get_linker_module_details ()->range->base_address;
.....

  return remote_address;
}


```

接着就跳到了 gum_android_get_linker_module_details 函数，又回到了上篇文章的那里

```
const GumModuleDetails *
gum_android_get_linker_module_details (void)
{
  static GOnce once = G_ONCE_INIT;
  g_once (&once, (GThreadFunc) gum_try_init_linker_details, NULL);
  if (once.retval == NULL)
  {
    g_critical ("Unable to locate the Android linker; please file a bug");
    g_abort ();
  }
  return once.retval;
}


```

调用了 gum_try_parse_linker_proc_maps_line，来寻找 maps 当中的 linker

```
static const GumModuleDetails *
gum_try_init_linker_details (void)
{
no_vdso:
  for (i = num_lines - 1; i >= 0; i--)
  {
    if (gum_try_parse_linker_proc_maps_line (lines[i], linker_path,
        linker_path_pattern, &gum_dl_module, &gum_dl_range))
    {
      result = &gum_dl_module;
      goto beach;
    }
  }

  return result;
}


```

最终到了我们的判断函数 gum_try_parse_linker_proc_maps_line，这里面最大的问题就是验证了 elf 头部信息这个根本不会被用到的东西，那么只要我们更改掉 maps 里面的 elf 头那么 frida 就找不到 linker 的地址了，那么 frida 就自然崩掉了，会抛出异常`Unable to locate the Android linker; please file a bug`

```
static gboolean
gum_try_parse_linker_proc_maps_line (const gchar * line,
                                     const gchar * linker_path,
                                     const GRegex * linker_path_pattern,
                                     GumModuleDetails * module,
                                     GumMemoryRange * range)
{
    .....
  const guint8 elf_magic[] = { 0x7f, 'E', 'L', 'F' };
  if (memcmp (GSIZE_TO_POINTER (start), elf_magic, sizeof (elf_magic)) != 0)
    return FALSE;
    ....
  return TRUE;
}


```

新方案之 findsymbol 流程中寻找 anti 点
----------------------------

承接上文的`findsymbol`, 这里其实存在一个巨大的`bug`，就是`somain`的获取他没有判断是否为空我们跟下去看一下，跟踪到最后发现它没有判断是否为空就直接取值了，就会造成地址不对这种情况，下面我们试一下。

```
static void
gum_enumerate_soinfo (GumFoundSoinfoFunc func,
                      gpointer user_data)
{

    ......
    
 somain = api->solist_get_somain ();
  gum_init_soinfo_details (&details, somain, api, &ranges);
.....
}

static void
gum_init_soinfo_details (GumSoinfoDetails * details,
                         GumSoinfo * si,
                         GumLinkerApi * api,
                         GHashTable ** ranges)
{
  details->path = gum_resolve_soinfo_path (si, api, ranges);
  details->si = si;
  
  details->body = gum_soinfo_get_body (si);
  details->api = api;
}

static GumSoinfoBody *
gum_soinfo_get_body (GumSoinfo * self)
{
  guint api_level = gum_android_get_api_level ();
  if (api_level >= 26)
  
    return &self->modern.body;
  else if (api_level >= 23)
    return &self->legacy23.body;
  else
    return &self->legacy.legacy23.body;
}



```

写一个简单的 demo, 搞到 app 里面, 这里写在了 init 段中就是，让他人不管是 spawn 或者 attch 都不能寻找符号。

```
void anti7(){
    char line[1024];
    int *start;
    int *end;
    int n=1;
    FILE *fp=fopen("/proc/self/maps","r");
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, "linker64") ) {
            __android_log_print(6,"r0ysue","%s", line);
            if(n==1){
                start = reinterpret_cast<int *>(strtoul(strtok(line, "-"), NULL, 16));
                end = reinterpret_cast<int *>(strtoul(strtok(NULL, " "), NULL, 16));

            }
            else{
                strtok(line, "-");
                end = reinterpret_cast<int *>(strtoul(strtok(NULL, " "), NULL, 16));
            }
            n++;

        }

    }
long* sr= reinterpret_cast<long *>(start);

    mprotect(start,PAGE_SIZE,PROT_WRITE|PROT_READ|PROT_EXEC);
    long off=findsym("/system/bin/linker64","__dl__ZL6somain");
    __android_log_print(6,"r0ysue","xxxxxxxx %p",off);
    long* somain= reinterpret_cast<long*>((char *) sr + off);

    *somain=0;
    void* sb=dlopen("libc.so",RTLD_NOW);
    void* ddd=dlsym(sb,"strstr");
    void* tt=dlopen("libc.so",RTLD_NOW);
    void* ts=dlsym(tt,"strstr");
    __android_log_print(6,"r0ysue","%p",ts);
}

extern "C" void _init(void){
    anti7();

}


```

用下面的 frida 脚本试一下，最后果然崩溃了。

```
function main(){
    var dlopen = Module.findExportByName(null, "android_dlopen_ext");

    Interceptor.attach(dlopen, {
        onEnter: function (arg) {
        var name=ptr(arg[0]).readCString();

        if(name.indexOf("libnative-lib.so")>=0){
               console.log(name)
        this.name=name;
        }

        }, onLeave: function (ret) {

            if(this.name!=undefined){

               var  libcrackme=Module.findBaseAddress("libnative-lib.so");

                console.log(libcrackme);

            }

        }

    })

}
setImmediate(main);


```

![](https://shs3.b.qianxin.com/attack_forum/2022/03/attach-1be4f46112a6510a1feb0f37a2fe712c9f021f70.png)

`anti-frida`与绕过`frida`的手段都在一直的进步，比如早期提出的 so 特征 antifrida 的方式就被 hluda 完美的绕过了，导致很长一段时间内 frida 畅通无阻；后来的从大致的原理出发的 ptrace 与 hook 特征，也有一定的局限性就是太容易被感知到了，比如双进程互相 ptrace 判断，这样 ps 就能知道手法。

最好的方式还是从源码出发，直接以找 bug 的心态阅读 frida 源码，当然前文介绍的这种方式也有一定的局限性，就是如果以 spawn 的方式启动 frida，此时 so 代码是没法影响到`frida-server`的，所以 spwan 去 hook 系统的 so 是确实 anti 不到，算是一个小的遗憾吧，但是只要我们的 app 启动 frida 就没法完成 hook 包括 java hook，总之说了这么多，脚本小子总会被掣肘，想愉快的逆向，最好是能自己开发一个主动调用兼 hook 框架。