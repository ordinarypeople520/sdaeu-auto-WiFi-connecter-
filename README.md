# 校园网自动登录（锐捷 RG-SAM+ / 10.22.10.82）

开机自动联网、掉线自动重连，全程不弹网页、不用手输账号密码。

已验证的认证环境：

| 项目 | 值 |
| --- | --- |
| 网关 | `http://10.22.10.82/` |
| 系统 | 锐捷 RG-SAM+ Portal（ePortal 组件） |
| 登录接口 | `POST http://10.22.10.82/eportal/InterFace.do?method=login` |
| 参数编码 | `userId` / `password` / `queryString` 需双重 URL 编码 |
| 密码 | 明文或 RSA 加密两种模式，程序会自动识别登录页配置 |

---

## 一、三步装好

**第 1 步：双击 `setup.bat`**

输入学号（账号）和密码。密码输入时不显示，保存后由 Windows DPAPI 加密，
只有当前 Windows 用户能解密，明文不落盘。设置完会立刻试登录一次。

**第 2 步：双击 `install.bat`**

写入开机自启（当前用户登录 Windows 后自动在托盘运行），并立即启动托盘程序。
右下角出现 WiFi 图标就说明它在工作了。

> 想在桌面/开始菜单放个入口，可以再双击 `创建快捷方式.bat`（可选）。

**第 3 步：完成**

之后不用管它。默认每 20 秒检测一次网络，一旦发现未认证/断网就自动登录；
登录失败会退避重试（15s → 30s → 60s → ... 最长 5 分钟），不会把账号刷到锁定。
想退出就右键托盘图标 → 退出。

---

## 二、文件说明

| 文件 | 作用 |
| --- | --- |
| `setup.bat` | 设置 / 修改账号密码 |
| `install.bat` | 开启开机自启 + 启动托盘程序 |
| `tray.bat` | 只启动托盘程序（不开自启） |
| `uninstall.bat` | 关闭自启 + 结束托盘程序 |
| `login-now.bat` | 立即登录一次（不想等托盘，手动触发用） |
| `status.bat` | 看当前联网状态、账号、自启状态 |
| `selftest.bat` | 网络与认证接口自检（排查问题时先跑这个） |
| `创建快捷方式.bat` | 在桌面和开始菜单创建带图标的快捷方式（可选） |
| `CampusLogin.ps1` | 程序本体（纯 PowerShell，无需装 Python / 第三方库） |
| `make-icon.ps1` | 图标生成器（改配色/换图形后重跑即可重新生成 icons 目录） |
| `icons\` | 图标文件：`app.ico` 主图标，`online`/`offline`/`busy` 三种状态 |

### 托盘图标

运行后右下角托盘出现一个**圆角方形 + WiFi 符号**的图标，底色代表状态：

| 图标 | 状态 |
| --- | --- |
| 🟢 绿色 | 已联网 |
| 🔴 红色 | 未认证 / 断网 |
| 🟠 橙色 | 正在登录 |
| 🔵 蓝色 | 仅用于快捷方式/程序标识（`app.ico`） |

鼠标悬停显示状态和 IP。图标是多尺寸 ICO（16～256 px），高分屏缩放下不会糊。

想换个配色或图形，改 `make-icon.ps1` 底部的色值再跑一次即可，
程序启动时自动读取 `icons\` 下的文件；文件缺失会回退成内置的纯色圆点。

**右键**图标：

| 菜单项 | 作用 |
| --- | --- |
| 状态 / 账号 | 当前状态和账号（灰色，只用来查看） |
| 立即登录 | 立刻强制登录一次（双击图标同效） |
| 查看状态 | 弹窗显示账号、网关、IP、自启状态、日志路径 |
| 打开日志 | 用记事本打开 `login.log` |
| 修改账号密码 | 打开设置窗口，改完立即生效 |
| 开机自动启动 | 勾选/取消，即时生效 |
| **退出** | 结束程序，托盘图标消失，之后不再自动登录 |

> 退出只是本次运行结束；开机自启如果还勾着，下次登录 Windows 它会自己回来。
> 想永久关掉，用 `uninstall.bat`（或托盘菜单里取消"开机自动启动"再退出）。

数据与日志（不在程序目录，卸了程序也不影响）：

```
C:\Users\<你的用户名>\AppData\Local\CampusAutoLogin\
    config.json    配置（网关地址、检测间隔等）
    cred.dat       加密后的密码
    state.json     最近一次登录的会话信息（注销时用）
    login.log      运行日志，超过 1MB 自动轮转
```

---

## 三、常用命令

在程序目录打开 PowerShell：

```powershell
# 立即登录一次并看详细过程
powershell -ExecutionPolicy Bypass -File .\CampusLogin.ps1 -Once -VerboseLog

# 看状态
powershell -ExecutionPolicy Bypass -File .\CampusLogin.ps1 -Status

# 当前账号主动下线
powershell -ExecutionPolicy Bypass -File .\CampusLogin.ps1 -Logout

# 修改检测间隔为 10 秒并保存
#   编辑 C:\Users\<你>\AppData\Local\CampusAutoLogin\config.json 里的 interval
```

---

## 四、config.json 可调项

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `interval` | 20 | 在线时的检测间隔（秒），越小越灵敏、越费电 |
| `account` | - | 账号，由 setup 写入 |
| `service` | 空 | 服务类型。门户有"校园网/运营商"选择且登录报服务错误时再填 |
| `passwordEncrypt` | `auto` | `auto` 自动识别登录页是否要求 RSA；也可强制 `true`/`false` |
| `registerMac` | `false` | 置 `true` 会登录后尝试注册 MAC 无感知认证（部分学校未开放） |
| `maxBackoff` | 300 | 连续失败时最长退避秒数 |

---

## 五、常见问题

**1. 日志里出现"门户当前要求输入图形验证码"**

门户触发了验证码（通常是短时间内多次失败或第一次使用）。手动打开
`http://10.22.10.82/` 登录一次，验证码通过后再让程序接管，一般就不会再要求了。

**2. 开了 Clash / v2rayN 之类的代理，程序还能用吗**

能。程序所有请求都强制直连，不经过系统代理。

**3. 换了 WiFi、换了网段（比如教学楼 → 宿舍）**

不用改任何东西。程序每次登录都重新读取本机 IP 和网关下发的会话参数。

**4. 提示"该账号已在线"**

说明这个账号在别的设备（手机/另一台电脑）上在线。校园网通常一个账号只能一个终端在线，
先让那台设备下线，或者等它超时。

**5. 怎么确认它真的在工作**

看托盘图标底色：绿色=已联网，红色=没认证，橙色=正在登录。鼠标悬停会显示状态和 IP。
再看日志：

```
type "%LOCALAPPDATA%\CampusAutoLogin\login.log"
```

日志里出现 `[OK] 认证成功` 就是正常工作；断网后 20~40 秒内应出现新的成功记录。

**6. 想彻底删掉**

先跑 `uninstall.bat`，再删除本文件夹和
`C:\Users\<你>\AppData\Local\CampusAutoLogin\` 即可，注册表只动了
`HKCU\Software\Microsoft\Windows\CurrentVersion\Run\CampusAutoLogin` 一项。
