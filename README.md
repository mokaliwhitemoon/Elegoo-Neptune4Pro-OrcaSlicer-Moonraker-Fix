# OrcaSlicer 无法上传到 Klipper/Moonraker —— 3层问题叠加排查实录

> 环境：Elegoo Neptune 4 Pro（已刷 MKS Pi + Armbian Buster + Klipper + Moonraker 1.0.5 + Fluidd 1.19.0）
> 客户端：OrcaSlicer 2.5.1（Elegoo 定制版） / Windows 11

---

## TL;DR

**3个独立问题叠加，最容易误判成「网络问题」。**

| # | 问题 | 根因 | 修复位置 |
|---|---|---|---|
| 1 | 上传报「无法解析服务器响应」 | `moonraker.conf` 缺 2 行 → 上传接口 500 | 打印机 |
| 2 | Orca 主机类型没有 Moonraker 选项 | Elegoo 定制版 UI 锁死 | Orca 预设 JSON |
| 3 | **打印时屏幕不进入打印模式** | **KlipperScreen 开机抢跑，订阅时 Moonraker 未就绪 → 503，此后永不重连** | 打印机 |

### 打印机侧（`moonraker.conf`）

```ini
[server]
host: 0.0.0.0
port: 7125
enable_debug_logging: False
klippy_uds_address: /tmp/klippy_uds
max_upload_size: 2048          # ← 原来没有这行，默认 1024

[file_manager]
config_path: /home/mks/klipper_config
log_path: /home/mks/klipper_logs
queue_gcode_uploads: True      # ← 原来没有这行，默认 False
```

### 打印机侧（`KlipperScreen.conf`）

**原文件只有 123 字节，全是注释，一行有效配置都没有：**
也不知道这些家伙为什么要这么干（（（

```ini
#~# --- Do not edit below this line. This section is auto generated --- #~#
#~#
#~# [main]
#~# theme = material-light
#~#
```

改成：

```ini
[printer generic]
moonraker_host: 127.0.0.1
moonraker_port: 7125

[main]
theme = material-light
```

### Orca 侧（机器预设 JSON）

```json
"host_type": "moonraker",
"print_host": "<PRINTER_IP>:7125",
"print_host_webui": "http://<PRINTER_IP>"
```

改完都必须**重启对应服务**，见各章节。

---

## 一、问题方向

### 陷阱 1：「无法解析服务器响应」

这个报错**不是连不上**，是 Orca 拿到了非预期的响应格式。

### 陷阱 2：主机类型选 Elegoo Link

海王星 4 Pro 出厂确实支持 ElegooLink，但**刷了 Klipper 之后 ElegooLink 就不存在了**。Orca 却默认给它选了 Elegoo Link，请求打到 `http://IP/api/upload` → 404。

---

## 二、真正的排查过程

### 第 1 步：确认打印机本身健康

```bash
# 网页端
curl -s http://<PRINTER_IP>:7125/server/info
```

```json
{
  "result": {
    "klippy_connected": true,
    "klippy_state": "ready",
    "components": ["klippy_connection", "file_manager", ...],
    "failed_components": [],
    "api_version_string": "1.0.5"
  }
}
```

`klippy_state: ready` + `failed_components: []` → **Klipper 和 Moonraker 都正常**。

### 第 2 步：看 Moonraker 日志（你要是敢不看你就敢改你也是人才）

配置可以通过 HTTP 直接读，**不需要 SSH**：

```bash
# 读配置
curl -s http://<PRINTER_IP>:7125/server/files/config/moonraker.conf

# 读日志
curl -s http://<PRINTER_IP>:7125/server/files/logs/moonraker.log
```

日志里赫然写着：

```
2026-11-15 13:56:52 [web.py:log_exception()] - Uncaught exception POST /server/files/upload
2026-11-15 13:56:52 [app.py:log_request()] - 500 POST /server/files/upload
```

**这就是答案。** 上传接口在崩，不是网络问题、不是 Orca 问题。

### 第 3 步：复现 500

