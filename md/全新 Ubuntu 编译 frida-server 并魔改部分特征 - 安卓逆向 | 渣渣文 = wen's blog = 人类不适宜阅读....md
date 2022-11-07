> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [wen2go.site](http://wen2go.site/2022/07/29/20220729/)

```
import lief
```

```
import sys
```

```
import random
```

```
import os
```

```
if __name__ == "__main__":
```

```
input_file = sys.argv[1]
```

```
print(f"[*] Patch frida-agent: {input_file}")
```

```
random_name = "".join(random.sample("ABCDEFGHIJKLMNO", 5))
```

```
print(f"[*] Patch `frida` to `{random_name}``")
```

```
binary = lief.parse(input_file)
```

```
if not binary:
```

```
exit()
```

```
for symbol in binary.symbols:
```

```
if symbol.name == "frida_agent_main":
```

```
symbol.name = "main"
```

```
if "frida" in symbol.name:
```

```
symbol.name = symbol.name.replace("frida", random_name)
```

```
if "FRIDA" in symbol.name:
```

```
symbol.name = symbol.name.replace("FRIDA", random_name)
```

```
all_patch_string = ["FridaScriptEngine","GLib-GIO","GDBusProxy","GumScript"]
```

```
for section in binary.sections:
```

```
if section.name != ".rodata":
```

```
continue
```

```
for patch_str in all_patch_string:
```

```
addr_all = section.search_all(patch_str)
```

```
for addr in addr_all:
```

```
print("current section ,hex(section.file_offset+addr))
```

```
patch = [ ord(n) for n in list(patch_str)[::-1]]
```

```
binary.patch_address(section.file_offset+addr,patch)
```

```
binary.write(input_file)
```

```
random_name = "".join(random.sample("abcdefghijklmn", 11))
```

```
print(f"[*] Patch `gum-js-loop` to `{random_name}`")
```

```
os.system(f"sed -b -i s/gum-js-loop/{random_name}/g {input_file}")
```