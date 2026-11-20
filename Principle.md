# 华为平板鼠标中键“返回”问题的定位与修复

> 记录日期：2026-11-17
> 设备：Huawei MatePad 11（2021，DBY-W09，Snapdragon 865）
> 系统：Android + KernelSU（susfs），内核 4.19.157-perf
> 模块：**Mouse Middle Button Fix** v1.3（id `mouse_middle_fix`）
> 结论：**华为在 `libinputreader.so` 里给鼠标中键用了私有编码 `0x100`。补一条 `getButtonState` 指令、加一个读属性的 trampoline，就能把中键恢复成 Android 标准的 `BUTTON_TERTIARY(0x4)`，并且支持不重启热切换。**

---

## 0. TL;DR

| 项目 | 内容 |
|---|---|
| 现象 | 接鼠标后，中键在**系统全局**触发“返回”；所有 App 都收不到中键（无 `BUTTON_TERTIARY`） |
| 链路 | `getButtonState(true)` 返回 `0x100` → `CursorInputMapper` 看 bit 8 → 生成私有键码 **747** → `MouseScrollWheelHandler` 注入 `KEYCODE_BACK` |
| 根因 | 华为把中键的 tertiary 编码从标准 `0x4` 改成私有 `0x100`；`getButtonState` 里用厂商加的 `bool` 参数二选一 |
| 修复 | 把 `getButtonState` 里那条 `tst w1,#1` 替换成跳转，落到 `.plt` 后页内空白处的 trampoline；由属性决定返回 `0x4` 还是 `0x100` |
| 开关 | `persist.mouse.middle_fix`：`1`=标准中键（默认），`0`=华为原厂（中键=返回） |
| 生效 | `post-fs-data.sh` 在 zygote 前用 bind mount 覆盖 `/system/lib64/libinputreader.so`；属性在**每次** `getButtonState` 调用时读取，所以切换即时生效，无需重启 |
| 结果 | ON 时中键不再触发返回、App 能收到标准中键；OFF 时完整还原原厂行为 |

---

## 1. 现象与环境

### 1.1 现象

- 平板接 USB / 2.4G 鼠标后，按中键 = 返回上一级，**在任何 App 里都一样**。
- 右键、滚轮、侧键正常。
- 用 `getevent` 能看到中键事件（`BTN_MIDDLE`，code 274）确实到达了系统，但 App 侧收不到标准中键。

### 1.2 环境

```text
设备：Huawei MatePad 11 (2021) DBY-W09
SoC：Snapdragon 865 / Adreno 650
内核：4.19.157-perf
系统：Android + KernelSU (susfs)
关键库：/system/lib64/libinputreader.so
原厂 SHA-256：82c3be7dd7a973b93887ecb4f2c8cea0b146b27c4ecce1aee7e684294c0ac5b6
```

---

## 2. 定位过程

### 2.1 第一步：抓“谁在发返回”

在平板上清日志，然后按一次中键：

```sh
su -c 'logcat -c'
# 按中键
su -c 'logcat -d -s MouseScrollWheelHandler:I'
```

看到：

```text
I MouseScrollWheelHandler: handleSingleTap
```

`MouseScrollWheelHandler` 是 `com.android.server.policy` 里的类，专门处理“鼠标滚轮键”。它收到某个键后调 `handleSingleTap()` 注入 `KEYCODE_BACK`。到这里可以确定：**返回键是系统策略层注入的**，不是某个 App 的行为。

### 2.2 第二步：747 是谁产生的

继续看 `InputDispatcher` / `InputReader` 的按键日志，能看到中键按下时系统里出现的是：

```text
keyCode=747  (KEYCODE_BUTTON_MOUSE_SCROLL_WHEEL)
```

747 不是标准鼠标键，而是 Android 给“鼠标滚轮当按键”预留的私有键码。`MouseScrollWheelHandler` 只认这个键码，所以只要 747 还在，返回就一定会发生。

那 747 是谁发的？在 `libinputreader.so` 里搜 `747` / `0x2EB`，跟到 `CursorInputMapper`：

- `CursorInputMapper` 读取 `CursorButtonAccumulator::getButtonState()` 的返回值；
- 如果返回值里带 **bit 8（0x100）**，就把它翻译成 `KEYCODE_BUTTON_MOUSE_SCROLL_WHEEL(747)`；
- 也就是说，**中键被编码成了 0x100**，而不是 Android 标准的 `BUTTON_TERTIARY(0x4)`。