```bash
curl -X POST http://<PRINTER_IP>:7125/server/files/upload
```

响应体：

```
streaming_form_data.parser.ParseFailedException: Content-Type is not multipart/form-data
Traceback (most recent call last):
  File "/home/mks/moonraker-env/lib/python3.7/site-packages/tornado/web.py", line 1681, in _execute
  File "/home/mks/moonraker/moonraker/app.py", line 841, in prepare
    self._parser = StreamingFormDataParser(self.request.headers)
```

MKS 这台的 Moonraker 跑在 **Python 3.7 + Tornado 6.1**，`streaming_form_data` 是很老的版本，解析大文件分块时容易崩。

### 第 4 步：查配置确认缺项

```bash
curl -s http://<PRINTER_IP>:7125/server/config | python -m json.tool
```

发现：

```json
"server": {
    "host": "0.0.0.0",
    "port": 7125,
    "max_upload_size": 1024    ← 默认值，配置文件里根本没写
},
"file_manager": {
    "config_path": "/home/mks/klipper_config",
    "log_path": "/home/mks/klipper_logs",
    "queue_gcode_uploads": false  ← 默认值，配置文件里根本没写
}
```

**`max_upload_size` 和 `queue_gcode_uploads` 都在吃默认值。**

---

## 三、修复 Moonraker

### 在 Fluidd 网页里改（不需要 SSH）

打开 `http://<PRINTER_IP>` → 左侧 `{...}` 图标（Configuration Files）→ 点开 `moonraker.conf`：

```ini
[server]
host: 0.0.0.0
port: 7125
enable_debug_logging: False
klippy_uds_address: /tmp/klippy_uds
max_upload_size: 2048           # ← 加这行

[file_manager]
config_path: /home/mks/klipper_config
log_path: /home/mks/klipper_logs
queue_gcode_uploads: True       # ← 加这行
```

### 重启 Moonraker（你可以选择不重启你要是不重启跑不起来别怪我）

**改了文件不等于生效**，内存里跑的还是旧配置（很神奇对吧）

Fluidd → 左侧齿轮 ⚙ → Service/System → 找 `moonraker` → Restart

或在 Console 里输 `RESTART`。

### 验证生效

```bash
# 1. 确认配置读进来了
curl -s http://<PRINTER_IP>:7125/server/config | grep -E 'max_upload_size|queue_gcode'
# 应输出：max_upload_size = 2048 / queue_gcode_uploads = True

# 2. 实测上传（造个假文件）
echo "G28" > test.gcode
curl -X POST -F "file=@test.gcode" -F "root=gcodes" \
     http://<PRINTER_IP>:7125/server/files/upload
```

**修复前**：`500 Internal Server Error`
**修复后**：`201 Created`

```json
{"item": {"path": "test.gcode", "root": "gcodes"}, "action": "create_file"}
```

---

## 四、修复 OrcaSlicer

### 问题：主机类型下拉框只有「Elegoo Link」

这个 Elegoo 定制版把选项藏起来了，也**没有「高级模式」开关**可切。

但源码里功能是全的。`PrintConfig.cpp` 里 `host_type` 枚举定义：

```cpp
def->enum_values.push_back("prusalink");
def->enum_values.push_back("prusaconnect");
def->enum_values.push_back("octoprint");      // 界面显示 Octo/Klipper
def->enum_values.push_back("duet");
def->enum_values.push_back("ultimaker");
def->enum_values.push_back("flashair");
def->enum_values.push_back("astrobox");
def->enum_values.push_back("repetier");
def->enum_values.push_back("mks");
def->enum_values.push_back("esp3d");
def->enum_values.push_back("crealityprint");
def->enum_values.push_back("obico");
def->enum_values.push_back("flashforge");
def->enum_values.push_back("simplyprint");
def->enum_values.push_back("elegoolink");
def->enum_values.push_back("3dprinteros");
def->enum_values.push_back("moonraker");      // ← 我们需要这个
```

`PrintHost.cpp` 里工厂函数确认它真的能用：

