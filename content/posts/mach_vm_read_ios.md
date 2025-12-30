---
title: "メモ : mach_vm_*をIOSでつかう"
date: 2019-07-07T23:04:00+09:00
tags: ["mach_vm", "iOS"]
---

<http://bbs.iosre.com/t/write-a-simple-universal-memory-editor-game-trainer-on-osx-ios-from-scratch/115>より.

自分のプロセスあてだったら脱獄せんでも使えるぞ！(多分？)

```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <mach/mach.h>
#include <sys/sysctl.h>
#import <Foundation/Foundation.h>

#if TARGET_OS_IPHONE || TARGET_IPHONE_SIMULATOR // Imports from /usr/lib/system/libsystem_kernel.dylib
extern kern_return_t
mach_vm_read(
vm_map_t        map,
mach_vm_address_t    addr,
mach_vm_size_t        size,
pointer_t        *data,
mach_msg_type_number_t    *data_size);

extern kern_return_t
mach_vm_write(
vm_map_t            map,
mach_vm_address_t        address,
pointer_t            data,
__unused mach_msg_type_number_t    size);

extern kern_return_t
mach_vm_region(
vm_map_t         map,
mach_vm_offset_t    *address,
mach_vm_size_t        *size,        
vm_region_flavor_t     flavor,
vm_region_info_t     info,        
mach_msg_type_number_t    *count,    
mach_port_t        *object_name);
#else
#include <mach/mach_vm.h>
#endif
```

ちなみにプロセスにルート権限を持たせたい（and mach_vm_*を使えるようにする）場合は、以下のようなxmlを作って...

```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>com.apple.springboard.debugapplications</key>
<true/>
<key>get-task-allow</key>
<true/>
<key>proc_info-allow</key>
<true/>
<key>task_for_pid-allow</key>
<true/>
<key>run-unsigned-code</key>
<true/>
<key>platform-application</key>
<true/>
</dict>
</plist>
```

ldid<http://iphonedevwiki.net/index.php/Ldid>に流すと良いぞ！

```
ldid -S<<XML FILE>>.xml <<BINARY FILE>>
```

Memgame IOS版がは開発できる！やった！