### 2.3 第三步：0x100 是谁写的

反汇编 `libinputreader.so` 的 `CursorButtonAccumulator::getButtonState`，关键片段（原厂逻辑）：

```asm
732d4:  orr  w9, w8, #0x4        ; 候选 A：state | BUTTON_TERTIARY (0x4)
732d8:  orr  w8, w8, #0x100      ; 候选 B：state | 0x100（华为私有）
732dc:  tst  w1, #1              ; 厂商加的 bool 参数
732e0:  csel w8, w9, w8, eq      ; bool 为 false → A(0x4)，true → B(0x100)
```

而调用点在 `CursorInputMapper::sync`：

```asm
74fb8:  tbnz w0, #0, ...
74fbc:  mov  w1, #1              ; 固定传 true
74fc0:  add  x0, x19, #0x28
74fc4:  bl   CursorButtonAccumulator::getButtonState(bool)
```

**鼠标路径固定传 `true`，于是永远走 0x100 分支。** 这就是根因：华为给 `getButtonState` 加了一个 AOSP 没有的 `bool` 参数，用它在“标准 0x4”和“私有 0x100”之间切换，而鼠标调用点选了后者。

### 2.4 完整链路

```text
硬件 BTN_MIDDLE (code 274)
        │
        ▼
CursorButtonAccumulator::getButtonState(true)
        │  华为分支：state | 0x100
        ▼
CursorInputMapper::sync
        │  检测 bit 8 (0x100)
        ▼
KeyEvent KEYCODE_BUTTON_MOUSE_SCROLL_WHEEL (747)
        │
        ▼
system_server: com.android.server.policy.MouseScrollWheelHandler
        │  单次点击 → inject KEYCODE_BACK
        ▼
全局“返回”
```

副作用：MotionEvent 里带的是 `0x100`，不是 `BUTTON_TERTIARY(0x4)`，所以浏览器“中键新标签页”、应用里的中键手势全部失效——**只去掉返回键是不够的，必须把 0x4 还给应用**。

---

## 3. 根因：一个被厂商改写的 tertiary 编码

Android 标准里，鼠标键位是 `MotionEvent.BUTTON_*`：

| 键 | 标准值 |
|---|---|
| `BUTTON_PRIMARY`（左） | `0x1` |
| `BUTTON_SECONDARY`（右） | `0x2` |
| `BUTTON_TERTIARY`（中） | `0x4` |
| `BUTTON_BACK` | `0x8` |
| `BUTTON_FORWARD` | `0x10` |

华为把中键写成 `0x100`，这是一个**没有公开语义的私有位**。它自己的 `CursorInputMapper` / `MouseScrollWheelHandler` 认这个位，于是把中键变成了返回；但标准 App 只认 `0x4`，所以中键在应用层“消失”。

修复思路因此很明确：**不要改 747（那是症状），要在 `getButtonState` 里把 tertiary 改回 0x4（这是病根）**。

---

## 4. 失败的尝试，以及为什么失败

### 4.1 把 747 改成 0 或别的键码

第一反应是“让 `MouseScrollWheelHandler` 收不到 747”：

- 把生成 747 的分支改成 `0`：返回键确实不触发了；
- 或者把 747 改成 `0xBC`（`KEYCODE_BUTTON_1`）：返回也没了。

但**中键依然无效**。原因：`0x100` 还在，MotionEvent 里依然没有 `BUTTON_TERTIARY`，App 还是不认。这说明改键码只是把“返回”这个症状按下去，没有恢复中键语义。

### 4.2 改调用点的 `mov w1,#1` → `#0`

既然 `bool=true` 走 0x100，那把调用点改成 `false` 不就行了？

实测：**返回键仍然触发**。说明这条调用路径不是唯一入口（或者该 `bool` 还影响别的状态），改调用点不可靠，放弃。

### 4.3 直接把 `orr w8,w8,#0x100` 改成 `#0x4`

这个能工作：两个 `csel` 候选都变成 `0x4`，中键恢复标准语义、返回消失（v1 静态补丁就是这么干的）。

但它有个硬伤：**改完必须重启 system_server 才生效**，而且没法临时切回原厂行为验证。用户明确要求“尽量用 KernelSU 模块、尽量不重启”，于是有了最终方案。

