---
name: autocut-workflow
description: 用户要求按 AutoCut 工作流剪片（文案→剪映草稿）、提到 autocut_* 工具，或要"用本地/在线素材剪一条片并导出剪映"时使用。按 11 阶段执行，文案只读、候选由用户拍板、只新建草稿，每阶段先读 .autocut/state.json。
---

# AutoCut 工作流：文案 → 剪映草稿

## 依赖

- **`dsh-autocut` 插件**（20 个 `autocut_*` 工具，>= 0.3.0）。本技能只是流程说明，不含插件；工具不存在时先装插件并完全重启 DSH。
- **剪映导出**：Windows + 剪映专业版 11.x + `videoeditor.dll`（草稿加密）。
- **Python 3.10+**：`faster-whisper`，以及 `aoguai/pyJianYingDraft` fork（PyPI 官方版不支持加密草稿）。
- **FFmpeg**：`ffmpeg` + `ffprobe` 真实二进制。

## 开工前检查（缺一就停，不要边修边剪）

1. **工具**：`autocut_*` 可用。没有 → 插件没挂上；改完 profile 必须完全退出 DSH（含托盘）再开。
2. **Python**：取 `PYTHON_PATH`（或 DSH 运行时 Python）。验证：
   ```text
   python -c "import pyJianYingDraft as d; print([n for n in dir(d) if 'Crypto' in n])"
   ```
   必须看到 `JianyingDraftCryptoCodec` 和 `DraftCryptoConfig`。
3. **FFmpeg**：`FFMPEG_PATH` / `FFPROBE_PATH` 指向真实文件（不是 0 字节占位），能读出时长。
4. **剪映**：`JY_INSTALL_DIR` 指向含 `videoeditor.dll` 的安装目录；草稿目录默认
   `%LOCALAPPDATA%\JianyingPro\User Data\Projects\com.lveditor.draft`。
5. **密钥**：`DEEPSEEK_API_KEY` 必需；`PEXELS_API_KEY` / `PIXABAY_API_KEY` / `JAMENDO_CLIENT_ID` 可选。只打印长度与末四位，不打印完整值。

任一项缺失：报告具体报错与判断，**不许**退回明文草稿、跳过验证或假装成功。

## 分工

用户负责文案、内容判断，并最终拍板：选哪条素材、哪首配乐、审片是否通过、草稿是否可用。执行方只做剪辑执行：不改写、不续写、不合并文案；自动匹配只做初筛，候选摆到用户面前。

## 五条硬规则

1. **不许静默降级**：依赖/二进制/格式不兼容，停下报错，不许用明文凑。
2. **不许假验证**：渲染必须 ffprobe 读得出时长；草稿必须 codec 解码看到轨道；最终必须用户在剪映里亲眼确认。
3. **单一写入者**：动手前读 `.autocut/state.json`，阶段顺序不对就停；同一阶段只跑一次，不并行写。
4. **文案与录音只读**：原始文案/录音不修改、不改写、不续写；处理只写 `.autocut/`。
5. **不碰既有草稿**：只新建独立草稿；禁止修改/删除/移动任何已存在草稿与 `root_meta_info.json` 既有条目。

## 11 阶段

| 阶段 | 调用 | 关键参数 | 产出 | 停点 |
|---|---|---|---|---|
| 0 建项目 | `autocut_init` → `autocut_analyze` | root, script, narration, assets[], music[], width/height/fps | `.autocut/script_graph.json` | 报告段数/字符数 |
| 1 在线素材 | `autocut_research`（可 + `autocut_import_research`） | per_segment | `online_materials.json` | — |
| 2 配乐 | `autocut_research_music` | — | `music_candidates.json` | — |
| 3 人工挑选 | `autocut_review` | — | `review.html` → `selections.json` | ★停 |
| 4 入库 | `autocut_prepare_selections` | download=true | 本地素材/覆盖 | 报告失败项 |
| 5 人声 | `autocut_transcribe` → `autocut_voice`；无录音用 `autocut_estimate_timing` | model | `clean_voice.wav` | 声明精度 |
| 6 索引 | `autocut_index` | directories | `asset_catalog` | — |
| 7 匹配 | `autocut_match` | top_k | `asset_matches` | — |
| 8 声音 | `autocut_sound` | generate | `sound_plan.json` | — |
| 9 时间线 | `autocut_timeline` | — | `timeline.json` + `timeline.otio` | 核对轨道 |
| 10 审片 | `autocut_preview` | subtitles | `preview.mp4` | ★停 |
| 11 交付 | `autocut_export_jianying` | draft_dir, name | 加密草稿 | ★用户确认 |

