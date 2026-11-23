# chflags 在 RootHide vroot 下二次路径翻译的初步分析

| 项目 | 内容 |
| --- | --- |
| 日期 | 2026-11-23 |
| 状态 | 初步分析（只读排查，未修改源码或构建产物） |
| 对象 | `file-cmds` 430.100.5 中的 `chflags(1)`；OwnGoal Stripe / Procursus-roothide 构建链 |
| 触发场景 | 设备上执行 `com.nan.disableupdates` 的 postinst，对 `/rootfs/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/` 设置 `schg` |
| 结论置信度 | 高（源码、二进制绑定、官方 bootstrap 对照均已检查）；截图个别报错路径仍建议在设备上复核 |

## 0. 摘要

1. **`chflags.c` 没有被改坏。** Apple `file_cmds` 400 与 430.100.5 的 `chflags.c` 逐字节相同；本仓库对 `file-cmds` 的唯一补丁是 `compress` 的 stdio 处理，不涉及 `chflags`。
2. **问题出在构建后的 `symredirect`。** 我们重新构建的 `chflags` 把 `_chflags`、`_lchflags`、`_fts_open` 全部重定向到 `libvrootapi`；roothide 官方 bootstrap 的 `chflags` 只重定向 `_lchflags` 和 `_fts_open`，`_chflags` 仍绑定 `libSystem`。
3. **根因是 vroot 路径被翻译两次。** `fts_open()` 先把 `/rootfs/var/...` 翻成真实 `/var/...`，`chflags()`/`lchflags()` 再把已经翻好的绝对路径当作 jbroot 路径，加上 `<jbroot>` 前缀，得到不存在的影子路径。
4. **影响面不止 `chflags(1)`。** 符号层还影响 `compress`、`login`、`pw`，以及 iPhone 侧的 `libpython`、`libarchive`、`libxar` 等；但只有“先经 `fts_open`/`realpath` 等翻译、再调用 `chflags`”的工具会确定性二次翻译，目前 `chflags(1)` 是明确命中的案例。
5. **最终源复核结论。** 在 Rachel 归档容器的最终索引上扫描 1,486 个 iOS Deb，符号层命中 8 个二进制 / 6 个包；**确定性受影响的实际只有 1 个包：`file-cmds`（`usr/bin/chflags`）**。`mtree-netbsd` 是相对路径导致的误报，`python3` 是调用方写法相关的潜在风险，其余为直接路径调用。
6. **建议修复位置在 shim 层。** 对 `usr/bin/chflags` 这类自身已经通过 `fts_open` 完成路径翻译的二进制，跳过 `_chflags`/`_lchflags` 重定向；不要只改 `file-cmds` 源码。修复后应同时覆盖非递归和 `-R` 两种路径。

### 0.1 分析范围与方法

已检查：

- Apple `file_cmds` 400 与 430.100.5 的 `chflags.c` 源码差异；
- 本仓库 `file-cmds` 配方、`AFTER_BUILD`/`symredirect` 调用链、宿主 `symredirect` 的构建来源；
- 本仓库的 `file-cmds_430.100.5_iphoneos-arm64e.deb`、tvOS `bootstrap_1900.tar.zst`；
- roothide 官方 `bootstrap-1900.tar.zst` 中的 `chflags` 与 `libvroot*`；
- 用宿主 `symredirect` 对官方 `chflags` 副本做对照处理。

未覆盖：

- 未直接登录设备检查实际安装的 `/usr/bin/chflags`（以截图和构建产物推断）；
- 未在设备上执行 roothide 官方 bootstrap 的 `-R` 行为；
- 未对 iPhone 正式 Bootstrap 的完整包集做穷举扫描。

## 1. 现象与复现

截图中的关键命令与结果：

```sh
chflags schg,schange,simmutable /rootfs/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/
# chflags: /var/containers/Bundle/Application/.jbroot-<BRAND>/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/: No such file or directory

ls /var/MobileSoftwareUpdate/MobileAsset/AssetsV2/
# ls: cannot access '/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/': No such file or directory

ls /rootfs/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/
# （成功，空目录）
```