### 4.4 教训

- 修输入问题要修**事件语义**（button state），不要修**下游键码**；
- 厂商改动往往藏在一个不起眼的 `bool` 参数里；
- 先做静态补丁验证可行性，再把它改造成可切换的。

---

## 5. 最终方案：一条指令 + 页内 trampoline

### 5.1 设计目标

1. ON：`getButtonState` 永远返回 `state | 0x4`（标准中键，无 747，无返回）；
2. OFF：完整保留原厂逻辑（`state | 0x100`，中键=返回）；
3. 切换不需要重启，下一个输入事件就生效；
4. 不修改 `/system`、`/vendor` 任何文件，可随时卸载恢复。

### 5.2 补丁位置

只改 `getButtonState` 里的 `tst w1,#1` 这一条（文件偏移 `0x732dc`）：

```asm
; 原厂
732dc:  7200003f   tst  w1, #1
732e0:  1a880128   csel w8, w9, w8, eq

; 补丁后
732dc:  1400d679   b    0xa8cc0        ; 跳到 trampoline
732e0:  1a880128   csel w8, w9, w8, eq ; 保留但不再执行（trampoline 直接回 0x732e4）
```

注意补丁点选的是 `tst`，不是 `orr #0x100`：

- 进入 trampoline 时，`w8` 已经是 `state | 0x100`（上一条 `orr` 算好的）；
- `w9` 已经是 `state | 0x4`；
- 所以 trampoline 只要决定“保留 w8 还是换成 w9”，然后跳回 `0x732e4` 即可，**OFF 路径等价于原厂 `csel` 选 B**。

### 5.3 trampoline

放在 `.plt` 结束后的页内空白（`0xa8cc0`，见 5.5）：

```asm
a8cc0:  sub  sp, sp, #0x30
a8cc4:  str  x0, [sp]            ; 保存 x0（后面还要 ldrb [x0]）
a8cc8:  str  x8, [sp, #8]        ; 保存 w8 = state|0x100
a8ccc:  str  x9, [sp, #16]       ; 保存 w9 = state|0x4
a8cd0:  str  x30, [sp, #24]      ; 保存 LR
a8cd4:  adrp x0, 0xa8000
a8cd8:  add  x0, x0, #0xd08      ; x0 = "persist.mouse.middle_fix"
a8cdc:  mov  w1, #1              ; 默认值 1（ON）
a8ce0:  bl   property_get_int32@plt
a8ce4:  mov  w10, w0             ; 结果先放到 w10
a8ce8:  ldr  x0, [sp]
a8cec:  ldr  x8, [sp, #8]
a8cf0:  ldr  x9, [sp, #16]
a8cf4:  ldr  x30, [sp, #24]
a8cf8:  add  sp, sp, #0x30
a8cfc:  cbz  w10, 0xa8d04       ; 0 → OFF，保留 w8(0x100)
a8d00:  mov  w8, w9              ; 非 0 → ON，w8 = state|0x4
a8d04:  b    0x732e4            ; 回到原函数
```

属性字符串放在 `0xa8d08`：

```text
persist.mouse.middle_fix\0
```

`adrp + add` 的 `#0xd08` 正好指向它（`0xa8000 + 0xd08 = 0xa8d08`）。

几个关键细节：

- **先保存再调用**：`property_get_int32` 会破坏 `x0–x18`，所以 `x0/x8/x9/x30` 必须在 `bl` 之前入栈。最早版本先调用后保存，结果函数返回值被属性值覆盖，行为直接错乱。
- **结果放 `w10`**：恢复 `x8/x9` 之后再判断，避免把属性值当成按钮状态。
- **`cbz` 而非 `cbnz`**：属性默认 1，只有显式设 0 才走 OFF。
- **不需要保存 NZCV**：`0x732e4` 之后的指令会重新 `cmp`，不依赖进入 trampoline 前的标志位。
- **栈 48 字节**：保持 16 字节对齐，满足 AAPCS。

### 5.4 字节级清单

