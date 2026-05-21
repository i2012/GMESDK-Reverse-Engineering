# GME SDK 音频质量配置文件分析
本项目包含对腾讯 **GME (Game Multimedia Engine) SDK** (`gmesdk.dll`) 的逆向分析数据库文件（IDA Pro `.i64` 格式），以及某动漫游戏中语音通信系统的技术分析。

## 免责声明
- 本仓库仅提供 IDA Pro 数据库文件 (`.i64`) 用于**教育研究和安全分析**目的
- 不提供任何已修补的二进制文件 (`.dll`)
- 请遵守相关软件的用户协议和当地法律法规

## 文件依赖关系
```
AnimeGameClient.exe (Unity 游戏主程序)
  │
  ├── Plugins/TencentGME.dll  (腾讯 GME Wwise 集成桥接层)
  │     │
  │     └── gmesdk.dll  (腾讯 GME SDK 核心库)  <── 本数据库对应文件
  │
  ├── Plugins/Wwise 相关组件...
  └── Plugins/其他 Unity 原生插件...
```

### 调用链

```
AnimeGameClient.exe
  └→ LoadLibrary("TencentGME.dll")
       └→ GMEWWisePlugin_SetAudioStreamProfile (导出函数)
            └→ LoadLibrary("gmesdk.dll")
                 └→ GetProcAddress("GMESDK_SetAudioStreamProfile")
                      └→ gmesdk.dll!GMESDK_SetAudioStreamProfile
                           └→ AppRoom::PrepareEncParam
                                └→ JSON 配置解析 → 编解码器初始化
```

## 分析过程记录

### 1. 定位麦克风/语音功能

在 AnimeGameClient.exe 中搜索音频相关字符串，发现以下关键导出函数：

| Function | Description |
|--------|------|
| `GMEWWisePlugin_SetAudioStreamProfile` | **设置音频流质量配置** |
| `GMEWWisePlugin_GetMicCount` | 获取麦克风数量 |
| `GMEWWisePlugin_SetAuthInfo` | 设置认证信息 |
| `GMEWWisePlugin_GetAudioSendStreamLevel` | 获取发送流音量 |
| `GMEWWisePlugin_GetAudioRecvStreamLevel` | 获取接收流音量 |
| `GMEWWisePlugin_SetLogLevel` / `SetLogPath` | 日志设置 |
| `GMEWWisePlugin_Pause` / `Resume` | 暂停/恢复 |
| `GMEWWisePlugin_GetMessage` | 消息收发 |

### 2. 追踪音频流配置调用

在 AnimeGameClient.exe 中找到调用链：

```
sub_1464287F0(a1, a2, profile_value, a4, a5, a6)
  │ 参数: profile_value = 1/2/3
  │
  ├── (profile_value - 1) <= 2  (验证值范围 1-3)
  │
  └── GetProcAddress("GMEWWisePlugin_SetAudioStreamProfile")
       └→ 实际转发至 TencentGME.dll
```

### 3. 分析 TencentGME.dll

`GMEWWisePlugin_SetAudioStreamProfile` 在 TencentGME.dll 中实现为：

```c
__int64 GMEWWisePlugin_SetAudioStreamProfile_0(unsigned int a1) {
    HMODULE v2 = sub_180004B40();  // LoadLibrary("gmesdk.dll")
    FARPROC func = GetProcAddress(v2, "GMESDK_SetAudioStreamProfile");
    if (func)
        return func(a1);
    return 1001;  // 错误码
}
```

### 4. 分析 gmesdk.dll

`GMESDK_SetAudioStreamProfile` 在 gmesdk.dll 中最终调用到 `AppRoom::PrepareEncParam`，该函数解析一个硬编码的 JSON 配置字符串。

**JSON 配置数据地址**: `0x14cbba70`

该 JSON 定义了 6 种类型的音频参数，用于 GME SDK 的编解码器初始化和 DSP 处理。

### 5. 配置文件结构