```cpp
case htMoonraker: return new Moonraker(config);
```

### 解法：直接改机器预设 JSON

路径：`%APPDATA%\OrcaSlicer\user\default\machine\<你的预设>.json`

例如 `Elegoo Neptune 4 Pro 0.4 nozzle - 拷贝.json`：

```json
{
	"from": "User",
	"host_type": "moonraker",
	"inherits": "Elegoo Neptune 4 Pro 0.4 nozzle",
	"name": "Elegoo Neptune 4 Pro 0.4 nozzle - 拷贝",
	"print_host": "<PRINTER_IP>:7125",
	"print_host_webui": "http://<PRINTER_IP>",
	"printer_extruder_id": ["1"],
	"printer_extruder_variant": ["Direct Drive Standard"],
	"printer_settings_id": "Elegoo Neptune 4 Pro 0.4 nozzle - 拷贝",
	"version": "2.4.0.14"
}
```

**重启 Orca → 下拉框出现 `Moonraker (...)`** ✅

### 界面里填什么

| 字段 | 值 |
|---|---|
| 主机类型 | `Moonraker (Klipper)` |
| 主机名 / IP / URL | `<PRINTER_IP>:7125` |
| 设备用户界面 | `http://<PRINTER_IP>` |
| API密钥 / 密码 | 留空（`trusted_clients` 已放行局域网） |
| HTTPS CA 文件 | 留空 |

**注意**：主机名填**纯 `IP:端口`**，别加 `http://` 前缀——某些版本解析会失败。

---

## 五、踩坑清单（你猜猜我研究多久tm的一周）

### 1. `OrcaSlicer.conf` 有 MD5 校验

文件最后一行：

```
# MD5 checksum CFBCB4678A1B8EB1EBC7631F2BD50EAE
```

**只改内容不更新 MD5，改动会被丢弃。**

算法（逆向出来的）：

```powershell
$raw = [IO.File]::ReadAllText($conf)
$idx = $raw.LastIndexOf('# MD5 checksum')
$content = $raw.Substring(0, $idx)
# 1. CRLF → LF
# 2. 去掉末尾换行
$normalized = ($content -replace "`r`n","`n").TrimEnd("`n")
# 3. UTF-8 → MD5 → 大写十六进制
```

### 2. PowerShell 函数名别叫 `H`

`H` 是 `Get-History` 的内置别名，会报一堆「找不到 ID xxx 的历史记录」。

```powershell
function H($bytes){...}   # ❌ 炸
function Get-Md5Hex($bytes){...}   # ✅
```

### 3. 项目文件会绑定打印机预设

`.3mf` 项目文件里记录了它用的打印机。改 Neptune 预设时，如果当前打开的项目绑的是 `Bambu Lab A1`，**改动完全不生效**。

必须先在北侧下拉框切到你改的那个预设，再打开对话框。

配置里的证据：

```json
"machine": "Bambu Lab A1 0.4 nozzle(ksr+FDMTest.3mf)"     ← 项目绑的
"machine": "Elegoo Neptune 4 Pro 0.4 nozzle - 拷贝"        ← 你改的
```

### 4. `raw.githubusercontent.com` DNS 经常挂

```
curl: (6) Could not resolve host: raw.githubusercontent.com
```

改用 GitHub API：

```bash
curl -s "https://api.github.com/repos/OrcaSlicer/OrcaSlicer/contents/path/to/file?ref=v2.5.1"
# 返回 base64 的 content 字段，解码后就是源码
```

```powershell
$j = (Invoke-WebRequest $url).Content | ConvertFrom-Json
$txt = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($j.content))
```

### 5. GitHub API 未认证有频率限制

大量扫描源码会撞 429。要批量查就分批、加延时，或者直接用 git clone。

### 6. Moonraker 的 `/machine/system_info` 在老版本返回空壳

```json
{"result": {"system_info": ...}}   ← 其他字段全空
```

`moonraker_version` 甚至返回字面量 `"?"`。这会让按版本号做分支判断的客户端走错路。