| 位置（文件偏移 = vaddr） | 原厂 | 补丁 |
|---|---|---|
| `0x732dc` | `3f 00 00 72`（`tst w1,#1`） | `79 d6 00 14`（`b 0xa8cc0`） |
| `0xa8cc0` | 空白（0x00） | trampoline，0x48 字节 |
| `0xa8d08` | 空白（0x00） | `"persist.mouse.middle_fix\0"`，后面补 0 |
| ELF `PT_LOAD[2].p_filesz`（文件偏移 `0xd0`） | `0x63cc0` | `0x64000` |
| ELF `PT_LOAD[2].p_memsz`（文件偏移 `0xd8`） | `0x63cc0` | `0x64000` |

trampoline 的 72 字节：

```text
ff c3 00 d1 e0 03 00 f9 e8 07 00 f9 e9 0b 00 f9
fe 0f 00 f9 00 00 00 90 00 20 34 91 21 00 80 52
00 f7 ff 97 ea 03 00 2a e0 03 40 f9 e8 07 40 f9
e9 0b 40 f9 fe 0f 40 f9 ff c3 00 91 4a 00 00 34
e8 03 09 2a 78 29 ff 17
```

### 5.5 为什么选 `0xa8cc0`

- `.text` 的最后一个 section（`.plt`）在 `0xa8cc0` 结束；
- 下一个 section `.data.rel.ro` 从 `0xa9000` 开始；
- 中间有 `0x340` 字节的页内空隙，文件里是 0，且不属于任何 section；
- 但原厂 `PT_LOAD[2]` 只覆盖到 `0xa8cc0`（`p_offset=0x45000`，`p_filesz=0x63cc0`），这段空隙**没有被映射**，直接放代码会段错误；
- 把 `PT_LOAD[2]` 的 `filesz/memsz` 扩到 `0x64000`（结束于 `0xa9000`），这段就变成可执行段的一部分；
- section header 不用改——动态加载器只认 program header。副作用是 `objdump -d` 不会反汇编 trampoline（没有 section 覆盖），要用 `-D -b binary` 或 gdb 看。

这是典型的“ELF 页内 slack + 扩段”补丁法：不新增 section、不挪任何原有字节、不动重定位。

### 5.6 运行期开关

属性：

```text
persist.mouse.middle_fix    1 = ON（默认）, 0 = OFF
```

- `getButtonState` 每次被调用都会读一次属性，所以切换后**下一个鼠标事件**就生效；
- `persist.` 前缀保证重启后仍保留；
- 默认值 1：属性不存在时 `property_get_int32` 返回默认值 1，模块刚装上就是修复状态。

---

## 6. KernelSU 模块

### 6.1 目录结构

```text
/data/adb/modules/mouse_middle_fix/
├── module.prop                 # id=mouse_middle_fix, name=Mouse Middle Button Fix
├── post-fs-data.sh             # zygote 前 bind mount + 哈希守卫 + 初始化开关
├── action.sh                   # KSU “执行”按钮：切换开关 + 刷新描述
├── service.sh                  # 每 15 秒同步描述
├── libinputreader_mmfix.so     # 打过补丁的输入库
└── README.md
```

### 6.2 挂载流程（post-fs-data.sh）

```sh
SRC="$MODDIR/libinputreader_mmfix.so"
DST="/system/lib64/libinputreader.so"

# 1. 原厂库哈希必须匹配，否则跳过（OTA 保护）
CUR=$(sha256sum "$DST" | awk '{print $1}')
[ "$CUR" = "$STOCK_SHA256" ] || exit 0

# 2. 补丁库自身哈希必须匹配，防止半截文件
PATCHED=$(sha256sum "$SRC" | awk '{print $1}')
[ "$PATCHED" = "$PATCHED_SHA256" ] || exit 0

# 3. 在 init 的 mount namespace 里 bind，保证 system_server 能看到
nsenter -t 1 -m -- mount --bind "$SRC" "$DST"
```

为什么必须 `nsenter -t 1 -m`：`system_server` 继承的是 init 的 mount namespace；只在模块脚本自己的 namespace 里 mount，zygote 和 `system_server` 看不到。

### 6.3 哈希守卫

| 哈希 | 值 |
|---|---|
| 原厂 `libinputreader.so` | `82c3be7dd7a973b93887ecb4f2c8cea0b146b27c4ecce1aee7e684294c0ac5b6` |
| 补丁库 | `a9b282487e494eecad6a4d5b8033e06900f99e33ac5703b87dd6a15d3a166626` |