### 参数以工具 schema 为准

- `autocut_init` 的旁白参数名是 **`narration`**，不是 `voice`；没有 `style` 参数，镜头/停顿参数用默认，需要时改 `.autocut/project.json` 后重跑受影响阶段。
- `autocut_export_jianying` 必须显式给 `draft_dir`，草稿名会自动追加时间戳，不覆盖同名。
- `autocut_import_research` 的 `candidates` 必须嵌在 `segments[]` 内部，放顶层会读不到。
- 素材/音乐目录传数组；没有就省略。

## 各阶段口径

### 0 建项目

- `root` 用干净新目录，不复用旧项目。
- 文案先落盘为 UTF-8 `.txt`/`.md`，原样不改。
- analyze 后检查：段数、段号范围、首句、**切分前后字符数一致**（不一致立即停），以及识别出的重点词/实体。

### 1 在线素材

- 只认具体素材页 URL；搜索结果页/首页/分类页会被拒收，不要放宽。
- 版权不确定写 `license: "未确认"`，不写“可商用”。
- 网页参考只登记链接，不下载。
- 零候选的段必须点名，这是补镜头/找本地素材的依据。

### 3 审阅

- 生成 `.autocut/review.html`，让用户在浏览器里勾选并导出 `selections.json` 放回 `.autocut/`。
- 用户没放回之前不要进入阶段 4。

### 5 人声

- 有录音：transcribe（字级时间码）→ voice（压停顿/去喘气/响度）。
- 无录音：estimate_timing，并在每次报告里注明“时间为估算”。
- 核对 `clean_voice.wav` 时长与时间线总时长，差 > 0.3 秒必须解释。

### 9/10 时间线与审片

- 报告：总时长、每轨段数/首末点、未匹配段号、人声与 BGM 是否抢拍。
- 渲染后用 ffprobe 核对分辨率/帧率/音轨/时长；抽帧看字幕与画面。
- 用户确认审片后才导出草稿。

### 11 导出

- `draft_dir` 用本机真实草稿目录；只新建，不覆盖。
- 返回 `encrypted: true` 才算加密生效；用 codec 解码确认轨道数与总时长。
- 让用户在剪映里确认：列表能看到、打开轨道齐全、画面与旁白对得上。确认前不宣称交付。

## 局部重剪

- 换镜头：`autocut_override(action=asset, id=段号, value=路径)` + 锁定 → 只重跑 timeline、preview。
- 换配乐：改 `sound_plan.json` 或重做 selections → sound → timeline → preview。
- 只调人声：voice → timeline → preview。
- 改文案：唯一需要回到阶段 0 的改动；段号会错位、旧勾选失效，先告知用户再动。
- 每次重跑前读 `state.json`；对比新旧 timeline 的总时长/段数，变化要解释。人工锁定优先，不得覆盖。

## 验证与报告

每阶段收尾一句话：做了什么 / 产物路径 / 怎么验证 / 残留风险。
零候选、时间码差异、依赖失败必须明说。禁止用“文件已生成”当成功。

## 排错

1. 工具不存在 → 插件没挂上；改 profile 后必须完全退出 DSH 再开。
2. 找不到 ffmpeg/ffprobe → 检查 `FFMPEG_PATH`/`FFPROBE_PATH` 指向真实二进制。
3. `ModuleNotFoundError` → 装到 DSH 运行时 Python，不是 shell 的 python。
4. 草稿列表看不到 → 查草稿文件夹、`root_meta_info.json` 登记、刷新列表。
5. 草稿打不开/空白 → 检查 `encrypted`，依赖是否被换回官方版。
6. 时间线对不上旁白 → 查 alignment 最后一个词与 `clean_voice.wav` 时长。
7. `import_research` 拒收 → 给具体页面 URL，不要放宽校验。
8. 下载全失败 → 查 Key 是否生效/重启、站点可达；网络问题就说网络问题。
