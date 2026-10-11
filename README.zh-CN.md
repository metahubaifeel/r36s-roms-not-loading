# R36S：游戏已经放进卡里了，为什么前端里看不到、或者点开就退出？

**[English](README.md)** · **中文**

> **一句话结论。** 在跑 ArkOS / dArkOS 的 R36S 上，游戏**不在** Windows 顺手给你的那个小 FAT32 分区里，
> 也**不在**卡的根目录。它们在卷标为 `EASYROMS` 的那个大 exFAT 分区里，路径是 `roms/<平台>/`，
> 而 `<平台>` 必须是前端配置里写死的那串小写目录名（`gba`、`nes`、`snes`、`psx`、`megadrive`……）。
> 这两件事任一做错，前端的行为就等同于"这个文件根本没被拷进来"——
> 一个**没有任何被识别游戏的平台会被整个隐藏掉**，于是菜单里连入口都没有，自然也谈不上报错。

这篇是那句话的完整版：真实分区结构、真实目录名、真正会让"已经拷对位置的 ROM"启动不了的五个原因，
以及每一条怎么自己验证。下面每条命令写文时都实际跑过，原始输出就在截图里。

---

## 一、群里到底在问什么

以下原文来自中文掌机 QQ 群，**只标群名**——原始数据无法可靠地把某条消息归属到某个具体发言人。

> `[开源掌机中文社区⑤群]` 问个小问题，为什么我下载的一些hack游戏rom，放掌机里没法玩呢，换光了核心也不行

> `[r36s（开源掌机）交流1群]` 意思是叫他来这里下载游戏放到掌机中玩，我也是刚到手，只知道有这个，但还不懂怎样把游戏放到掌机中

> `[安伯尼克掌机交流群]` @土豆 有没有视频教程解压后怎么放到内存卡根目录。

> `[R36S掌机爱好者]` 我发现掌机只能识别fat32文件系统

> `[r36s（开源掌机）交流1群]` 如果下了一个64g的游戏包后面还能一个一个往里面添加自己喜欢的游戏吗

> `[r36s（开源掌机）交流1群]` @旋转企鹅Z   资源群公告和群文件里全都有。懒得自己翻看就别张口发问，无脑伸手直接踢。这里歧视弱智

从这批数据本身能得出两个结论：这是掌机群里被重复问得最多的一类问题；而群管理自己给出的答案是
**"路径在这儿"**，不是"链接在这儿"。这篇就按这个思路写。

**先提醒一件事：群里流传的"标准答案"有一部分是错配的。** 掌机群覆盖几十种机器，
被转得最多的那几条里混着**安卓掌机**的流程，跟 R36S 不通用：

> `[安伯尼克掌机交流群]` 电脑下整合包，注意分卷用 WinRAR/7-Zip 解 part01 ，别多层嵌套。解出来应是 ROMs/gba 、 ROMs/gb …
> 掌机连电脑选"文件传输"，电脑会出 KPA 设备盘。… 装模拟器 APK 并首次全开给权限：RetroArch… PPSSPP（PSP）DraStic（NDS）

R36S **不是安卓**。没有 KPA 设备盘，没有 APK 要装，也没有"首次全开权限"这个界面——
它上面所有模拟器都是系统镜像预装好的。把安卓那套流程照搬到 R36S，
正是"卡看着满了、机器里还是空的"的常见成因之一。

> ⚠️ **未实测**：我们手上没有安卓掌机、也没有 KPA 设备，因此**不评价**那套流程在那边是否可用。
> 我们只主张一件事：它描述的不是 R36S。

---

## 二、为什么会这样

### 2.1 卡上有三个分区，前端只扫其中一个

![R36S 系统卡的三个分区，以及各自装什么](docs/01-card-layout.png)

下面这张分区表是 R36S 的 dArkOS 镜像自己文档里给出的：MBR 分区表，三个分区。
我们用 `fdisk` 在一个测试文件上把它复现出来，好看清真实的分区类型码：