系统 OTA 一旦更新 `libinputreader.so`，哈希不匹配，模块**静默跳过挂载**，绝不会拿旧库覆盖新库。代价是升级后补丁失效，需要重新对新库做偏移分析。

### 6.4 状态显示

KernelSU 模块列表的 `description` 直接显示状态：

- `🟢 中键修复已开启：标准中键，不触发返回`
- `🔴 中键修复已关闭：中键变返回（华为原厂）`

三个入口保证它不飘：

| 脚本 | 时机 | 作用 |
|---|---|---|
| `post-fs-data.sh` | 开机 | 按属性校正描述 |
| `action.sh` | 点“执行” | 切换后立即改写描述 |
| `service.sh` | 开机后常驻 | 每 15 秒对齐一次，兜住手动 `setprop` |

`update_desc()` 用 `grep -qF` 先比较，内容没变就不写文件，避免无意义地刷 `module.prop`。

### 6.5 热切换原理

```text
action.sh: setprop persist.mouse.middle_fix 0/1
                    │
                    ▼
下一次鼠标事件 → getButtonState → trampoline → property_get_int32
                    │
        ┌───────────┴───────────┐
       1│                       │0
        ▼                       ▼
   w8 = state|0x4         w8 = state|0x100
   标准中键                华为原厂（返回）
```

因为分支在 `getButtonState` 内部，而它每个输入事件都会被调用，所以**不需要重启 system_server，也不需要重新 dlopen**。这也是“在 .so 里做分支 + 属性控制”相比“换库/改调用点”的最大优势。

---

## 7. 验证

### 7.1 挂载与哈希

```sh
su -c 'nsenter -t 1 -m -- grep inputreader /proc/self/mountinfo'
su -c 'sha256sum /system/lib64/libinputreader.so'
su -c 'cat /data/local/tmp/mouse_middle_fix.log'
```

实测：

```text
191 42 259:24 /adb/modules/mouse_middle_fix/libinputreader_mmfix.so \
  /system/lib64/libinputreader.so rw,nosuid,nodev,noatime shared:38 ...
a9b282487e494eecad6a4d5b8033e06900f99e33ac5703b87dd6a15d3a166626
=== post-fs-data ===
bind ok (init ns)
```

### 7.2 行为测试（不需要真的按鼠标）

找到鼠标的 event 节点（重启后编号会变）：

```sh
su -c 'grep -B2 -A5 "Wireless Receiver Mouse" /proc/bus/input/devices'
# 记下 Handlers=eventN
```

注入一次中键：

```sh
su -c '
logcat -c
sendevent /dev/input/event4 1 274 1   # BTN_MIDDLE down
sendevent /dev/input/event4 0 0 0     # SYN
sendevent /dev/input/event4 1 274 0   # BTN_MIDDLE up
sendevent /dev/input/event4 0 0 0
sleep 2
logcat -d -s MouseScrollWheelHandler:I
'
```

| 开关 | 预期 `MouseScrollWheelHandler` 日志 | 实测 |
|---|---|---|
| ON (`1`) | 无输出 | ✅ 无输出 |
| OFF (`0`) | `handleSingleTap` | ✅ `handleSingleTap` |

ON 时无日志 = 没有 747 = 没有注入返回；OFF 时有日志 = 完整复现原厂行为。

### 7.3 开关与描述

```sh
su -c 'sh /data/adb/modules/mouse_middle_fix/action.sh'   # OFF
su -c 'sh /data/adb/modules/mouse_middle_fix/action.sh'   # ON
```

实测两次输出与描述：

```text
Mouse middle fix: OFF
Huawei original behavior: middle button = Back (tertiary 0x100).
description=🔴 中键修复已关闭：中键变返回（华为原厂）

Mouse middle fix: ON
Middle button = Android BUTTON_TERTIARY (0x4), no Back key.
description=🟢 中键修复已开启：标准中键，不触发返回
```

`ps` 里能看到 `service.sh` 常驻；`getprop persist.mouse.middle_fix` 返回 `1`。

---

## 8. 踩过的坑

### 8.1 bind mount 的源文件被“就地更新”会回退

改库时如果直接覆盖模块目录里**已经被 bind mount 的同一个文件**，出现过重启后内容回退、挂载还是旧库的怪现象。后来每次改库都用**新文件名**（`libinputreader.so` → `libinputreader_switch.so` → `libinputreader_mmfix.so`），重新 bind，问题消失。

