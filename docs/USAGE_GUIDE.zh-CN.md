# PiPup 使用说明

PiPup 是 Android TV、Google TV 与 Fire TV 的局域网弹窗应用。应用在电视上监听 `7979` 端口，家庭自动化系统或电脑可以通过 HTTP 请求展示文字、图片、视频与网页内容。

## 使用条件

- Android 7.0 或更高版本。
- 电视与发送通知的设备位于同一可信局域网。
- 已安装 `adb` 的电脑用于首次侧载和授权。
- 电视具备可用网络连接。

## 安装 APK

1. 在电视设置中开启开发者选项与 ADB 调试。
2. 将电脑连接至电视，首次连接时在电视上确认授权。
3. 安装 APK 并授予悬浮窗权限。

```bash
adb connect <电视IP>:5555
adb install -r app-debug.apk
adb shell appops set nl.rogro82.pipup SYSTEM_ALERT_WINDOW allow
```

安装完成后从电视应用列表打开 PiPup 一次。状态页会显示服务器地址，例如 `192.168.1.50:7979`。

## 中文界面与语音

电视系统语言为简体中文时，PiPup 会显示中文状态页、服务通知和更新提示。通知正文支持中文。中文语音需要电视安装并启用中文 TTS 引擎，发送通知时指定 `ttsLanguage` 为 `zh-CN`。

## 发送第一条通知

将 `<电视IP>` 替换为 PiPup 状态页显示的电视 IP：

```bash
curl -X POST "http://<电视IP>:7979/notify" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "门铃提醒",
    "message": "有人在门口",
    "duration": 15,
    "position": "TopRight",
    "tts": "有人在门口",
    "ttsLanguage": "zh-CN"
  }'
```

## `/notify` 字段说明

JSON 请求的顶层字段全部可选。发送 `{}` 会以默认样式显示一个持续 30 秒的空白弹窗；通常建议至少提供 `title` 或 `message`。

| 字段 | 类型 | 必填性 | 默认值 | 作用 |
|---|---|---|---|---|
| `duration` | 整数 | 可选 | `30` | 弹窗显示秒数；`0` 或负数表示持续显示，直到被关闭或被新弹窗替换。 |
| `id` | 字符串 | 可选 | 无 | 弹窗标识。相同 `id` 且内容相同的通知会保留视图并重新计时。 |
| `position` | 字符串枚举 | 可选 | `TopRight` | 弹窗位置：`TopRight`、`TopLeft`、`BottomRight`、`BottomLeft`、`Center`。 |
| `backgroundColor` | 字符串 | 可选 | `#CC000000` | 背景色，格式为 `#RRGGBB` 或 `#AARRGGBB`。 |
| `title` | 字符串 | 可选 | 无 | 标题文本，支持中文。 |
| `titleSize` | 数字 | 可选 | `16` | 标题字号，单位为 Android `sp`。 |
| `titleColor` | 字符串 | 可选 | `#ffffff` | 标题颜色。 |
| `message` | 字符串 | 可选 | 无 | 正文文本，支持中文。 |
| `messageSize` | 数字 | 可选 | `12` | 正文字号，单位为 Android `sp`。 |
| `messageColor` | 字符串 | 可选 | `#ffffff` | 正文颜色。 |
| `media` | 对象 | 可选 | 无 | 外部图片、视频或网页媒体。 |
| `tts` | 字符串 | 可选 | 无 | 弹窗出现时朗读的文本。 |
| `ttsLanguage` | 字符串 | 可选 | 电视默认语言 | TTS 语言标签；简体中文使用 `zh-CN`。 |
| `urgency` | 字符串枚举 | 可选 | 无 | 提示边框级别：`info`、`warning`、`critical`。 |
| `showProgress` | 布尔值 | 可选 | `false` | 对有限时长弹窗显示倒计时进度条。 |
| `buttons` | 数组 | 可选 | `[]` | 遥控器可操作的按钮列表。 |
| `callback` | 字符串 | 可选 | 无 | 按下按钮时接收 POST 回调的地址。 |

### `media` 媒体对象

`media` 必须且只能包含一个类型对象。类型对象中的 `uri` 为必填字段。

| `media` 内容 | 必填字段 | 可选字段 | 默认值 | 作用 |
|---|---|---|---|---|
| `image` | `uri` | `width` | `width: 480` | 显示远程图片。 |
| `video` | `uri` | `width`、`muted` | `width: 480`、`muted: false` | 播放远程视频。 |
| `web` | `uri` | `width`、`height`、`muted` | `width: 640`、`height: 480`、`muted: false` | 显示网页、直播页或摄像头流页面。 |

### `buttons` 按钮对象

`buttons` 中每一项都必须同时包含 `id` 和 `label`。

| 字段 | 类型 | 必填性 | 作用 |
|---|---|---|---|
| `id` | 字符串 | 必填 | 按钮的稳定标识，回调载荷中的 `button` 字段使用该值。 |
| `label` | 字符串 | 必填 | 电视界面上显示的按钮文字，支持中文。 |

使用按钮时，确认键会发送 `{"popup","button","label","device","name"}` 到 `callback` 并关闭弹窗；返回键会直接关闭弹窗。

## 常用通知场景

### 持续显示摄像头网页

使用 `duration: 0` 展示持续弹窗。相同 `id` 的同内容通知会延长展示时间并保留已播放的媒体。

```bash
curl -X POST "http://<电视IP>:7979/notify" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "door-camera",
    "title": "门口摄像头",
    "duration": 0,
    "media": {
      "web": {
        "uri": "http://<摄像头服务IP>/live",
        "width": 960,
        "height": 540,
        "muted": true
      }
    }
  }'
```

### 紧急提示与倒计时

```bash
curl -X POST "http://<电视IP>:7979/notify" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "烟雾报警",
    "message": "请立即检查厨房",
    "duration": 60,
    "urgency": "critical",
    "showProgress": true,
    "tts": "烟雾报警，请立即检查厨房",
    "ttsLanguage": "zh-CN"
  }'
```

## 关闭弹窗

```bash
curl -X POST "http://<电视IP>:7979/cancel"
curl -X POST "http://<电视IP>:7979/cancel?id=door-camera"
```

## 查询状态

```bash
curl "http://<电视IP>:7979/state"
```

响应包含应用版本、在线时长、屏幕状态、当前弹窗、最近一次收到的弹窗和设备信息。`visible: true` 表示当前有弹窗显示。

## 安全说明

PiPup 的 HTTP 服务没有认证，通信采用明文 HTTP。将运行 PiPup 的电视和发送通知的设备放入可信网络；请为安全敏感的按钮回调使用不可预测的一次性令牌，并在接收端验证令牌。
