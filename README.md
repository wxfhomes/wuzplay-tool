# Wuzplay NFC 工具 (Web Bluetooth)

一个独立的网页工具，参考 [Joysfusion NFC Updater] 和 [Mochi] 为 Wuzplay 设备打造：
通过 **Web Bluetooth** 连接设备，管理 NFC 标签 `.bin` 文件，并升级固件。

设备跑的是开源的 **pixl.js / allmiibo** 固件，所以协议是公开的。

## 文件

| 文件 | 说明 |
|------|------|
| `index.html` | 主程序：品牌界面 + 蓝牙协议层 + 文件管理 + **内置库** + 固件升级 |
| `library.json` | 内置 amiibo 库清单（由 `build-library.py` 生成，勿手改） |
| `library/` | 内置标签库的 `.bin` 数据（由 `build-library.py` 生成） |
| `firmware/` | OTA 固件包 `.zip` + `firmware.json` 清单（由 `build-firmware.py` 生成） |
| `secure-dfu.js` | Nordic Secure DFU 库（固件升级用），**MIT 许可**，来自 Mochi |
| `README.md` | 本文件 |

## 功能清单

**文件管理（蓝牙）**：连接/自动重连、设备版本与容量、目录浏览、上传（多文件、
**整个文件夹递归**、拖拽）、实时进度条 + 取消、下载、重命名、删除、新建文件夹、
格式化、NFC tag ID 标记。

**内置库（Library 标签）**：页面自带一整套 NFC 标签 `.bin`（按系列/游戏分类），用户
无需自备文件——浏览分类、全局搜索、**一键发到设备**（单个 bin 或整个系列文件夹，
保留子目录结构），也可下载到本地。数据随页面一起托管，用相对路径按需拉取。

**固件升级（OTA / Nordic Secure DFU）**：本地 .zip 或**在线固件列表**、244 字节
大包传输、3 次自动重试、移动端调优、双镜像（SoftDevice+应用）自动衔接、
**中断救援**（设备卡在升级模式也能重新选中续刷）。

### 更新内置库
源数据放在项目根目录的 `ammibodata/`（按 `系列/[子系列/]xxx.bin` 组织）。改动后运行：

```bash
python3 build-library.py
```

它会把 `.bin` 拷进 `tool/library/` 并重新生成 `tool/library.json`。库数据默认随
`tool/` 一起部署（页面用相对路径 `./library/` 取文件）；若想把数据托管到别处（如
CDN），改 `index.html` 里的 `LIBRARY_BASE` 常量即可。

> 注意：设备固件的文件名上限是 **47 字节**，个别超长的标签名会发送失败并在
> 结果里提示（属设备限制，不做静默截断）。

### 配置 OTA 固件列表（文件夹式）
把 OTA 固件 `.zip` 丢进 `tool/firmware/` 文件夹，然后运行：

```bash
python3 build-firmware.py
```

它会扫描该文件夹、生成 `tool/firmware/firmware.json`，页面加载时自动读取并在
「固件升级」标签顶部为每个包渲染一个可点选的按钮（选中即下载、准备好刷入）。
仓库里已放了一个占位空包 `wuzplay-ota-260716.zip`——**上线前请用真实固件替换它**
（空包点「开始升级」会在解析阶段安全报错、不会碰设备）。

- 显示名默认取文件名（去掉 `.zip`）。想要更好看的名字，直接编辑
  `firmware.json` 里的 `name` 字段；重跑脚本会保留你已改过的名字。
- 也可在 `index.html` 的 `FIRMWARE_SOURCES` 数组里手动加托管在别处（同源）的包：
  `{name:"…", url:"…"}`。两者会合并显示。
- 文件夹为空 / 未部署时，该区域自动隐藏。

> `index.html` 会在需要时自动从 CDN 加载 JSZip（仅"固件升级"用到）。

## 部署（关键）

**不要**把它塞进 Tilda 的 HTML/iframe 嵌入块——Web Bluetooth 必须运行在顶层页面，
放进跨域 iframe 会因权限策略连不上设备。Joysfusion 也是这么做的（独立页面）。

正确做法：

1. 把整个 `tool/` 文件夹（含 `library/`、`library.json`、`firmware/`）上传到你们的
   网站服务器，例如让它可以通过 `https://wuzplay.com/tool/` 访问。（**必须是 HTTPS**，
   Web Bluetooth 的硬性要求。）
2. 在 Tilda 里放一个按钮，链接指向 `https://wuzplay.com/tool/`（可设为新标签打开）。

> 没有独立服务器也可以用 **GitHub Pages** 免费托管：新建仓库 → 上传 `tool/` 里的所有文件 →
> Settings → Pages 打开，即可得到一个 HTTPS 地址，再从 Tilda 链接过去。

## 浏览器支持