经验：模块里被 bind mount 的源文件，尽量只增不改；要改就换名。

### 8.2 trampoline 先调用后保存，返回值被覆盖

第一版 trampoline：

```asm
bl property_get_int32
str x0, [sp]     ; 错：x0 已经被属性调用破坏
...
```

结果 `getButtonState` 返回的是属性值（1/0），按钮状态全乱。正确顺序是**进入 trampoline 先保存 x0/x8/x9/x30，再调用**，并把属性结果单独放 `w10`，恢复完再判断。

### 8.3 `sed` 替换库文件名时误改了目标路径

用 `sed 's|libinputreader\.so|libinputreader_v2.so|'` 批量改脚本，把 `DST="/system/lib64/libinputreader.so"` 也改成了 `libinputreader_v2.so`，挂载目标直接错。教训：**源文件名和目标路径分开维护**，不要对整份脚本做无差别替换。

### 8.4 杀 zygote / system_server 会黑屏甚至起不来

调试时为了重载库杀过 zygote，结果黑屏、反复重启。正确做法：需要换库就正常重启；能热切换的（本方案）就不要碰进程。模块本身的设计目标之一就是**避免为了切换而重启**。

### 8.5 重启后 event 节点编号会变

`/dev/input/eventN` 的 N 不固定，测试脚本不要写死。每次从 `/proc/bus/input/devices` 里按设备名重新取。

### 8.6 `objdump -d` 看不到 trampoline

因为 trampoline 不在任何 section 里（只靠 program header 映射）。验证时要：

```sh
dd if=libinputreader_mmfix.so of=/tmp/t.bin bs=1 skip=$((0xa8cc0)) count=$((0x48))
aarch64-linux-gnu-objdump -D -b binary -m aarch64 --adjust-vma=0xa8cc0 /tmp/t.bin
```

---

## 9. 适用范围、兼容性与回退

### 9.1 适用范围

- 已验证：MatePad 11 (2021) DBY-W09，原厂库哈希 `82c3be7d...`；
- 同代华为/荣耀设备如果 `libinputreader.so` 哈希一致，可直接用；
- 哈希不同则模块自动跳过，不会破坏系统；
- 纯 Android 输入层修复，**全局生效**。

### 9.2 回退

三种粒度：

1. **临时**：KSU 模块“执行”切到 OFF，立即恢复原厂行为，不用重启；
2. **停用**：`touch /data/adb/modules/mouse_middle_fix/disable` 后重启；
3. **卸载**：删除模块目录后重启，`/system` 分区从未被修改。

### 9.3 风险

- 依赖原厂库的精确指令布局，OTA 后需重新分析；
- trampoline 在可执行段页内 slack，若未来库布局变化（`.plt` 结束位置不同），需要重新找空白；
- 只改一个函数、一条指令，影响面极小；OFF 路径与原厂 `csel` 语义等价。

---

## 10. 文件与哈希

| 文件 | 说明 | SHA-256 |
|---|---|---|
| `libinputreader_mmfix.so` | 带运行期开关的补丁库 | `a9b282487e494eecad6a4d5b8033e06900f99e33ac5703b87dd6a15d3a166626` |
| `mouse_middle_fix-v1.3.zip` | KernelSU 刷机包 | `999b5b7713b4d8a33d2cb97f2e005e1c1254afb74eaec386d22d67dab756dbb4` |
| 原厂 `libinputreader.so` | 补丁基线 | `82c3be7dd7a973b93887ecb4f2c8cea0b146b27c4ecce1aee7e684294c0ac5b6` |

模块信息：

```ini
id=mouse_middle_fix
name=Mouse Middle Button Fix
version=1.3
versionCode=4
```

---

## 11. 批量适配：特征与一键脚本

前面是“本机 + 动态开关”的定制方案。如果要在很多华为设备上批量修，可以不用 trampoline：
直接把这个函数里生成 `0x100` 的那条指令改成生成 `0x4`，只动 4 个字节，行为等价于“开关永远 ON”。

### 11.1 可机器识别的特征

**锚点**：`libinputreader.so` 导出符号

```text
_ZNK7android23CursorButtonAccumulator14getButtonStateEb
```

它是 `GLOBAL DEFAULT` 的导出符号，普通 strip 不会删掉；本机两份库都有：