| 分区 | 起始扇区 | 镜像内大小 | 类型码 | 装什么 |
|---|---|---|---|---|
| p1 | 32768 | 100 MiB | `0c` W95 FAT32 (LBA) | `KERNEL`、`.dtb` 文件、`boot.ini`、机型选择脚本 |
| p2 | 237568 | 6.7 GiB | `83` Linux (ext4) | Linux 根文件系统 |
| p3 | 14368768 | 100 MiB 占位 | `07` HPFS/NTFS/exFAT | **首次开机后长成整卡并转成 exFAT，就是 `EASYROMS`** |

也就是说，Windows 能读的只有两个：小的 `BOOT` 和大的 `EASYROMS`。
`BOOT` 是插上就读出来、看起来"就是这张卡"的那个，也恰恰是绝对不能放游戏的那个——
前端只扫 `EASYROMS/roms`。把 ROM 拷进 `BOOT`，你会得到一张"明显占用了空间"的卡，
和一台"什么都没多出来"的机器。

`p2` 是 ext4。Windows 完全读不了 ext4——这就是它偶尔会劝你重新格式化一张完全健康的卡的原因。
那个弹窗怎么处理，见同系列的另一篇：
[`r36s-sd-card-readonly-recovery`](https://github.com/metahubaifeel/r36s-sd-card-readonly-recovery)。

> ⚠️ **未实测**：我们没有 dump 过商家原装卡的实测分区表。上面的结构来自 R36S 的 dArkOS 项目
> 对自己镜像的文档。不同商家刷的卡在**尺寸**上可能不同；真正要紧的是**结构**——
> 小 FAT32 引导分区 + Linux 分区 + 大 exFAT 游戏分区。

### 2.2 目录名本身就是接口

ArkOS 把目录结构直接刷在卡上，而且你可以从项目自己的仓库里把权威清单读出来，
不必相信任何一篇论坛帖：

![实机终端输出：ArkOS 自带的 122 个默认游戏目录](docs/04-folder-list-verified.png)

122 个一级目录。最常用的那些，以及 R36S 这一档机器把它们映射到哪个核心、哪个 BIOS：

![ROM 文件必须落到的路径，以及常见平台对应的目录名](docs/02-folder-names.png)

| 平台 | 目录 | Linux 路径 | 核心 | BIOS 文件 |
|---|---|---|---|---|
| Game Boy Advance | `gba` | `/roms/gba` | mGBA | `gba_bios.bin` |
| Game Boy / Color | `gb`、`gbc` | `/roms/gb`、`/roms/gbc` | Gambatte | `gb_bios.bin` |
| NES / Famicom | `nes` | `/roms/nes` | Nestopia | — |
| SNES / SFC | `snes` | `/roms/snes` | Snes9x | — |
| MD / Genesis | `megadrive`、`genesis` | `/roms/megadrive` | Genesis Plus GX | — |
| Master System / GG | `mastersystem`、`gamegear` | `/roms/mastersystem` | Genesis Plus GX | `bios_E.sms` |
| PlayStation | `psx` | `/roms/psx` | PCSX-ReARMed | `psxonpsp660.bin` |
| 街机 | `arcade` | `/roms/arcade` | FinalBurn Neo | — |
| Neo Geo | `neogeo` | `/roms/neogeo` | FinalBurn Neo | `neogeo.zip` |
| Dreamcast | `dreamcast` | `/roms/dreamcast` | Flycast | `dc/dc_boot.bin` |
| Nintendo 64 | `n64` | `/roms/n64` | 独立模拟器 | — |
| Nintendo DS | `nds` | `/roms/nds` | 独立模拟器 | — |
| PSP | `psp` | `/roms/psp` | 独立模拟器 | — |
| PC 移植 | `ports` | `/roms/ports` | PortMaster | — |

因为底层是 Linux 而不是 Windows，有两条后果：

- **大小写敏感。** 配置指向 `/roms/snes`。一个叫 `SNES` 或 `Snes` 的目录，是另一个**并不存在**的路径。
- **没有"扫全卡"模式。** 文件不是靠内容识别的：它既要在配置的路径下，扩展名还要在该平台允许的扩展名列表里，
  两者缺一，它就不是"游戏"。

### 2.3 一个"没有游戏的平台"会被整个隐藏

这一点是整件事最让人困惑的地方。与其猜，不如直接读前端源码。
ArkOS 用的那个 EmulationStation 分支里写得很清楚：

```cpp
// es-app/src/SystemData.cpp
bool SystemData::isVisible()
{
    if ((getDisplayedGameCount() > 0 || ... ))
    {
        ... return std::find(hiddenSystems.cbegin(), hiddenSystems.cend(), getName()) == hiddenSystems.cend();
    }
    return false;                          // ← 没有游戏 ⇒ 这个平台整个不显示
}
```

而当配置的路径不是目录时，扫描直接放弃：

```cpp
void SystemData::populateFolder(FolderData* folder, ...)
{
    const std::string& folderPath = folder->getPath();
    if(!Utils::FileSystem::isDirectory(folderPath))
    {
        LOG(LogWarning) << "Error - folder with path \"" << folderPath << "\" is not a directory!";
        return;
    }
    ...
            if(!isGame && Utils::FileSystem::isDirectory(filePath))
            {
                ...
                populateFolder(newFolder, fileMap);   // ← 它确实会递归进子目录
            }
}
```

两段代码合起来读，所有症状就都能解释：

1. **目录名不对** → 路径不存在 → **零个游戏 → 平台被隐藏**。你的 GBA 游戏不是"打不开"，
   而是菜单里压根没有 GBA 这个入口。
2. **扩展名不对** → 这个文件不被算作游戏 → 同样的结果。
3. **多套一层目录，通常是无害的**，因为扫描会递归。这一点上群里流传最广的那句
   （*"别再套一层 Room/ROMs/ROMs"*）在技术上是**错的**，它会把人带去找一个根本不存在的嵌套问题。
   真正会坏的是**把平台目录本身挤走了**：如果你的解压结果是 `ROMs/gba/...`，
   而你把整个 `ROMs` 目录拷进了卡上的 `roms` 里，你现在得到的是 `roms/ROMs/gba/...`，
   而 `/roms/gba` 没了。递归也好不递归也好，这个路径就是不存在。
4. **点开头的隐藏文件会被跳过**（`if(!showHidden && fileInfo.hidden) continue;`）。
   这正是 ArkOS 文档会专门提醒"从 Mac 拷 ROM 可能不显示"的原因——
   macOS 会在真文件旁边写 `._game.gba` 这类 AppleDouble 伴生文件，而名字以点开头的文件前端直接忽略。

### 2.4 五个真正会让游戏启动不了的原因，按值得检查的顺序

![已经拷过文件、游戏仍然启动不了的五个有据可查的原因](docs/03-five-failures.png)

| # | 原因 | 你会看到什么 | 怎么修 |
|---|---|---|---|
| 1 | 文件拷进了 `BOOT` 而不是 `EASYROMS` | 卡看着满了，游戏库没变 | 拷到 `EASYROMS:\roms\<平台>\` |
| 2 | 还是压缩包（`.rar`、分卷 `.7z` 等） | 列表里有，点开秒退，换哪个核心都一样 | 先用 7-Zip 解压；街机合集保持打包 |
| 3 | 目录名或大小写不对（`GBA`、`Snes`） | 整个平台从菜单里消失 | 用完全一致的小写名 |
| 4 | 插了第二张卡但没在机器上切换过去 | 一切仍然读内置的 `/roms` | `设置 > 高级设置 > Switch to SD2 for Roms`，重启 |
| 5 | 缺必需的 BIOS 文件 | PS1 / Dreamcast / Neo Geo 启动即退 | 把 BIOS 放进 `/roms/bios`，文件名要完全一致 |

关于第 2 条，说得再准一点：ArkOS 自带的平台定义里，很多平台**确实**把 `.zip` 和 `.7z`
列为合法扩展名，但**没有任何一个平台列 `.rar`**；而某个核心究竟能不能打开某个压缩包，
又取决于核心和打包方式。所以"我把核心换光了也不行"这句话之所以走不通，
是因为问题在容器上，不在核心上。**注意有一个刻意的例外：FinalBurn Neo / MAME 的街机合集
本来就是以 zip 形式分发的，它们必须保持打包状态。**

> ⚠️ **部分实测**：上面的扩展名列表读自 ArkOS 仓库里公开的 `es_systems.cfg`
> （2020 年的 RG351 时期文件，78 个平台）。我们没有从 R36S 卡里把 `es_systems.cfg` 抽出来，
> 所以请把它理解成"ArkOS 是这么造列表的"，而不是"你卡上就是这些字节"。

### 2.5 卡该用什么格式——顺便纠正群里那个说法

日志里有一条值得纠正的说法：

> `[R36S掌机爱好者]` 我发现掌机只能识别fat32文件系统

这一档机器的镜像，有据可查的事实是：

- 引导分区是 **FAT32**（`0c` W95 FAT32 LBA）。这一半是对的。
- 游戏分区 `EASYROMS` 是 **exFAT**——ArkOS 自己的文档写明：镜像首次开机时会把这个占位分区扩容并转换成
  exFAT，并且让你去找一个卷标就叫 `EASYROMS` 的盘符。
- 用来放游戏的**第二张**卡可以是 `exFAT`、`EXT4`、`FAT`、`FAT32` **或** `NTFS`；
  ArkOS 文档说 EXT4/exFAT/NTFS 性能更好，并警告：如果用 EXT4，第二张卡里的内容属主必须是
  用户/组 `1002:1002`，否则某些平台的游戏会启动失败。

所以这台机器**不是只认 FAT32**。这个印象通常来自：卡被格式化成了内核根本不挂载的文件系统，
于是它在电脑上看得见、在掌机上永远看不见。

> ⚠️ **未实测**：我们没有在真机上把第二卡槽的每种文件系统都试一遍。上面这份清单是引用 ArkOS 官方 FAQ。

另外有一条真实限制，在你决定给大卡选 FAT32 之前应该知道：
**FAT32 单个文件不能超过 4 GiB。** PS1、Dreamcast、PSP 和 DVD 类镜像很容易超过。
exFAT 没有这个限制。这是文件系统本身的属性，跟 R36S 无关。

---

## 三、怎么修

按顺序做，游戏一出现就停。

**第 0 步：先分清你的两张卡。**
彻底关机。R36S 的系统卡和游戏卡在不同卡槽；把系统卡用读卡器插到电脑上。

**第 1 步：找到对的分区。**
打开这张卡。你应该看到一个小盘（`BOOT`，约 100 MB），以及在 Windows 10 1703 或更新版本上，
看到第二个大得多的、卷标为 `EASYROMS` 的盘。**不要在 `BOOT` 里操作。**
如果 `EASYROMS` 没有盘符，用磁盘管理分配一个：

```
Win + X  →  磁盘管理
右键卷标为 EASYROMS 的分区  →  更改驱动器号和路径…  →  添加  →  确定
```

这就是 ArkOS 和 dArkOS 两边都记录的"roms 分区看不见"的修法。

**第 2 步：拷到正确的层级。**
`EASYROMS` 里本来就应该有一个 `roms` 目录。打开它，按平台对上号：

```
EASYROMS:\roms\gba\      ← GBA 的 ROM 放这里
EASYROMS:\roms\snes\     ← SNES 的 ROM 放这里
```

**不要**自己建 `EASYROMS:\roms\ROMs\gba`。**不要**把文件直接丢在 `EASYROMS:\` 根下。
**不要**删掉里面已经存在的目录——尤其 NDS 和 PSP 那两个目录里本来就有模拟器需要的文件，
在你放进第一个游戏之前它们就必须在。

**第 3 步：先解压再拷。**
下到的合集往往是 `.rar`、`.7z` 或者分卷（`.part01.rar`、`.z01` + `.zip`）。
用 7-Zip 从**第一卷**开始解，然后把解出来的 ROM 文件拷进去。
需要的 BIOS 放进 `EASYROMS:\roms\bios\`，文件名要和文档里写的完全一致
（`gba_bios.bin`、`psxonpsp660.bin`、`neogeo.zip` 等）。

**第 4 步：如果用了第二张卡，要在机器上切过去。**
安全弹出、插卡、开机，然后：

```
START（ES 菜单）  →  设置（Options）  →  高级设置（Advanced）  →  Switch to SD2 for Roms
```

关机，把游戏挪到第二张卡上（推荐 exFAT，卷标取个唯一的，比如 `ROMS2`，**不要**叫 `EASYROMS`），
再开机。两个有据可查的坑：一旦切成双卡模式，**两张卡都必须插着**机器才能启动；
而且从此以后 BIOS 文件应该放在第二张卡上。

![实机：R36S 上的 ArkOS / EmulationStation 主菜单](docs/05-real-r36s-arkos-menu.png)

*实机照片：R36S 上的中文主菜单。第 4 步要走的「高级设置」就在这里，底部的版本串同时给出了 ArkOS 版本
和 EmulationStation 版本——报问题时把它一起贴出来最有用。*

**第 5 步：如果 Windows 两个分区都看不见，就别再用读卡器折腾了。**
ArkOS 记录了两种完全不碰读卡器的加游戏方式：
`设置 → Enable Remote Services` 会打开 SSH 和 Samba，你可以直接把文件丢进 `/roms/...`；
系统内建的 FileBrowser（ArkOS Browser）可以用浏览器管理 roms 目录。

---

## 四、怎么确认修好了

1. **在机器里数平台。** 开机动到平台列表。**缺席**的平台 = 路径/扩展名问题
   （`/roms/<名字>` 不存在，或者那个目录里没有文件命中扩展名列表）。
   **在列表里、但点开游戏秒退**的平台 = 压缩包/BIOS/核心问题。
2. **看分区卷标，不要看盘符。** 在 Linux 上插好卡：

   ```bash
   lsblk -o NAME,SIZE,FSTYPE,LABEL
   ```

   正常应该能看到一个约 100 MB 的 `vfat`、卷标 `BOOT`；一个 `ext4` 分区（没有用户卷标）；
   以及一个大的 `exfat`、卷标 `EASYROMS`。如果那个大分区的卷标不是 `EASYROMS`，
   那你手上这张就不是系统卡。Windows 上「磁盘管理」的"卷标"列看的是同一件事。
3. **确认你刚拷的文件真的在你以为的地方。** 按类型排序，看到 `roms\psx\` 里躺着一个 `.rar`，
   是抓第 2 类错误最快的办法。
4. **确认"消失的平台"确实是空的。** 往那个目录里放一个已知可用、已解压、命名正确的 ROM，
   重启，看平台是否回来。因为空平台会被隐藏，一个正确的文件就能把整个入口带回来。

---

## 五、怎么别再踩

- **两张卡，两个职责。** 系统卡别动，游戏放第二张卡。这样游戏卡就是一张普通数据卡，
  可以随便格式化重建而不动系统，整卡备份也轻松得多。
- **一个平台一个目录，不要一个合集一个目录。** 解压出来的合集经常是 `合集名/gba/...`。
  要把合集里各平台目录的**内容**拷进卡上对应的目录，而不是把合集根目录整个搬进去。
- **永远不要让 Windows "修好"一张它读不了的卡。** 如果它要格式化你 R36S 卡上的某个分区，取消。
  它读不了的那个分区是 Linux 分区，格式化它毁掉的是整套系统，不只是几个游戏。
- **一开始就买好卡。** ArkOS 自己的说明：最小 8 GB，强烈建议 16 GB 以上，
  并且不要用低质量或杂牌卡——它们很快就会坏，还会造成模拟性能不稳定。
- **街机合集保持打包，其它一律解压。** 如果非要用 Mac，导出前先把 `._*` 伴生文件清掉。
- **加完游戏预期要重启。** 游戏库是在开机时扫描的，没有一个能跳过这步的可靠"刷新"。

### 游戏从哪来

我们不放 ROM 站点的链接，也不会假装"直接下那个 128 GB 合集"是个中立的建议：
它正是"卡里塞满文件、一个都启动不了"的最常见成因。合规的简短版本是：
自己卡带/光盘的自制 dump、自制游戏与公有领域作品，以及通过 PortMaster 直接在设备上安装的
大量开源移植。无论哪条路，这篇讲的机制都成立。

---

## 六、来源

上面所有事实性内容都出自下面之一。凡是我们自己跑过命令的，截图里就是原始输出。

| 主张 | 来源 |
|---|---|
| `roms/` 下 122 个默认目录名 | [christianhaitian/arkos](https://github.com/christianhaitian/arkos) 仓库里的 `defaultromsfolderstructure.tar`，2026-10-11 用 `tar tf` 实跑核对 |
| R36S 这一档机器各平台的目录名、路径、核心、BIOS 文件 | [dArkOSRE-R36 wiki](https://github.com/southoz/dArkOSRE-R36/wiki)（各平台分页） |
| R36S dArkOS 镜像的真实分区表与尺寸 | [dArkOSen-R36S wiki → Installation](https://github.com/djparentx/dArkOSen-R36S/wiki/Installation) |
| `EASYROMS` 是 exFAT 游戏分区；Windows 可能需要手动分配盘符；不要删预置目录；用 SSH/Samba/FileBrowser 传文件；macOS 的 `._` 文件会导致不显示；第二卡槽支持的文件系统与 EXT4 的 `1002:1002` 警告 | [ArkOS wiki → Home](https://github.com/christianhaitian/arkos/wiki/Home) 与 [ArkOS FAQ](https://github.com/christianhaitian/arkos/wiki/Frequently-Asked-Questions---RG353V) |
| 第二张卡卷标建议、以及 `设置 → 高级设置 → Switch to SD2 for Roms` 路径 | [dArkOSRE-R36 wiki → Firmware Installation](https://github.com/southoz/dArkOSRE-R36/wiki/Firmware-Installation) |
| 目录递归扫描、空平台被隐藏、点开头文件被跳过、扩展名匹配 | [EmulationStation-fcamod 的 `SystemData.cpp`](https://github.com/christianhaitian/EmulationStation-fcamod)（ArkOS 用的前端分支），2026-10-11 阅读 |
| 扩展名列表里有 `.zip`/`.7z`、没有 `.rar` | ArkOS 仓库里的 `es_systems.cfg`（2020 年版，78 个平台） |
| "要格式化磁盘"弹窗指的是 Linux 分区；`TF1-OS` / `TF2-GAME` 卡槽命名 | [r36s.org → How to Add Games to R36S](https://r36s.org/articles/tutorial-add-games)（第三方指南） |
| 机器跑基于 Ubuntu 的 ArkOS，开启远程服务后 SSH 用户为 `ark/ark` | [dov/r36s-programming](https://github.com/dov/r36s-programming) |
| 第一节引用的 QQ 原文 | 对 9 个中文掌机 QQ 群、22,869 条去重消息的内部分析 |

**QQ 数据的局限，直说：** 拿不到可靠的逐条时间戳；无法把某条消息可靠归属到某个具名发言人；
11 个掌机相关群里有 2 个只抓到群名没抓到正文；样本被一个大群主导。
本仓库里出现的任何计数都是**消息条数**，不是单位时间频次。

## 相关

- 姊妹篇：[R36S 内存卡提示"需要格式化"——先别格，先救](https://github.com/metahubaifeel/r36s-sd-card-readonly-recovery)
- r36s.space —— [上手第一步与安全设置](https://r36s.space/guide)
- r36s.space —— [R36S 常见问题：充电、游戏文件、主板版本](https://r36s.space/faq)
- r36s.space —— [裸机 R36S：主板版本与配色](https://r36s.space/products/r36s)
- r36s.space —— [买裸机 R36S 之前：一份清单](https://r36s.space/blog/bare-r36s-checklist)

## 许可

MIT —— 见 [LICENSE](LICENSE)。`docs/` 里的截图包含我们自己硬件的实拍照片，可自由转载，注明出处即可。