### 7. 日志显示 201 但文件没出现？

核对**文件名**。为了测试上传而造的临时文件也会返回 201，别把测试请求当成用户的操作成功了。

---

## 六、屏幕不进入打印模式（最隐蔽的一个）

**这是另一个独立问题，和上传完全无关。** 上传修好、打印正常跑起来之后才暴露出来。

### 现象

- 打印**确实在进行**（温度、坐标都在动，网页端进度正常）
- 但**物理屏幕停留在主菜单**，不切换到打印界面
- **屏幕上点不了暂停/停止**
- 屏幕温度显示 19/20°C（室温），和网页端的 240°C 完全对不上

### 关键排查：查 websocket 连接

```bash
curl -s http://<PRINTER_IP>:7125/machine/system_info
```

```json
"available_services": ["klipper", "moonraker", "klipper_mcu"]
```

**列表里没有 `KlipperScreen`！** 这不是说它没运行，而是 MKS 没有把它注册成受管服务。

再看日志里谁连上了：

```bash
curl -s http://<PRINTER_IP>:7125/server/files/logs/moonraker.log
```

找到决定性证据：

```
14:17:22 [authorization.py] Trusted Connection Detected, IP: 127.0.0.1
14:17:22 [app.py] 101 GET /websocket? (127.0.0.1) [_TRUSTED_USER_]
14:17:22 [websockets.py] Websocket Opened: Proxied: False,
          User Agent: WebSocket++/0.8.1, Host Name: localhost
14:17:25 [websockets.py:build_error()] - JSON-RPC Request Error: 503
14:17:25 [websockets.py:build_error()] - JSON-RPC Request Error: 503
14:17:26 [websockets.py:build_error()] - JSON-RPC Request Error: 503
```

**两个信息点：**

1. `User Agent: WebSocket++/0.8.1` + `Proxied: False` + `127.0.0.1`
   → 这不是浏览器（浏览器是 `Mozilla/5.0`），**这是屏幕程序（KlipperScreen）本体**
2. **连上后立刻收到 3 次 `503 JSON-RPC Request Error`**

### 根因：开机时序竞争

KlipperScreen 启动比 Moonraker 快，等它去订阅数据时，Moonraker 还在初始化 → 返回 503 → **KlipperScreen 不重试，永久放弃**。

所以：
- **界面能显示** → KlipperScreen 自己渲染，不需要 Moonraker
- **打印状态拿不到** → 订阅失败，永远显示旧状态
- **温度对不上** → 没订阅到温度数据

### 为什么 `KlipperScreen.conf` 是空的

原文件只有 123 字节，**全是注释，没有任何有效配置**：

```ini
#~# --- Do not edit below this line. This section is auto generated --- #~#
#~#
#~# [main]
#~# theme = material-light
#~#
```

也就是说这台机器**从来没有配过 Moonraker 连接参数**，一直靠某种默认值在跑，而默认值有时序问题。

### 修复

**在 Fluidd 里编辑**（左侧 `{...}` 图标 → 点 `KlipperScreen.conf`），全部替换为：

```ini
[printer generic]
moonraker_host: 127.0.0.1
moonraker_port: 7125

[main]
theme = material-light
```

**⚠️ 注意：如果浏览器装了自动翻译插件，不要让它把配置翻译成中文！（因为他翻译的东西真的会影响到你写的东西最好用原本语言去写我不要挂翻译）**

翻译后变成这样就是错的，KlipperScreen 完全读不懂：

```ini
[打印机通用]          # ❌ 必须是 [printer generic]
moonraker_host: 127.0.0.1
moonraker_port: 7125

[主要角色]            # ❌ 必须是 [main]
主题 = 物质-光        # ❌ 必须是 theme = material-light
```

**改完必须重启打印机电源**（KlipperScreen 不是 systemd 服务，`RESTART` 命令不管它）。

### 验证

重启后确认屏幕温度显示**真实室温**（约 20°C），而不是固定不变的 19°C。