| 库 | 符号值 | size |
|---|---|---|
| `/system/lib64/libinputreader.so` | `0x732b8` | 96 |
| `/system/lib/libinputreader.so` | `0x52d7d`（Thumb，bit0=1） | 66 |

**指令指纹**：

| 架构 | 特征 | 掩码 / 基址 |
|---|---|---|
| arm64 | `orr Wd,Wn,#0x4` 紧邻 `orr Wd,Wn,#0x100` | `(w & 0xFFFFFC00) == 0x321E0000` / `== 0x32180000` |
| arm32 | `cmp rN,#0` + `bf0c` + `orr #0x4` + `add/orr #0x100` | `cmp`：`(h & 0xF800)==0x2800`；`orr #4`：`0xF0400004`；`add #0x100`：`0xF5007080`；`orr #0x100`：`0xF4407080`；寄存器掩码 `0xFFF0F0FF` |

补丁公式（保留寄存器位）：

```text
arm64:  new = 0x321E0000 | (old & 0x3FF)                 # orr Wd,Wn,#0x100 -> #0x4
arm32:  new = ((0xF040|Rn)<<16) | (0x0004 | (Rd<<8))     # add/orr #0x100 -> orr #0x4
```

### 11.2 一键脚本 [`huawei_mmb_patch.py`](huawei_mmb_patch.py)

单文件、仅 Python 3 标准库，输入原厂库，输出静态 Magisk/KernelSU 模块：

```bash
# 单个库（架构自动识别，目标路径自动推断）
python3 huawei_mmb_patch.py libinputreader.so

# 64 位 + 32 位打进同一个模块
python3 huawei_mmb_patch.py --so lib64.so --so lib32.so

# 只检测，不写文件
python3 huawei_mmb_patch.py --inspect libinputreader.so
```

输出：

```text
[+] libinputreader.so: patched 0x732d8 (arm64)
    stock   82c3be7d...
    patched 4eeb4446...
[+] module dir: out/huawei_mmb_fix_82c3be7d
[+] zip       : huawei_mmb_fix_arm64_82c3be7d.zip
```

生成的模块是**静态**的：`post-fs-data.sh` 先校验原厂/补丁哈希，再 `nsenter -t 1 -m -- mount --bind`；没有 `action.sh`、`service.sh`、属性开关。

### 11.3 已验证基线

| 产物 | SHA-256 |
|---|---|
| arm64 原厂 | `82c3be7dd7a973b93887ecb4f2c8cea0b146b27c4ecce1aee7e684294c0ac5b6` |
| arm64 静态补丁 | `4eeb444608e03e4088ccab9ec365e9b15614aa4cce90541c184fc4000e5fe0dc` |
| arm32 原厂 | `5fecdc388eb9f7232a3ad84a5b440cbe59a34b213c11a6563cb1c44f27963875` |
| arm32 静态补丁 | `91bdf40665ce002709767ba55c63705c5c3f1e3f85a95077789c4203e66487ae` |

arm32 的补丁点在本机 `/system/lib/libinputreader.so` 上验证：补丁后反汇编为 `orreq r0,r0,#4` / `orrne r0,r0,#4`，两条路径都返回 `0x4`。

### 11.4 适用范围与限制

- 支持 **AArch64** 与 **ARM32 Thumb**；ARM 模式（符号 bit0=0）会明确拒绝。
- 符号存在时按函数范围扫描；符号缺失（被 strip）时退化为全 `.text` 指纹扫描，要求**唯一匹配**。
- 全 `.text` 多候选、函数内多候选、找不到 `0x100`、输入已补丁：一律拒绝或提示，不盲改。
- 只适用于 `libinputreader.so`；OTA 更新后原厂哈希变化，生成的模块会自动跳过挂载，需要重新跑脚本。
- 拉取原厂库前要确认当前没有其它模块 bind-mount 了 `libinputreader.so`（例如先停用动态模块并重启）；否则拿到的是补丁库，生成的模块会因原厂哈希不匹配而跳过。
- 生成的静态模块与本文的动态模块 `mouse_middle_fix` 互斥（都挂同一个目标），只能二选一。

### 11.5 遇到不支持的 build

先跑 `--inspect`；如果报“未找到厂商 0x100 特征”或多候选，把以下信息一起反馈，便于补指纹：