| 平台 | 是否支持 |
|------|----------|
| 安卓 Chrome | ✅ |
| 电脑 Chrome / Edge | ✅ |
| iPhone / iPad | ❌ Safari 不支持，需用 [Bluefy] 浏览器 |
| 桌面 Safari | ❌ |

页面会自动检测，不支持时顶部会给出提示。

## 自定义品牌

视觉已对齐 wuzplay.com：黑灰背景 + 青蓝强调色 `#00d2ff`（与站点 Logo 一致）、
斜体 "Wuzplay" 字标、5-6px 小圆角、默认英文（中文浏览器自动切中文）。
改颜色只需编辑 `index.html` 顶部 CSS 里的 `--accent`；Logo 文字在 `<div class="logo">` 里。
工具名称与站点栏目一致："Bluetooth Data Manager"。

## 固件升级注意

- 页面上的固件包来自 `tool/firmware/`，由 `build-firmware.py` 生成 `firmware.json`。
  每个包可写 `name`（显示名）、`desc`（一行说明）、`recommended: true`（推荐标记），
  重跑脚本会保留这些字段，推荐包排在最前。
- 当前提供两个固定发布文件名，后续发版直接覆盖同名文件：
  - `wuzplay-full-data.zip` — 固件 + 内置标签库
  - `wuzplay-no-data.zip` — 仅固件，不含标签库
- 页面下载固定文件名时使用 `cache: "no-store"`，避免浏览器继续使用旧固件包。
- 也支持用户手动选择本地固件 `.zip`。
- **JSZip 已本地化**（`jszip.min.js`，MIT）。不能改回 CDN：在 "Start update"
  的点击处理里 await 一个网络请求会耗掉浏览器的 user gesture，
  `requestDevice()` 随后就打不开设备选择框了。
- 同理，恢复路径里 `requestDevice()` 必须是点击后的**第一个 await**，
  固件包解析挪到了选择固件时（`prepareFwImages`）。

### ENTER_DFU (0x02) 的时序约束 —— 不要再踩

固件端（`~/projects/rmiibo`，`src/rmiibo/mod/df/df_proto_info.c`）分两步：

1. 收到 0x02 → **立刻回 OK 帧**；
2. 等这帧发送完成（BLE TX complete）产生 `DF_EVENT_DATA_TRANSMIT_READY`
   → 才 `ble_disable()` + `enter_dfu()` 真正重启。

而 `df_core_on_event()` 在收到 `DF_EVENT_LINK_DISCONNECTED` 时会
`m_df_ctx.cmd = NULL`，把待处理命令槽清掉。

**所以：收到 0x02 的 OK 回包后绝对不能主动断开连接**，否则第 2 步永远
不会执行，设备不重启，bootloader 不广播，选择框自然是空的。
必须等设备**自己**断链 —— 那个断链才是"已重启"的信号
（`PixlClient.waitForDisconnect()`）。

### bootloader 事实（来自固件仓库）

- 广播名 `Wuzplay dfu`（`bootloader/config/sdk_config.h` 的 `NRF_DFU_BLE_ADV_NAME`）。
  旧版按上游 readme 编译的是 `rmiibo dfu`。
- 广播包里**带** 16-bit DFU Service UUID `0xFE59`（`nrf_dfu_ble.c` `advertising_init`），
  所以按 UUID 过滤是可行的，名字前缀只是兜底。
- bootloader 的 MAC 是 app 的 **MAC+1**（`gap_address_change()`），
  所以必须重新弹一次选择框，不能复用之前的 device 对象。
- 手动进 DFU：设备菜单 **设置 → 固件更新**，或者拔电池、按住按键、再装电池。
- 设备重启进 DFU 后，选择器同时按 DFU 服务 UUID 和名称前缀
  （`Dfu` / `rmiibo` / `Pixl` / `Wuz`）过滤；若仍找不到，界面会出现
  “显示全部蓝牙设备”按钮兜底。

## ⚠️ 上线前请用真机测试

蓝牙通信无法在没有设备的情况下自动测试。上线前请务必：
1. 用一台安卓手机 Chrome 打开页面；
2. 连接 → 浏览/上传/下载/重命名/删除一个 `.bin` 文件；
3. （谨慎地）用一台可牺牲的设备测一次固件升级。

## 致谢 / 许可

- BLE 协议层参考自 [Mochi]（MIT License, © tasken）。
- `secure-dfu.js` 基于 [web-bluetooth-dfu]（MIT License, © Rob Moran / Augusto Daniele）。
- 请在正式发布时保留 `secure-dfu.js` 顶部的版权声明。

[Joysfusion NFC Updater]: https://joysfusion.com/nfc-updater/index.html
[Mochi]: https://github.com/tasken/Mochi
[Bluefy]: https://apps.apple.com/app/bluefy-web-ble-browser/id1492822055
[web-bluetooth-dfu]: https://github.com/thegecko/web-bluetooth-dfu
