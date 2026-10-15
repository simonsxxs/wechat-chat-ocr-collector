# 微信聊天记录采集助手（wechat-chat-ocr-collector）

一个 AI Agent Skill：在 Windows 上用「**滚屏截图 + 本地离线 OCR**」把微信 / 企业微信的群聊、私聊记录自动采集成纯文本，再按需求整理（提取提问、某人发言、待办、时间线、可检索清单）。

**只读屏幕，不解密数据库、不注入进程、不联网上传、不发消息。**

---

## 为什么是这个方案

导出微信聊天记录，网上常见两条路都是死的：

1. **解密微信数据库** —— 微信 4.x 本地聊天库是 SQLCipher 加密的，历史上的解密工具（wx_key / PyWxDump / WeChatMsg）已被全部下架，解密属「绕过技术保护措施」，有法律风险
2. **模拟 Ctrl+A / Ctrl+C 复制** —— 很多受限环境禁止向其他进程注入键鼠输入，按键根本送不进微信窗口

唯一稳定可行的是第三条路：**屏幕截图 + 本地离线 OCR**。把微信窗口提到最前 → 截图聊天区 → 本地 OCR 识别 → 用窗口消息向上滚动 → 循环到最早 → 合并去重输出纯文本。全程只读屏幕。

## 里面有什么

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 技能主文件（含 YAML frontmatter），AI 助手的执行任务书 |
| `scripts/collect.py` | 采集脚本（Windows 实测可用） |
| `references/collect-script.md` | 脚本全文的 md 形式，纯 md 安装包场景下按它重建脚本 |

## 安装

**方式 A：clone 到技能目录**（ZCode / Claude Code 等）

```bash
git clone https://github.com/simonsxxs/wechat-chat-ocr-collector.git ~/.agents/skills/wechat-chat-ocr-collector
```

**方式 B：下载 zip**

在仓库页下载 zip，解压到你的技能目录即可。要求平台支持加载含脚本的技能包；若平台只支持 md，解压后按 `references/collect-script.md` 使用。

## 使用

对支持 Agent Skill 的 AI 助手（ZCode、Claude Code 等）说：

> 帮我把这个群的聊天记录导出来，整理成纯文本
>
> 采集一下我跟某某的私聊记录，把里面所有提问提出来
>
> 把群里的聊天记录做成一个可以搜索的网页清单

AI 会照 `SKILL.md` 自己装依赖、校准、采集、整理。

**手动使用**（不用 AI）：

```bash
pip install pywin32 pillow rapidocr-onnxruntime

python collect.py --list        # 确认窗口（多开账号用 --hwnd 指定）
python collect.py --calibrate   # 校准截图区域（必做）
python collect.py --out "聊天记录.txt"   # 正式采集，后台运行
```

参数说明见 `scripts/collect.py` 文件头。

## 系统要求

- Windows（依赖 Win32 API）
- 微信 PC 版已登录，且**本人先手动点开目标聊天**（程序无法切换会话）
- Python 3.9+

## 能力边界

- ❌ 图片、语音、文件采不到（只能读到 `[图片]`、`[语音]` 占位文字）
- ❌ 不能自动切换聊天会话，一次只能采一个聊天
- ⚠️ 日期按聊天内时间戳就近推断，可能差 ±1 天
- ⚠️ OCR 昵称、时间戳、数字容易错，关键内容需人工校对

## 合规说明

本工具只读取用户自己设备屏幕上的内容，用于整理用户自己的聊天记录：不读取、不解密任何数据库；不注入进程、不绕过任何安全机制；不联网上传、不发送任何消息。整理结果请妥善保管，涉及他人隐私请勿外传。

---

## 作者

- **作者**：simon，一个不懂代码的AI极客
- **微信交流群**：simonsxxs
- **GitHub**：https://github.com/simonsxxs
- **Gitee**：https://gitee.com/simonsxx

## 许可证

[MIT](LICENSE)