```bash
sha256sum libinputreader.so
python3 huawei_mmb_patch.py --inspect libinputreader.so
# 以及该库的 getButtonState 反汇编（可选）
```

### 11.6 本机示例

已用脚本为本机生成 `huawei_mmb_fix_arm64_arm32_82c3be7d.zip`（含 64 位和 32 位两份补丁），
SHA-256：`edfb416316438a3a5fc231b699138451dc376ed150870e9e98bef2d1b188702a`。
刷之前先停用动态模块 `mouse_middle_fix`，两者不能同时挂载同一个目标。

---

## 附录 A：原厂 vs 补丁反汇编

```asm
; ---- 原厂 getButtonState ----
732b8:  ldrb w8, [x0]
732bc:  ldrb w9, [x0, #1]
732c0:  orr  w10, w8, #0x2
732c4:  cmp  w9, #0x0
732c8:  ldrb w9, [x0, #2]
732cc:  csel w8, w8, w10, eq
732d0:  cbz  w9, 732e4
732d4:  orr  w9, w8, #0x4          ; 标准 tertiary
732d8:  orr  w8, w8, #0x100        ; 华为私有 tertiary
732dc:  tst  w1, #1               ; 厂商 bool
732e0:  csel w8, w9, w8, eq       ; false→0x4, true→0x100
732e4:  ldrb w9, [x0, #3]
...

; ---- 补丁后 ----
732d4:  orr  w9, w8, #0x4
732d8:  orr  w8, w8, #0x100
732dc:  b    0xa8cc0              ; → trampoline
732e0:  csel w8, w9, w8, eq       ; 保留但不再执行
732e4:  ldrb w9, [x0, #3]
```

```asm
; ---- trampoline @ 0xa8cc0 ----
a8cc0:  sub  sp, sp, #0x30
a8cc4:  str  x0, [sp]
a8cc8:  str  x8, [sp, #8]
a8ccc:  str  x9, [sp, #16]
a8cd0:  str  x30, [sp, #24]
a8cd4:  adrp x0, 0xa8000
a8cd8:  add  x0, x0, #0xd08
a8cdc:  mov  w1, #1
a8ce0:  bl   0xa68e0              ; property_get_int32@plt
a8ce4:  mov  w10, w0
a8ce8:  ldr  x0, [sp]
a8cec:  ldr  x8, [sp, #8]
a8cf0:  ldr  x9, [sp, #16]
a8cf4:  ldr  x30, [sp, #24]
a8cf8:  add  sp, sp, #0x30
a8cfc:  cbz  w10, 0xa8d04         ; OFF
a8d00:  mov  w8, w9               ; ON: state|0x4
a8d04:  b    0x732e4
a8d08:  "persist.mouse.middle_fix"
```

---

## 附录 B：复现补丁的关键步骤

> 仅适用于上表哈希的原厂库；偏移不同不要硬套。

1. 复制原厂 `libinputreader.so`；
2. 在 `0x732dc` 写入 `79 d6 00 14`（`b 0xa8cc0`）；
3. 在 `0xa8cc0` 写入 72 字节 trampoline（见 5.4）；
4. 在 `0xa8d08` 写入 `persist.mouse.middle_fix\0`，剩余补 0；
5. 把 ELF 偏移 `0xd0` / `0xd8` 处的 `p_filesz` / `p_memsz` 改成 `0x64000`；
6. 用 `aarch64-linux-gnu-objdump` 复核指令，`readelf -l` 确认 LOAD 到 `0xa9000`；
7. 更新模块里的 `PATCHED_SHA256`，换新文件名，重新打包。

---

## 附录 C：常用命令速查

```sh
# 当前开关
su -c 'getprop persist.mouse.middle_fix'

# 模块日志 / 挂载 / 库哈希
su -c 'cat /data/local/tmp/mouse_middle_fix.log'
su -c 'nsenter -t 1 -m -- grep inputreader /proc/self/mountinfo'
su -c 'sha256sum /system/lib64/libinputreader.so'

# 手动切换（立即生效）
su -c 'setprop persist.mouse.middle_fix 1'   # ON
su -c 'setprop persist.mouse.middle_fix 0'   # OFF

# 找鼠标 event 节点
su -c 'grep -B2 -A5 "Wireless Receiver Mouse" /proc/bus/input/devices'

# 观察返回注入
su -c 'logcat -s MouseScrollWheelHandler:I'
```