直接结论：

- `/rootfs/var/.../AssetsV2` 在真实 rootfs 上存在；`ls` 成功说明不是目录缺失。
- `/var/.../AssetsV2` 作为 jbroot 相对路径不存在；这符合 roothide 的路径规则。
- `chflags` 报错里出现 `.jbroot-...`，说明失败发生在 vroot 翻译之后的 jbroot 影子路径，而不是 `chflags` 对真实 rootfs 没有权限。

postinst 还包含 `chflags -R noschg,noschange,nosimmutable /rootfs/var/MobileSoftwareUpdate/MobileAsset/`。按同一机制，`-R` 的绝对路径根节点也存在二次翻译风险，需要和第二步一起回归；第二步的非递归 `schg` 设置是当前包能否阻止 OTA 的关键。

## 2. 对照分析

### 2.1 源码层：`chflags.c` 相同

对照 Apple 开源 `file_cmds` 的两个 tag：

```text
file_cmds-400/chflags/chflags.c
file_cmds-430.100.5/chflags/chflags.c
```

`diff` 为空，两者均为 247 行，`chflagsat()` shim、`fts_open()` 遍历、`chflags()/lchflags()` 调用和错误输出逻辑完全一致。

仓库内相关文件：

- [`makefiles/file-cmds.mk`](../../makefiles/file-cmds.mk)：版本固定为 430.100.5，按 `chflags compress ipcrm ipcs pax xattr` 逐个编译，编译后调用 `AFTER_BUILD`。
- [`build_patch/file-cmds/stdio-compress.patch`](../../build_patch/file-cmds/stdio-compress.patch)：只修改 `compress`，与 `chflags` 无关。
- `git status` 干净；没有未提交的 `chflags` 源码修改。

因此“源码被改坏”这一假设可以排除。

### 2.2 构建层：`AFTER_BUILD` 会重新绑定符号

[`Makefile`](../../Makefile) 的 `AFTER_BUILD` 对 roothide 目标的每个 Mach-O 执行：

```sh
if [ ! -z "$(findstring roothide,$(MEMO_TARGET))" ]; then
    if [ -z "$(4)" ]; then symredirect $$file || exit 1; fi
fi
```

`symredirect` 的 shim 列表来自 `vroot.h`。本分支的宿主工具由 [`build_tools/prepare_native_tools.sh`](../../build_tools/prepare_native_tools.sh) 从 pinned `libroothide` 源码现场编译；[`makefiles/roothide.mk`](../../makefiles/roothide.mk) 当前 pin 的是 `77af8e9`。该版本的 `vroot.h` 同时包含：

```c
VROOT_API_WRAP(int, chflags,  (const char *path, __uint32_t flags), (newpath,flags), path)
VROOT_API_WRAP(int, lchflags, (const char *path, __uint32_t flags), (newpath,flags), path)
```

`file-cmds` 是源码重建的，不是直接复用官方预编译二进制，所以新编出来的 `chflags` 会被当前 `symredirect` 处理。

这一点与 [`docs/upstream-workflow.md`](../upstream-workflow.md) 中“本分支新增：patched ldid/symredirect”的说明一致：宿主工具从源码生成，原本是为了让 host shim 与 target `libvroot` 同源、构建可复现；副作用是每次更新 `libroothide` 都会自动带上新的 shim 项。

### 2.3 二进制层：关键差异

对两个实际产物做 `nm -m`：

| 二进制 | `_chflags` | `_lchflags` | `_fts_open` |
| --- | --- | --- | --- |
| roothide 官方 `bootstrap-1900.tar.zst` 中的 `usr/bin/chflags` | `libSystem` | `libvrootapi` | `libvrootapi` |
| 本仓库构建的 `file-cmds_430.100.5_iphoneos-arm64e.deb` 中的 `usr/bin/chflags` | **`libvrootapi`** | `libvrootapi` | `libvrootapi` |

复现命令：

```sh
nm -m /usr/bin/chflags | grep -E '_chflags|_lchflags|_fts_open'
```