然后发一个打印任务，屏幕应切换出：进度条、温度、暂停/停止按钮。

### 已知残留问题（不影响使用）

- **缩略图不显示**：gcode 文件里确实有 `; thumbnail begin 160x160 6440` 数据，但 MKS 定制版 KlipperScreen 不渲染预览图（大概率是精简版砍了这个功能或缺少图像库依赖）。不影响功能。
- **每次开机可能需要重启屏幕**：503 是时序竞争，开机时如果 KlipperScreen 又抢跑，得再重启一次。

---

---

## 七、验收标准

全部通过才算修好：

```bash
# 1. Moonraker 活着
curl -s http://<PRINTER_IP>:7125/server/info | grep klippy_state
# → "klippy_state": "ready"

# 2. 配置生效
curl -s http://<PRINTER_IP>:7125/server/config | grep -E 'max_upload_size|queue_gcode'
# → max_upload_size = 2048
# → queue_gcode_uploads = True

# 3. 上传可用
curl -X POST -F "file=@test.gcode" -F "root=gcodes" \
     http://<PRINTER_IP>:7125/server/files/upload
# → 201 Created

# 4. Orca 主机类型
# 界面里显示 "Moonraker (Klipper)"，测试按钮通过

# 5. 屏幕已连上（关键）
curl -s http://<PRINTER_IP>:7125/server/files/logs/moonraker.log | grep 'WebSocket++'
# → 应看到 User Agent: WebSocket++/0.8.1
# → 且后面没有 503 Error
```

---

## 七、备选方案（如果 Orca 上传实在修不好）

**直接绕过 Orca 的上传通道：**

1. Orca 里切片 → 导出 G-code 到本地
2. 浏览器打开 `http://<PRINTER_IP>`（Fluidd）
3. 把 gcode 拖进文件管理区上传
4. 点打印

**这条路径一直可用**，完全不依赖 Orca 的印主机配置。虽然多两步，但零风险。

---

## 八、安全提醒

排查过程中发现这台机器的默认配置相当开放：

```ini
[authorization]
trusted_clients:
    192.168.0.0/16      # 信任整个局域网段
```

Orca 配置里：

```json
"access_code": "<ACCESS_CODE>"   // Neptune 出厂默认值
```

**后果**：同一 WiFi 下任何设备（包括蹭网的）都能无密码操控打印机、看摄像头。

建议加固：

```ini
[authorization]
trusted_clients:
    192.168.2.0/24      # 收窄到实际网段
force_logins: True      # 强制登录
```

同时改掉 `access_code`。

---

## 附：关键路径速查

| 项目 | 位置 |
|---|---|
| Orca 配置 | `%APPDATA%\OrcaSlicer\OrcaSlicer.conf` |
| Orca 机器预设 | `%APPDATA%\OrcaSlicer\user\default\machine\*.json` |
| Orca 日志 | `%APPDATA%\OrcaSlicer\log\debug_*.log.0` |
| Moonraker 配置 | `http://<PRINTER_IP>:7125/server/files/config/moonraker.conf` |
| Moonraker 日志 | `http://<PRINTER_IP>:7125/server/files/logs/moonraker.log` |
| Moonraker 实时配置 | `http://<PRINTER_IP>:7125/server/config` |
| 系统信息 | `http://<PRINTER_IP>:7125/server/info` |
| 文件列表 | `http://<PRINTER_IP>:7125/server/files/list?root=gcodes` |

**重要**：Moonraker 的配置和日志都能通过 HTTP 直接读，**不需要 SSH 密码**。这是本次排查省时间最多的一条。

---

本身也是研究很久才搞定的一开始搞来搞去写了半天也不知道写出来一个什么屌玩意毕竟我也知道我自己的语言跟一坨屎似的
希望这些能帮助到各位在研究自己打印机无法连接的问题
本身也是算半个入门的新手只是会点语言罢了希望能帮助到各位如果有更好的改进方案的话也可以提出来加入项目帮忙改进一下特别感谢

---

*License: 随意使用、修改、转载，不用署名。*
