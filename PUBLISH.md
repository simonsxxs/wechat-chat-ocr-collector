# 发布记录：微信聊天记录采集助手（wechat-chat-ocr-collector）

> 每发布一个平台就更新一行。状态：✅已上架 / ⏳审核中 / ❌失败（写原因）/ ➖不适用

| 平台 | 状态 | 链接 | 发布日期 | 备注 |
|------|------|------|---------|------|
| GitHub | ✅已上架 | https://github.com/simonsxxs/wechat-chat-ocr-collector | 2026-10-15 | 完整包：SKILL.md + scripts/collect.py + references |
| Gitee | ✅已上架 | https://gitee.com/simonsxx/wechat-chat-ocr-collector | 2026-10-15 | GitHub 镜像，SSH 推送 |
| SkillHub | ⏳安全审核中 | https://skillhub.cn/skills/indiv-simonsxx/wechat-chat-ocr-collector | 2026-10-15 | 2026-10-15 提交成功（API 201），分类：办公效率/文档处理，v1.0.0，审核通过后公开 |
| ima 技能广场 | ➖不适用 | | | 用户未指定 |
| aiSkill.market | ➖不适用 | | | 用户未指定 |
| ClawHub | ➖不适用 | | | 用户未指定 |

## 统一物料

- **发布名**：微信聊天记录采集助手
- **slug**：`wechat-chat-ocr-collector`
- **简介（短，50字）**：Windows 上用滚屏截图 + 本地离线 OCR 把微信聊天记录自动采集成纯文本并整理，只读屏幕，不解密不注入不联网，合规安全。
- **简介（长，200字）**：帮你在 Windows 上导出微信 / 企业微信的群聊、私聊记录。原理是把微信窗口提到最前后循环「截图聊天区 → 本地离线 OCR 识别 → 窗口消息向上滚动」，直到翻到最早，合并去重输出纯文本，再按需求整理成提问清单、时间线、待办、可搜索网页等。全程只读屏幕：不解密数据库、不注入进程、不联网上传、不发消息，规避了数据库解密的法律风险。AI 助手加载本技能后可自动装依赖、校准、采集、整理。
- **分类标签**：科技 / 效率工具
- **使用示例（3条）**：
  1. 帮我把这个群的聊天记录导出来，整理成纯文本
  2. 采集一下我跟某某的私聊记录，把里面所有提问提出来
  3. 把群里的聊天记录做成一个可以搜索的网页清单
- **版本号**：1.0.0
- **变更说明**：首次发布。

## 安装包

| 文件 | 用途 |
|------|------|
| `wechat-chat-ocr-collector.zip` | 完整包（SKILL.md + 脚本 + README + LICENSE），GitHub Release / ima 用 |
| `wechat-chat-ocr-collector-skillhub-md.zip` | 纯 md 包（SkillHub 只允许 .md 文件），脚本代码内嵌在 references/collect-script.md |