进一步验证这不是官方二进制自身的问题，而是我们的宿主工具会这样处理：

```sh
cp <official>/usr/bin/chflags /tmp/chflags_ref_test
build_work/host-tools/bin/symredirect /tmp/chflags_ref_test
nm -m /tmp/chflags_ref_test | grep _chflags
```

处理前：

```text
_chflags (from libSystem)
```

处理后：

```text
_chflags (from libvrootapi)
```

说明：**只要用当前 `symredirect` 处理一个 `chflags`，它就会把 `_chflags` 加上 shim**，不管输入来自官方还是我们自己的构建。

## 3. 根因分析

### 3.1 vroot 的翻译规则

`libroothide` 的 `jbroot_alloc()` 大致规则：

```text
"/rootfs/..."  -> 去掉 /rootfs，返回真实 rootfs 绝对路径
"/var/..."     -> <JBROOT>/var/...（jbroot 影子路径）
"relative"     -> 原样返回
```

关键点：函数没有“这个路径已经翻译过”的标记，也没有判断输入是否已经以 `JBROOT` 开头。因此每次调用都会按当前规则再翻一次。

### 3.2 `chflags(1)` 的两次翻译

```text
argv: /rootfs/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/

  vroot_fts_open()
      /rootfs/var/...  ->  /var/...                 （第一次，落到真实 rootfs）
      fts_path = /var/.../AssetsV2/

  vroot_chflags()
      /var/...         ->  <JBROOT>/var/...          （第二次，落到 jbroot 影子）
      jbroot 下没有 MobileSoftwareUpdate/...          -> ENOENT
```

`chflags.c` 里 `fts_open()` 和 `chflags()` 是两个独立调用点，在普通系统上这是正确设计；在 vroot 下，两个调用点都被 shim，就违反了“一个路径只翻译一次”的前提。

### 3.3 官方为什么没有暴露

官方 bootstrap 的 `chflags` 二进制是在 `chflags` wrapper 加入之前构建的（或使用了旧版 `symredirect`），因此只有 `lchflags` 和 `fts_open` 被重定向。值得注意的是，官方 bootstrap 里的 `libvrootapi` 其实已经导出 `_chflags`，只是那个预编译的 `chflags` 没有被重新 `symredirect`，属于“库比二进制新”的版本不同步状态：

- 非递归命令（不带 `-R`/`-h`）使用 `FTS_LOGICAL`，`chflagsat(..., atflag=0)` 走 `chflags()`；官方的 `chflags()` 未 shim，拿到 `fts_open()` 翻好的真实 `/var/...`，成功。
- `-R` 默认 `FTS_PHYSICAL`，`atflag=AT_SYMLINK_NOFOLLOW`，走 `lchflags()`；官方的 `lchflags()` 被 shim，因此 `-R` 的绝对路径根节点仍有二次翻译风险。

也就是说，官方不是“完全正确”，而是“非递归路径碰巧没被 shim”。我们的重建把 `chflags` 也纳入 shim，于是非递归路径同样失败。

### 3.4 历史版本对照

上游 `roothide/libroothide` 的 `vroot.h`：

