---
name: create-yourself
description: 数字永生自我蒸馏 Skill。对话式录入代号、基本信息、自我画像并可导入聊天记录、日记、照片等原材料，分析后生成专属自我记忆与人格 Skill。当用户要求创建自己的 skill、把自己蒸馏成 skill、数字永生、新建自我镜像时使用。
---

# 数字永生 · 自我 Skill 创建器（Trae 适配版）

把用户本人蒸馏成一个可运行的自我 Skill：录入基础信息、导入原材料（聊天记录 / 日记 / 照片 / 口述），分析生成 Part A 自我记忆 + Part B 人格，组合成可被 Trae 调用的项目 Skill。

## 运行环境（Trae Code · Windows）

- 命令环境是 **PowerShell**（不是 Bash），不要使用 `mkdir -p`、heredoc、`/tmp` 等 Unix 语法
- 相对路径一律基于 **git 仓库根目录**（本 skill 位于 `<repo>/.trae/skills/create-yourself/`）
- Python 用 `python` 命令；中文乱码时先执行 `$env:PYTHONIOENCODING='utf-8'`
- 本 skill 的文件布局：
  - `scripts/` — Python 工具（聊天记录解析、照片分析、文件管理、版本管理）
  - `references/` — 分步骤提示模板，用到时再读取
- 生成的自我 Skill 写入 `<repo>/.trae/skills/{slug}/`，共 4 个文件：`SKILL.md`（组合版）、`self.md`、`persona.md`、`meta.json`

## 触发条件

- 用户要求创建自己的 skill / 数字分身 / 数字永生，例如"帮我创建一个自己的 skill""把自己蒸馏成 skill""新建自我镜像"
- 对已有自我 Skill 追加材料时进入进化模式（见下文）

## 主流程：创建新自我 Skill

### Step 1 基础信息录入（3 个问题）

读取 [references/intake.md](references/intake.md) 的问题序列，只问 3 个问题：

1. **代号/昵称**（必填）：如"小北""20岁的我"。按 slug 规则转拼音/小写：中文转拼音连字符连接（青云 → qing-yun），英文小写、空格换 `-`
2. **基本信息**（可跳过）：年龄、职业、城市等一句话。解析为 age / occupation / city / education 字段
3. **自我画像**（可跳过）：MBTI、星座、性格标签、自我评价。解析为 mbti / zodiac / personality 标签列表 / impression

除代号外均可跳过。收集完先汇总确认再继续。

### Step 2 原材料导入

询问用户选哪种来源（可混用，可跳过）：

```
原材料怎么提供？数据越多，还原度越高。
  [A] 微信聊天记录导出（WeChatMsg / 留痕 / PyWxDump）
  [B] QQ 聊天记录导出（txt / mht）
  [C] 社交媒体 / 日记 / 笔记（截图、Markdown、TXT）
  [D] 照片（提取 EXIF 时间地点，构建人生时间线）
  [E] 直接粘贴/口述
```

- 方式 A（微信）：

```powershell
python .trae/skills/create-yourself/scripts/wechat_parser.py --file {path} --target "我" --output $env:TEMP\wechat_out.txt --format auto
```

- 方式 B（QQ）：

```powershell
python .trae/skills/create-yourself/scripts/qq_parser.py --file {path} --target "我" --output $env:TEMP\qq_out.txt
```

- 方式 C / E：聊天记录、日记、笔记等文本直接用 Read 工具读取；截图用 Read 工具查看；口述内容直接作为文本原材料。可参考 [references/intake.md](references/intake.md) 末尾的引导问题让用户口述

- 方式 D（照片）：

```powershell
python .trae/skills/create-yourself/scripts/photo_analyzer.py --dir {photo_dir} --output $env:TEMP\photo_out.txt
```

用户说"没有文件"或"跳过"时，仅凭 Step 1 的手动信息生成（先读取 references/self_builder.md 与 references/persona_builder.md 了解结构，信息不足处标注 `[待补充]`）。

### Step 3 分析原材料

把全部原材料与基础信息汇总，按两条线分析：

- 线路 A（Self Memory）：读取 [references/self_analyzer.md](references/self_analyzer.md)，提取个人经历、核心价值观、生活习惯、重要记忆、人际关系图谱、成长轨迹
- 线路 B（Persona）：读取 [references/persona_analyzer.md](references/persona_analyzer.md)，按标签翻译表把用户标签转成具体行为规则，提取说话风格、情感模式、决策模式、人际行为