配置文件包含 6 种音频配置，每种配置定义以下参数：

| Parameter | Type | Description | Possible Values |
|------|------|------|--------|
| `sample_rate` | int | 音频采样率 (Hz) | 16000, 48000 |
| `kbps` | int | 音频编码码率 (kbps) | 30, 64, 96... |
| `channel` | int | 声道数 | 1 (单声道), 2 (立体声) |
| `au_scheme` | int | 音频方案优先级编号 | 5-8 (数字越小优先级越高) |
| `codec_prof` | int | 编解码器配置 | 4106 (SILK), 4108 (SILK), 4129 (Opus) |
| `aec` | int | 声学回声消除 | 0 (关闭), 1 (开启) |
| `agc` | int | 自动增益控制 | 0 (关闭), 1 (开启) |
| `ans` | int | 自动噪声抑制 | 0 (关闭), 1 (开启) |
| `ains` | int | 智能噪声抑制 | 0 (关闭), 1 (开启) |
| `frame` | int | 音频帧大小 (ms) | 40 |
| `anti_dropout` | int | 抗丢包策略 | 0/1 |
| `silence_detect` | int | 静音检测 (VAD) | 0/1 |
| `max_antishake_max` | int | 最大抗抖动最大值 (ms) | 1000-2000 |
| `max_antishake_min` | int | 最大抗抖动最小值 (ms) | 400-1200 |
| `min_antishake` | int | 最小抗抖动 (ms) | 200-1000 |

## `gmesdk.dll.i64` patch

### 6种默认配置总览

```
{"data": {"biz_id": 1400079347, "conf": [
  { "role": "esports",   "type": 1,  },
  { "role": "Werewolf",  "type": 2,  },
  { "role": "Rhost",     "type": 3,  },
  { "role": "Raudience", "type": 4,  },
  { "role": "host",      "type": 5,  },
  { "role": "audience",  "type": 6,  }
], "scheme": 3, "sequence": 0}}
```

### patch的配置参数

| Type | Profile | Parameter | **Default** | **Patch** | Notes |
|------|------|------|:----------:|:----------:|------|
| **1** | **esports** | sample_rate | ~~16000~~ | **48000** | 48kHz CD级采样率 |
| | | kbps | ~~30~~ | **96** | 码率提升至3倍 |
| | | agc | ~~1~~ | **0** | 关闭自动增益 |
| | | ans | ~~1~~ | **0** | 关闭自动降噪 |
| | | au_scheme | ~~8~~ | **5** | 高优先级音频方案 |
| 2 | Werewolf | (全部) | 48000/64/0/0 | 不变 | 原始已为高质量 |
| 3 | Rhost | (全部) | 48000/64/0/0 | 不变 | 原始已为高质量 |
| **4** | **Raudience** | sample_rate | ~~16000~~ | **48000** | 48kHz CD级采样率 |
| | | kbps | ~~30~~ | **96** | 码率提升至3倍 |
| | | agc | ~~1~~ | **0** | 关闭自动增益 |
| | | ans | ~~1~~ | **0** | 关闭自动降噪 |
| 5 | host | (全部) | 48000/64/0/0/2ch | 不变 | 原始已为高质量 |
| 6 | audience | (全部) | 48000/64/0/0/2ch | 不变 | 原始已为高质量 |

都是保守值, 更高采样率、码率可以自行尝试 (过高的数值会增加服务器与用户设备的负担)

### 编解码器
SDK 使用两种编解码器：
- **SILK** (codec_prof: 4106, 4108) — Skype 开发的语音编解码器，低码率下语音质量优秀
- **Opus** (codec_prof: 4129) — IETF 标准音频编解码器，支持更广泛的采样率和码率
`au_scheme` 值越小，编解码方案优先级越高（5 > 6 > 7 > 8）。

## 贡献
欢迎通过 Issues 提交改进建议或报告问题, 请注意本仓库不接收二进制文件提交。

## 许可
本项目仅用于教育和研究目的。