- 2023-08-31 初始版本：已有 `lchflags` wrapper。
- 2024-01-31 的 `42db289b`：仍只有 `lchflags`。
- 2024-09-15 的 [`5f3f808c8a`](https://github.com/roothide/libroothide/commit/5f3f808c8a)：在 “add support for building on linux” 提交中新增 `chflags` wrapper。

该提交的注释上下文还写着：

```c
//file-cmds/chmod.c->modify_file_acl? but its from apple opensource probject,
//this shim library is just for gnu opensource project
```

这说明 vroot 原本的定位主要面向 GNU 工具，并不是专门为 Apple `file-cmds` 的 `chflags(1)` 设计。`chflags` 的一行改动更像“与 `lchflags` 保持 API 对称”的补全，没有针对 `chflags(1)` 的 `fts_open + chflags` 组合做回归。

## 4. 影响面

### 4.1 符号层扫描结果

扫描方法：在 bootstrap / stage 中查找链接 `@loader_path/.jbroot/usr/lib/libvrootapi.dylib` 的 Mach-O，再用 `nm -m` 过滤 `_chflags`/`_lchflags`。

**tvOS 全量 bootstrap（383 个链接 libvrootapi 的 Mach-O）：**

| 文件 | 被 shim 的相关符号 |
| --- | --- |
| `usr/bin/chflags` | `_chflags`、`_lchflags`、`_fts_open` |
| `usr/bin/compress` | `_chflags` |
| `usr/bin/login` | `_chflags` |
| `usr/sbin/pw` | `_lchflags` |

**iPhone stage（671 个链接 libvrootapi 的文件，抽样）：**

| 文件 | 被 shim 的相关符号 |
| --- | --- |
| `libpython3.9.dylib` | `_chflags`、`_lchflags` |
| `libarchive.13.dylib` | `_lchflags` |
| `libxar.1.dylib` | `_chflags` |

iPhone stage 不是完整安装集；`file-cmds` 本身也未 staged，但对应 Deb 中的 `chflags`/`compress` 已确认带 shim。

### 4.2 行为层分类

| 类别 | 工具 | 判断 |
| --- | --- | --- |
| 确定二次翻译 | `chflags(1)` | `fts_open` + `chflags` 组合，`/rootfs/...` 和绝对 jbroot 路径都会失败 |
| 单次翻译，通常正常 | `compress` | 直接对 argv/输出文件名调用 `chflags(name, ...)`，没有 `fts_open` 预翻译；shim 在这里反而是正确的一层 |
| 单次翻译，通常正常 | `login`、`pw` | 直接路径调用，没有 `fts_open` 预翻译 |
| 单次翻译，通常正常 | `libpython`、`libarchive`、`libxar` | 库函数直接接收调用方路径；只要调用方不传已翻译路径，就是单次 |
| 同类风险，待回归 | `pax` | 本身不 import `_chflags`，本次不受直接影响；但它有 `fts_open` + `lchmod`/`lchown` 的类似组合，属于同一类“路径生产者 + 元数据 shim”风险 |

### 4.3 系统性风险

这个 shim 不是 `file-cmds` 专属的。所有经本仓库 `AFTER_BUILD` 处理、且 import `_chflags`/`_lchflags` 的二进制都会受影响。任何第三方工具只要满足：

```text
fts_open / realpath / glob / getcwd（已被 shim）
        ↓
chflags / lchflags（也被 shim）
```

就会发生同样的二次翻译。换句话说，问题在 **shim 的组合语义**，不在某个包的源码。

### 4.4 最终软件源复核（2026-11-23）

复核对象是 Rachel 服务器上的归档容器 `procursus-repository`（即最终同步的构建容器）：

- 容器内仓库：`/work`，git `8208189be`（与本仓库当前 HEAD 一致）；
- 最终索引：`build_repo/dists/1900/Release`，日期 `2026-11-18`；
- 全部 Deb：2,946 个；其中 iOS pool `2.6G`、1,486 个，tvOS pool `2.1G`、1,460 个；
- 按本次要求只分析 iOS；将 iOS pool 复制到 Rachel 宿主机 `/tmp/chflags-scan-ios/pool`，在副本上用 Mach-O 解析器和 `dpkg-deb -x` 扫描；
- 容器在扫描前后保持 `exited` 状态，未在容器内落盘分析数据；临时副本已清理，扫描结果保留在 Rachel 宿主机 `/tmp/chflags-scan-ios/result.jsonl`。

iOS pool 扫描结果（1,486 个 Deb）：

| 包 | 二进制 | 同时存在的 producer | 结论 |
| --- | --- | --- | --- |
| `file-cmds` | `usr/bin/chflags` | `fts_open` | **确认实际受影响**：`fts_open` 先翻译，`chflags`/`lchflags` 再翻译一次 |
| `file-cmds` | `usr/bin/compress` | 无 | 直接对输出文件名调用 `chflags`，单次翻译，正常 |
| `libarchive` | `libarchive.13.dylib` | 无 | 直接调用 `lchflags`，单次翻译，正常 |
| `mtree-netbsd` | `usr/bin/mtree` | `fts_open`、`getcwd` | **误报**：源码始终 `chdir` 后 `fts_open(".")`，`lchflags(p->fts_accpath)` 收到相对路径，不会二次翻译 |
| `system-cmds` | `usr/bin/login` | 无 | 直接路径调用，正常 |
| `system-cmds` | `usr/sbin/pw` | 无 | 直接路径调用，正常 |
| `python3` | `libpython3.9.dylib` | `getcwd`、`realpath` | **潜在**，不是确定性故障：`os.chflags`/`os.lchflags` 是给调用方的 API；`shutil.copystat` 的 `dst`、`tempfile` 的模板路径都是原样传入，单次翻译。只有用户脚本把 `realpath`/`getcwd` 的结果再交给 `chflags` 才会踩中 |
| `xar` | `libxar.1.dylib` | 无 | 直接调用 `chflags`，单次翻译，正常 |

控制脚本扫描：

- 对 1,486 个 iOS Deb 的 `DEBIAN/` 脚本做了 `chflags` 文本扫描；
- 没有发现任何包在 `postinst`/`preinst` 等控制脚本里直接调用 `chflags`；
- 只有 `file-cmds` 自身的 `md5sums`/`control` 和 `zsh` 的补全函数提到 `chflags`。

因此，**官方软件源中真正被这个双重翻译问题确定性影响的包只有 1 个：`file-cmds`**（具体是 `usr/bin/chflags`）。`mtree-netbsd` 是符号层误报；`python3` 属于依赖调用方写法的潜在风险；其余 4 个包是直接路径调用，不构成二次翻译。

现实中 `com.nan.disableupdates` 之所以中招，是因为它在 postinst 里调用了这个受影响的 `/usr/bin/chflags`，并传入 `/rootfs/...`；它不是源里的包，而是这次问题的触发者。

## 5. 修复建议

### 5.1 推荐：按二进制定向排除

在 `symredirect` 中增加“按安装路径跳过符号”的规则，例如对 `usr/bin/chflags` 跳过 `_chflags` 和 `_lchflags`，保留 `_fts_open`：

```text
usr/bin/chflags:
    skip _chflags
    skip _lchflags
```

理由：

- `chflags(1)` 的入参翻译已经由 `fts_open` 完成；syscall 层不应再翻一次。
- 同时跳过 `lchflags` 也能修复 `-R` 路径；官方只是恰好没有 shim `chflags`，`lchflags` 仍然被 shim，所以官方的 `-R` 未必干净。
- 不影响 `compress`、`libarchive` 等直接调用者继续使用 shim。

实现上建议使用可维护的排除表（例如 `build_misc/roothide/symredirect-skip.txt`），而不是在代码里硬编码；`symredirect` 处理每个文件时读取 `basename` 或安装路径匹配。

### 5.2 备选：让路径翻译幂等

在 `libroothide` 层让 `jbroot_alloc()` 识别“已经翻译过的路径”，或为 vroot 调用链增加“仅翻译一次”的上下文标记。这个方案更根本，但影响所有 vroot API，风险和验证成本更高，不建议作为首个修复。

### 5.3 不建议

- **不要全局删除 `vroot.h` 的 `chflags`/`lchflags`。** `compress`、`libarchive` 等直接调用者依赖这层翻译；全局删除会把它们的目标从 jbroot 改到真实 rootfs。
- **不要简单回退到旧 `symredirect` 作为长期方案。** 那只是让二进制重新落到旧版本的行为，后续重建仍会遇到版本漂移。
- **不要只给 `com.nan.disableupdates` 的 postinst 加 `|| true`。** 那会掩盖设置失败，OTA 目录实际上没有被保护。

### 5.4 设备上的临时规避

在修复版 bootstrap 发布前，可用相对路径绕过二次翻译（`jbroot_alloc()` 对相对路径原样返回）：

```sh
cd /rootfs/var/MobileSoftwareUpdate/MobileAsset
chflags schg,schange,simmutable AssetsV2
chflags -R noschg,noschange,nosimmutable .
```

注意：`cd` 本身会经过 `chdir` shim，把 `/rootfs/...` 映射到真实 `/var/...`；随后相对路径不再被加 jbroot 前缀。

### 5.5 建议决策

| 优先级 | 建议 |
| --- | --- |
| P0 | 如果 OTA 禁用是本次 Bootstrap 的发布要求，先不要把这个 `chflags` 构建作为默认版本发布；至少用 5.4 的相对路径方案验证目标 flags 确实生效 |
| P1 | 在 `symredirect` 增加 `usr/bin/chflags` 的符号排除表，重新构建 `file-cmds` 并做设备回归 |
| P1 | 把 `chflags` 的非递归和 `-R` 用例加入 Bootstrap 发布前检查 |
| P2 | 评估 vroot 的幂等化，作为上游长期修复方向 |

### 5.6 发布影响：必须重打 Bootstrap

`file-cmds` 确实存在于 Bootstrap 中，不是只存在于 APT pool：

- `build_info/bootstrap/iphoneos-arm64e.txt` 和 `appletvos-arm64e.txt` 第 19 行都固定了 `file-cmds==430.100.5`；
- 最终 iOS Bootstrap `bootstrap-iphoneos-arm64e.tar.zst` 中包含 `./usr/bin/chflags` 和 `./Library/dpkg/info/file-cmds.*`；
- Bootstrap 内的 `chflags` SHA-256 为 `0e960a8bc92765433e377ed2c225edc8589dd82d89e28d58fbffbb3818d4414f`，与最终 `file-cmds_430.100.5` Deb 内的二进制完全一致；
- 该二进制仍然是 `_chflags`/`_lchflags`/`_fts_open` 全部来自 `libvrootapi` 的版本。

因此修复发布分两部分：

1. **APT 包更新**：让已安装用户通过软件源升级 `file-cmds`，替换 `/usr/bin/chflags`；
2. **Bootstrap 重发布**：Bootstrap 是单一签名 tar.zst，不能只替换其中一个成员；必须重新组装、重新签名并上传 iOS 和 tvOS 两个 Bootstrap。

只重编 `file-cmds` 即可，不需要重编全仓。但要注意版本衔接：

- 建议提升 `file-cmds` 的 Debian 版本（例如 `430.100.5-1` 或带项目后缀的版本），同步修改 `build_info/bootstrap/*.txt` 的 `file-cmds==...` 固定值；
- 如果保持同一版本重新打包，`publish_repo.py` 默认会拒绝“同名不同内容”，Bootstrap 组装也可能因同版本内容冲突失败，需要显式 `--replace`，不利于客户端判断；
- 重新组装时使用 `make bootstrap BOOTSTRAP_PACKAGES_DIR=...`，两个平台分别执行，并用同一签名身份重签 `.asc`。

## 6. 验证方案

### 6.1 构建产物

修复后重新打包 `file-cmds`，在宿主上检查：

```sh
nm -m <stage>/usr/bin/chflags | grep -E '_chflags|_lchflags|_fts_open'
```

期望：

```text
_chflags  (from libSystem)
_lchflags (from libSystem)
_fts_open (from libvrootapi)
```

`_chflags`/`_lchflags` 来自 `libSystem` 表示不再二次翻译；`_fts_open` 仍来自 `libvrootapi` 表示入参翻译保留。

### 6.2 设备回归

| 用例 | 期望 |
| --- | --- |
| `chflags schg,schange,simmutable /rootfs/var/MobileSoftwareUpdate/MobileAsset/AssetsV2/` | 退出码 0；`ls -lO` 显示 `schg` |
| `chflags -R noschg,noschange,nosimmutable /rootfs/var/MobileSoftwareUpdate/MobileAsset/` | 退出码 0；子项 flags 清除 |
| `chflags schg /rootfs/tmp/foo`（临时文件） | 成功，证明 `/rootfs` 前缀路径可用 |
| `chflags schg /usr/share/...`（jbroot 路径） | 成功，证明 jbroot 语义未被破坏 |
| `compress`、`xattr`、`pax` 基本操作 | 无回归 |

### 6.3 `symredirect` 侧回归

建议加一个最小 fixture 测试：准备一个 import `_chflags`、`_lchflags`、`_fts_open` 的 Mach-O，跑 `symredirect` 后断言 `_chflags`/`_lchflags` 的绑定未被改写、`_fts_open` 被改写；再准备一个 `compress` 风格的 fixture，断言 `_chflags` 仍被改写。这样可以把“按二进制定向排除”的语义固定下来。

## 7. 未决问题

1. **截图第三条报错路径的精确来源。** 按现有两个构建的绑定，非递归命令的错误信息应打印 `fts_path`（即 `/var/...`）；截图中出现 `.jbroot-...` 的路径，说明设备上的 `chflags` 可能不是当前 Deb 的同一份，或还有一层我们尚未定位的翻译。建议在设备上执行 `nm -m /usr/bin/chflags` 和 `otool -L /usr/bin/chflags` 确认实际二进制。
2. **官方 `-R` 的根节点行为。** 官方 `lchflags` 被 shim，按代码路径推断 `-R` 在绝对路径根节点上也会二次翻译；本次未在设备上执行官方 bootstrap 验证。
3. **iPhone 完整包集的扫描范围。** 目前扫描的是 tvOS 全量 bootstrap 和 iPhone stage 抽样；iPhone 正式 bootstrap 的包集合可能更大，建议在修复前对最终 Bootstrap 做一次完整符号扫描。
4. **`chflags` 本身是否需要额外权限。** 本次错误是 `ENOENT` 而非 `EPERM`，指向路径翻译而非权限；修复路径后仍需在真实设备上确认 `schg` 能否成功设置。

## 8. 附录

### 8.1 关键文件

- [`makefiles/file-cmds.mk`](../../makefiles/file-cmds.mk)
- [`Makefile`](../../Makefile)（`AFTER_BUILD` / `symredirect`）
- [`makefiles/roothide.mk`](../../makefiles/roothide.mk)
- [`build_tools/prepare_native_tools.sh`](../../build_tools/prepare_native_tools.sh)
- [`build_patch/file-cmds/stdio-compress.patch`](../../build_patch/file-cmds/stdio-compress.patch)
- [`docs/upstream-workflow.md`](../upstream-workflow.md)

### 8.2 外部参照

- Apple `file_cmds` 400 / 430.100.5 `chflags.c`：<https://github.com/apple-oss-distributions/file_cmds>
- roothide `libroothide` 的 `chflags` wrapper 提交：<https://github.com/roothide/libroothide/commit/5f3f808c8a>
- roothide 官方 bootstrap 产物：<https://github.com/roothide/Bootstrap/blob/main/strapfiles/bootstrap-1900.tar.zst>
- roothide 路径规则说明：<https://github.com/roothide/Developer/blob/main/roothide.md>

### 8.3 检查命令摘要

```sh
# 源码对照
diff -u <file_cmds-400>/chflags/chflags.c <file_cmds-430.100.5>/chflags/chflags.c

# 绑定对照
nm -m <official>/usr/bin/chflags | grep -E '_chflags|_lchflags|_fts_open'
nm -m <ours>/usr/bin/chflags     | grep -E '_chflags|_lchflags|_fts_open'

# 证明宿主 symredirect 会加上 _chflags shim
cp <official>/usr/bin/chflags /tmp/chflags_ref_test
build_work/host-tools/bin/symredirect /tmp/chflags_ref_test
nm -m /tmp/chflags_ref_test | grep _chflags
```

### 8.4 当前判断

| 问题 | 判断 |
| --- | --- |
| 是否 `chflags.c` 被改坏 | 否；源码与 Apple 430.100.5 一致 |
| 是否我们的构建产物与官方不同 | 是；`_chflags` 被 shim 到 `libvrootapi` |
| 是否 vroot 二次翻译 | 是；`fts_open` + `chflags` 两个调用点各翻译一次 |
| 是否只影响 `chflags(1)` | 符号层影响多个工具；行为层 `chflags(1)` 确定性失败 |
| 修复位置 | `symredirect` 按二进制定向排除，或 vroot 幂等化 |
| 当前风险等级 | 对 OTA 禁用包为高（目标 flags 未设置）；对系统整体为局部功能风险 |

## 9. 构建与 vphone 验证（2026-11-23）

### 9.1 构建环境

本机 Mac 的构建目录已清理，且本机 Ubuntu 容器 `stripe-local-publish` 只有 `apt-get`，不具备编译工具；实际构建在 Rachel 的 OrbStack iOS 构建容器 `procursus-ios-full` 中完成（Linux 宿主、内置 `iPhoneOS16.5.sdk` 和 host tools）。

构建入口：

```sh
. build_work/host-tools/host-env.sh
build_tools/prepare_native_tools.sh
make rebuild-file-cmds-package
```

Linux 下 `prepare_native_tools.sh` 需要显式定义 `__LITTLE_ENDIAN__` 才能编译 cctools 的 foreign Mach-O 头；该定义已加入配方，后续不需要手工传环境变量。

### 9.2 产物形态

生成的 Deb：

```text
build_dist/iphoneos-arm64-roothide/1900/file-cmds/file-cmds_430.100.5_iphoneos-arm64e.deb
```

Deb 内 `usr/bin/chflags`：

```text
_chflags   (from libSystem)
_lchflags  (from libSystem)
_fts_open  (from libvrootapi)
```

同一包内 `usr/bin/compress` 仍保持：

```text
_chflags   (from libvrootapi)
```

说明按二进制排除没有误伤 `compress`。Deb SHA-256 为 `d0048aa0efabd626bfb1e60f7c4c0c6b4e763a1e69214178f1e78bebd5bc38b9`，其中 `chflags` 的 SHA-256 为 `b25e86ce427c82a0efe017505d82a62cc5fc39ce0b81606bcf3cfc8b7c01b92d`。

### 9.3 vphone 测试

测试设备：vphone `Fig`（iPhone17,3，iOS 26.5），使用正式构建的 `chflags` 二进制部署到 `/usr/bin/chflags.fixed` 执行；测试后已删除，原 `/usr/bin/chflags` 未替换。

| 用例 | 结果 |
| --- | --- |
| 原 `/usr/bin/chflags hidden /rootfs/var/tmp/cf-test` | 失败：`chflags: /var/tmp/cf-test: No such file or directory`，复现二次翻译 |
| 修复版 `chflags.fixed -v hidden /rootfs/var/tmp/cf-test` | 成功，退出码 0，`-v` 打印 `/var/tmp/cf-test` |
| 重复 `chflags.fixed -R -v hidden` | 第二次没有输出，说明第一次设置的 flags 已持久化 |
| `chflags.fixed -v 0 /rootfs/var/tmp/cf-test` | 成功清除 |
| `chflags.fixed -R -v hidden /rootfs/var/tmp/cf-tree` | 文件、子目录、根目录均成功 |
| 原 `/usr/bin/chflags -R hidden /rootfs/var/tmp/cf-tree` | 失败：`chflags: /var/tmp/cf-tree: No such file or directory` |
| `chflags.fixed -v hidden /tmp/cf-test2`（jbroot 路径） | 成功，路径显示为 `<jbroot>/tmp/cf-test2` |

结论：修复版在 `/rootfs` 路径和 jbroot 路径上都能正确执行，非递归与 `-R` 均通过；原版仍可按预期复现故障。

### 9.4 发布前仍需完成

验证构建使用 `430.100.5`；代码提交已将 Debian 版本提升为 `430.100.5-1`，并同步两个 Bootstrap 清单。正式发布前还需要：

1. 提升 `file-cmds` 版本并更新两个 `build_info/bootstrap/*.txt` 的固定值；
2. 重新打包 iOS/tvOS 两个目标的 Deb；
3. 更新 APT pool/index，让已安装用户能升级；
4. 重新组装并重签 iOS/tvOS 两个 Bootstrap。