### Step 4 生成与预览

- 按 [references/self_builder.md](references/self_builder.md) 的模板生成 self.md 内容
- 按 [references/persona_builder.md](references/persona_builder.md) 的 5 层结构生成 persona.md 内容
- 向用户展示两份摘要（各 5-8 行），确认或调整后再写入

### Step 5 写入文件（生成 SKILL.md）

用户确认后，优先用脚本一键创建。先把 meta.json 写入临时文件 `$env:TEMP\{slug}_meta.json`，self.md 内容写入 `$env:TEMP\{slug}_self.md`，persona.md 内容写入 `$env:TEMP\{slug}_persona.md`（用 Write 工具即可），然后执行：

```powershell
python .trae/skills/create-yourself/scripts/skill_writer.py --action create --slug {slug} --base-dir ./.trae/skills --meta $env:TEMP\{slug}_meta.json --self $env:TEMP\{slug}_self.md --persona $env:TEMP\{slug}_persona.md
```

meta.json 内容：

```json
{
  "name": "{name}",
  "slug": "{slug}",
  "created_at": "{ISO时间}",
  "updated_at": "{ISO时间}",
  "version": "v1",
  "profile": { "age": "...", "occupation": "...", "city": "...", "gender": "...", "mbti": "...", "zodiac": "..." },
  "tags": { "personality": [], "lifestyle": [] },
  "impression": "{impression}",
  "memory_sources": [],
  "corrections_count": 0
}
```

脚本失败时的 fallback：用 Write 工具直接把 `self.md`、`persona.md`、`meta.json` 写入 `<repo>/.trae/skills/{slug}/`，然后执行 `python .../skill_writer.py --action combine --slug {slug} --base-dir ./.trae/skills` 生成 SKILL.md；combine 也失败就手动按 combine 模板写入 SKILL.md。

成功后告知用户：

```
✅ 自我 Skill 已创建！
位置：.trae/skills/{slug}/
调用：在 Trae 中通过 Skill 名称 {slug} 与你对话
如果哪里不像你，直接说"我不会这样"，我来更新。
```

## 进化模式：追加材料

用户提供新的聊天记录、照片或笔记时：

1. 按 Step 2 的方式读取新内容
2. 用 Read 读取现有 `./.trae/skills/{slug}/self.md` 和 `persona.md`
3. 读取 [references/merger.md](references/merger.md)，按增量不覆盖原则分析
4. 先备份当前版本：

```powershell
python .trae/skills/create-yourself/scripts/version_manager.py --action backup --slug {slug} --base-dir ./.trae/skills
```

5. 用 Edit 工具把增量追加到 self.md / persona.md 对应章节
6. 重新生成 SKILL.md：

```powershell
python .trae/skills/create-yourself/scripts/skill_writer.py --action combine --slug {slug} --base-dir ./.trae/skills
```

7. 用 Edit 更新 meta.json 的 version 和 updated_at

## 进化模式：对话纠正

用户说"不对""我不会这样说""我应该是""这不像我"时：

1. 读取 [references/correction_handler.md](references/correction_handler.md) 识别纠正内容并分类（Self Memory 事实类 / Persona 性格类）
2. 向用户复述确认理解，生成 Correction 记录
3. 用 Edit 追加到对应文件的 `## Correction 记录` 节，同时修改被纠正的原文并标注 `[已纠正，见 Correction #N]`
4. 重新执行 combine 生成 SKILL.md，纠正立即生效

## 管理操作

- 列出所有自我 Skill：

```powershell
python .trae/skills/create-yourself/scripts/skill_writer.py --action list --base-dir ./.trae/skills
```

- 查看历史版本 / 回滚：

```powershell
python .trae/skills/create-yourself/scripts/version_manager.py --action list --slug {slug} --base-dir ./.trae/skills
python .trae/skills/create-yourself/scripts/version_manager.py --action rollback --slug {slug} --version {version} --base-dir ./.trae/skills
```

- 删除（必须先向用户确认）：

```powershell
Remove-Item -Recurse -Force .\.trae\skills\{slug}
```

## 注意事项

- 全程使用与用户一致的语言（默认中文）
- 只有依据才写，没有依据就标注"原材料不足"或 `[待补充]`，严禁虚构经历
- 不做心理学诊断，不添加用户没表达过的价值观
- 解析脚本支持 Windows 路径与中文文件名；若输出乱码，先 `$env:PYTHONIOENCODING='utf-8'` 再重跑