# 🎵 Targeted Musical Content Generator

[中文](#-定向音乐内容生成器) | [English](#-targeted-musical-content-generator)

---

## 🇬🇧 Targeted Musical Content Generator

An AI-powered tool that generates **personalized music exercise scores** for Western classical instrument students, directly targeting their specific technical or sight-reading weaknesses described in natural language by the teacher.

**Try it online**: [ModelScope Space](https://www.modelscope.cn/studios/JeffreyZhou2026/TargetedMusicalContentGenerator-15/)https://www.modelscope.cn/studios/JeffreyZhou2026/TargetedMusicalContentGenerator-15/
Powered by [ChatMusician](https://huggingface.co/m-a-p/ChatMusician) — an open-source music LLM that natively generates ABC notation.

### How It Works

1. **Teacher describes the weakness** — e.g. *"Violin beginner, string crossing clumsy and rough, unwanted open string sounds"*
2. **AI generates a targeted exercise** — ChatMusician produces ABC notation tailored to the described weakness
3. **Score is validated & exported** — The output is parsed, validated, and exported as `.mxl` / `.musicxml` (openable in [MuseScore](https://musescore.org/)), `.abc`, and a `report.md`

### Key Features

- **9 Western classical instruments**: Piano, Violin, Viola, Cello, Flute, Clarinet, Trumpet, Horn, Percussion
- **3 difficulty levels**: Beginner (0–1 sharps/flats), Intermediate (0–3), Advanced (unrestricted keys)
- **95-entry music pedagogy knowledge base** with 2,352+ multilingual keywords and 5-tier fuzzy matching (substring → bag-of-words → fuzzy bag-of-words → fuzzy single-word → stem-substring)
- **18 quick-prompt buttons** with 54 pre-written template scores (18 weaknesses × 3 levels)
- **AI-only delivery** — User natural language prompts always go through ChatMusician; no preset fallback templates are substituted to fake results
- **Level-aware key constraints** — Beginners only get C/G/F major and Am/Em/Dm minor; no absurd keys like F♯ major for beginners
- **Bounded AI repair** — Up to 3 repair passes if the initial output fails parsing; honest failure reporting if all repairs fail
- **Bilingual UI** — English and Chinese interface; teacher input accepts both languages
- **Output formats**: `.mxl`, `.musicxml`, `.abc`, `report.md`
- **CPU-deployable** — Runs on 2 vCPU + 16 GB RAM via GGUF Q4_K_M quantization (~4.1 GB model)

### Architecture

```
Teacher natural language or quick-prompt button
  ├─ Quick button → Match pre-written template score → Use directly
  └─ Custom input → build_direct_prompt() → ChatMusician.generate_streaming()
                    → _extract_abc() / parse_exercise_abc()
                    → validate_generated_score() [only blocks empty/all-rest scores]
                    → _enforce_key_constraint() [level-based key compliance]
                    → On failure: build_repair_prompt() + up to 3 AI repairs
                    → On success: export .mxl / .musicxml / .abc / report.md
                    → If all repairs fail: return real AI attempt file + low confidence report
```

### Quick Start

#### Local (CPU + GGUF Q4_K_M)

```bash
pip install -r requirements.txt
python app.py
```

First launch downloads ChatMusician-GGUF (~4.1 GB) and takes 5–10 minutes. Subsequent launches reuse the cached model.

#### ModelScope Space

1. Create a Space with mirror: `ubuntu22.04-py311-torch2.9.1-modelscope1.35.0`
2. Upload this repository
3. Ensure deployment uses this repository's `requirements.txt`
4. On startup, `app.py` will auto-heal a missing `llama-cpp-python` in hosted environments

### System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| RAM | 12 GB | 16 GB |
| CPU | 2 vCPU | 4 vCPU |
| Disk | 15 GB | 20 GB |
| GPU | Not required | Optional |

**Memory budget (16 GB RAM, CPU + GGUF Q4_K_M)**:
- ChatMusician-GGUF Q4_K_M: ~7–8 GB
- Python + Gradio runtime: ~1.5 GB
- music21 + other deps: ~0.5 GB
- Inference temporary tensors: ~0.5 GB
- **Total: ~9.5–10.5 GB, leaving ~5.5–6.5 GB free**

### Project Structure

```
TargetedMusicalContentGenerator-ModelScope/
├── app.py                          # Gradio frontend & main generation pipeline
├── backend/
│   ├── __init__.py
│   ├── exercise_generator.py       # Core: prompt building, ABC extraction, validation, 5-tier keyword matching
│   ├── llm_engine.py              # GGUF Q4_K_M model loading + streaming inference
│   ├── music_knowledge.py         # 95-entry knowledge base + 5-tier search + 2,352 keywords
│   ├── output_exporter.py         # Export .mxl/.musicxml/.abc
│   ├── template_scores.py         # 54 pre-written ABC template scores
│   ├── weakness_parser.py         # Weakness parsing (legacy, retained)
│   ├── i18n.py                    # Internationalization module
│   └── json_validator.py          # JSON validation (legacy, retained)
├── locales/
│   ├── en.json                    # English UI translations
│   └── zh.json                    # Chinese UI translations
├── prompts/
│   ├── exercise_generator.txt     # Prompt template (reference, not directly loaded)
│   ├── exercise_fewshot_piano.json
│   ├── exercise_fewshot_violin.json
│   └── weakness_parser.txt
├── Dockerfile                     # ModelScope Space deployment
├── preload_assets.py              # Model preloading script
├── requirements.txt
├── start.sh                       # Space entrypoint
└── README.md
```

### Key Design Decisions

1. **Direct ABC generation** — ChatMusician outputs ABC notation natively; `music21` parses it into a Score object.
2. **Single-voice for all instruments** — Piano generates single-hand exercises (V:1 treble or bass); strings/winds/percussion use single staff. Grand staff generation is not supported.
3. **Level-aware key constraints** — Beginner exercises are restricted to 0–1 sharps/flats (C/G/F/Am/Em/Dm); intermediate to 0–3; advanced has no restriction. Non-compliant keys are auto-replaced by the closest permitted key via circle-of-fifths mapping.
4. **AI-only delivery** — The app never substitutes a preset fallback score for user natural language prompts. If AI generation fails after all repair attempts, it returns the real AI attempt file with an honest low-confidence report.
5. **Bounded repair loop** — Up to 3 AI repair passes; transparent failure reporting if all repairs fail.
6. **5-tier fuzzy keyword matching** — Handles typos, word-order variations, morphological changes, and prefix truncation across 2,352+ multilingual keywords.
7. **Deployment-first packaging** — `Dockerfile` + `preload_assets.py` preload the GGUF model so first-click latency is reduced.
8. **Bilingual knowledge retrieval** — Chinese and English teacher descriptions map to the same musical concepts and validator targets via `_TEXT_ALIASES` (50+ Chinese→English term mappings).

### Model Details

| Property | Value |
|----------|-------|
| Base model | ChatMusician (LLaMA2-7B, continued pre-training on ABC notation) |
| Quantization | GGUF Q4_K_M (~4.1 GB) |
| Inference engine | `llama-cpp-python` (CPU-optimized) |
| Context window | 2,048 tokens |
| Max generation tokens | 2,048 |
| Temperature | 0.5 (direct), 0.3–0.5 (repair, decreasing per attempt) |
| Prompt format | LLaMA2 chat: `Human: {content} </s> Assistant: ` |
| Model source | [ModelScope](https://www.modelscope.cn/models/JeffreyZhou2026/ChatMusician-GGUF) (primary) / [HuggingFace](https://huggingface.co/MaziyarPanahi/ChatMusician-GGUF) (fallback) |

### Supported Instruments & Quick Prompts

| Family | Instruments | Quick Prompts |
|--------|------------|---------------|
| 🎹 Piano/Keyboard | Piano | Finger independence, Thumb crossing, Octave reach, Chord voicing, Arpeggio, Dotted rhythm |
| 🎻 Strings | Violin, Viola, Cello | Intonation, Bowing, Vibrato, Shifting, String crossing, Dotted rhythm |
| 🌬️ Woodwinds | Flute, Clarinet | Breath control, Tonguing/articulation, Fingering, Dynamics |
| 🎺 Brass | Trumpet, Horn | Embouchure, Articulation |
| 🥁 Percussion | Percussion | Stick control, Rhythm accuracy |

Each quick prompt has 3 pre-written template scores (beginner/intermediate/advanced).

### Generation Pipeline Details

| Stage | Progress | Description |
|-------|----------|-------------|
| Prompt building | 0–3% | Assembles compact prompt with instrument, level, knowledge-base hints, and ABC pattern example |
| AI generation | 3–61% | ChatMusician streaming inference with real-time ETA updates |
| ABC extraction | 61–66% | Extracts and cleans ABC notation from LLM output |
| Validation | 66–71% | Checks for empty/all-rest scores, structure compliance |
| Repair (if needed) | 71–85% | Up to 3 targeted AI repair passes |
| Key constraint enforcement | — | Auto-replaces non-compliant keys based on level |
| Score building | 85–92% | Parses ABC into music21 Score |
| File export | 92–98% | Exports .mxl, .musicxml, .abc, report.md |
| Complete | 98–100% | Exercise ready for download |

### License

This project uses [ChatMusician](https://huggingface.co/m-a-p/ChatMusician) (LLaMA2-based, open-source). Check the model license at https://huggingface.co/m-a-p/ChatMusician

### Author

**Jeffrey Zhou**



## ⚠️ Copyright Notice

© 2026 Jeffrey Zhou. All rights reserved.

This repository and its contents are protected by copyright law.  
No part of this project may be copied, reproduced, modified, or distributed without prior written permission from the author.

Commercial use is strictly prohibited.


*Built with ❤️ for music education*


---

## 🇨🇳 定向音乐内容生成器

一个基于 AI 的工具，针对西方古典乐器学习者的**演奏技术弱点或识谱视奏弱点**，生成个性化音乐练习乐谱。教师只需用自然语言描述学生的问题，系统即可产出可直接在 [MuseScore](https://musescore.org/) 中打开的乐谱文件。

**在线试用**：[ModelScope Space](https://www.modelscope.cn/studios/JeffreyZhou2026/TargetedMusicalContentGenerator-15/)

基于 [ChatMusician](https://huggingface.co/m-a-p/ChatMusician) —— 一个原生支持 ABC 记谱法生成的开源音乐大语言模型。

### 工作原理

1. **教师描述弱点** — 如 *"小提琴初学者，换弦笨拙粗鲁，碰响空弦"*
2. **AI 生成针对性练习** — ChatMusician 生成与弱点匹配的 ABC 记谱
3. **校验并导出乐谱** — 输出经解析、校验后导出为 `.mxl` / `.musicxml`（MuseScore 可直接打开）、`.abc` 及 `report.md`

### 核心特性

- **9 种西方古典乐器**：钢琴、小提琴、中提琴、大提琴、长笛、单簧管、小号、圆号、打击乐
- **3 个难度级别**：Beginner（0-1个升降号）、Intermediate（0-3个）、Advanced（不限调号）
- **95 条音乐教学知识库**，含 2,352+ 多语言关键词和 5 层模糊匹配（子串→词袋→模糊词袋→模糊单词→词干子串）
- **18 个快捷按钮** + 54 个预写模板乐谱（18个弱点 × 3个级别）
- **纯 AI 交付** — 用户自然语言输入必须经过 ChatMusician 生成，绝不用固定模板冒充 AI 结果
- **级别感知调号约束** — 初学者仅限 C/G/F 大调和 Am/Em/Dm 小调，不会出现 F♯ 大调这类荒唐调号
- **有限 AI 修复** — 首次输出解析失败时最多 3 次 repair；全部失败则诚实报告
- **双语界面** — 英文/中文 UI 切换；教师输入支持中英双语
- **输出格式**：`.mxl`、`.musicxml`、`.abc`、`report.md`
- **CPU 可部署** — 通过 GGUF Q4_K_M 量化（~4.1 GB 模型），在 2 vCPU + 16 GB RAM 环境即可运行

### 架构

```
教师自然语言 或 快捷按钮
  ├─ 快捷按钮 → 匹配预写模板乐谱 → 直接使用
  └─ 自定义输入 → build_direct_prompt() → ChatMusician.generate_streaming()
                    → _extract_abc() / parse_exercise_abc()
                    → validate_generated_score() [仅阻止空谱和全休止符]
                    → _enforce_key_constraint() [级别感知调号合规]
                    → 失败则 build_repair_prompt() + 最多3次AI repair
                    → 通过后 export .mxl / .musicxml / .abc / report.md
                    → 若最终仍失败，返回真实AI尝试文件与低置信度说明
```

### 快速开始

#### 本地运行（CPU + GGUF Q4_K_M）

```bash
pip install -r requirements.txt
python app.py
```

首次启动需下载 ChatMusician-GGUF（~4.1 GB），耗时 5–10 分钟。后续启动复用缓存模型。

#### ModelScope Space 部署

1. 创建 Space，选择镜像：`ubuntu22.04-py311-torch2.9.1-modelscope1.35.0`
2. 上传本仓库
3. 确保部署使用本仓库的 `requirements.txt`
4. 启动时 `app.py` 会自动补装托管环境中缺失的 `llama-cpp-python`

### 系统需求

| 组件 | 最低 | 推荐 |
|------|------|------|
| 内存 | 12 GB | 16 GB |
| CPU | 2 核 | 4 核 |
| 磁盘 | 15 GB | 20 GB |
| GPU | 不需要 | 可选 |

**内存预算（16 GB RAM，CPU + GGUF Q4_K_M）**：
- ChatMusician-GGUF Q4_K_M：~7–8 GB
- Python + Gradio 运行时：~1.5 GB
- music21 及其他依赖：~0.5 GB
- 推理临时张量：~0.5 GB
- **总计：~9.5–10.5 GB，剩余 ~5.5–6.5 GB 足够**

### 项目结构

```
TargetedMusicalContentGenerator-ModelScope/
├── app.py                          # Gradio 前端与主生成流水线
├── backend/
│   ├── __init__.py
│   ├── exercise_generator.py       # 核心：Prompt构建、ABC提取、校验、5层关键词匹配
│   ├── llm_engine.py              # GGUF Q4_K_M 模型加载 + 流式推理
│   ├── music_knowledge.py         # 95条知识库 + 5层搜索 + 2352关键词
│   ├── output_exporter.py         # 导出 .mxl/.musicxml/.abc
│   ├── template_scores.py         # 54个预写ABC模板乐谱
│   ├── weakness_parser.py         # 弱点解析（旧版保留）
│   ├── i18n.py                    # 国际化模块
│   └── json_validator.py          # JSON校验（旧版保留）
├── locales/
│   ├── en.json                    # 英文界面翻译
│   └── zh.json                    # 中文界面翻译
├── prompts/
│   ├── exercise_generator.txt     # Prompt模板（参考用，非直接加载）
│   ├── exercise_fewshot_piano.json
│   ├── exercise_fewshot_violin.json
│   └── weakness_parser.txt
├── Dockerfile                     # ModelScope Space 部署配置
├── preload_assets.py              # 模型预加载脚本
├── requirements.txt
├── start.sh                       # Space 启动脚本
└── README.md
```

### 核心设计决策

1. **直接 ABC 生成** — ChatMusician 原生输出 ABC 记谱法，`music21` 直接解析为 Score 对象
2. **所有乐器单声部** — 钢琴仅生成单手练习（V:1 treble 或 bass）；弦乐/管乐/打击乐使用单行谱。不支持大谱表生成
3. **级别感知调号约束** — 初学者练习限制在 0-1 个升降号（C/G/F/Am/Em/Dm）；中级限制在 0-3 个；高级不限。不合规调号自动替换为五度圈最近合规调
4. **纯 AI 交付** — 用户自然语言输入绝不使用预设模板替代 AI 结果。若 AI 多轮修复后仍失败，返回真实 AI 尝试文件与诚实低置信度报告
5. **有限 AI 修复** — 最多 3 次 AI repair；全部失败则透明报告
6. **5 层模糊关键词匹配** — 覆盖拼写错误、词序颠倒、词性变化、前缀截断等场景，2,352+ 多语言关键词
7. **部署优先打包** — `Dockerfile` + `preload_assets.py` 预加载 GGUF 模型，降低首次点击延迟
8. **双语知识检索** — 中文和英文教师描述通过 `_TEXT_ALIASES`（50+ 中文→英文术语映射）映射到相同的音乐概念和校验目标

### 模型详情

| 属性 | 值 |
|------|---|
| 基座模型 | ChatMusician（LLaMA2-7B，ABC 记谱法持续预训练） |
| 量化方式 | GGUF Q4_K_M（~4.1 GB） |
| 推理引擎 | `llama-cpp-python`（CPU 优化） |
| 上下文窗口 | 2,048 tokens |
| 最大生成 tokens | 2,048 |
| 温度 | 0.5（直接生成），0.3–0.5（修复，逐轮递减） |
| Prompt 格式 | LLaMA2 chat: `Human: {content} </s> Assistant: ` |
| 模型来源 | [ModelScope](https://www.modelscope.cn/models/JeffreyZhou2026/ChatMusician-GGUF)（主源）/ [HuggingFace](https://huggingface.co/MaziyarPanahi/ChatMusician-GGUF)（备选） |

### 支持乐器与快捷按钮

| 乐器族 | 乐器 | 快捷按钮 |
|--------|------|---------|
| 🎹 钢琴/键盘 | 钢琴 | 手指独立、拇指穿指、八度伸展、和弦声部、琶音、附点节奏 |
| 🎻 弦乐 | 小提琴、中提琴、大提琴 | 音准、运弓、揉弦、换把、换弦、附点节奏 |
| 🌬️ 木管 | 长笛、单簧管 | 气息控制、吐音连音、指法、力度 |
| 🎺 铜管 | 小号、圆号 | 口型、吐音 |
| 🥁 打击乐 | 打击乐 | 鼓棒控制、节奏准确性 |

每个快捷按钮有 3 个预写模板乐谱（beginner/intermediate/advanced）。

### 生成流水线详情

| 阶段 | 进度 | 描述 |
|------|------|------|
| 构建Prompt | 0–3% | 组装紧凑prompt，含乐器、级别、知识库提示、ABC模式示例 |
| AI生成 | 3–61% | ChatMusician 流式推理，实时 ETA 更新 |
| ABC提取 | 61–66% | 从LLM输出中提取并清洗ABC记谱 |
| 校验 | 66–71% | 检查空谱/全休止符、结构合规性 |
| 修复（如需） | 71–85% | 最多3次定向AI修复 |
| 调号约束执行 | — | 根据级别自动替换不合规调号 |
| 构建Score | 85–92% | 将ABC解析为music21 Score |
| 导出文件 | 92–98% | 导出 .mxl、.musicxml、.abc、report.md |
| 完成 | 98–100% | 练习已就绪，可下载 |

### 许可证

本项目使用 [ChatMusician](https://huggingface.co/m-a-p/ChatMusician)（基于 LLaMA2，开源）。请查看模型许可证：https://huggingface.co/m-a-p/ChatMusician

### 作者

**Jeffrey Zhou**



## License

MIT License - See [LICENSE](../LICENSE) for details.

---

## ⚠️ Copyright Notice

© 2026 Jeffrey Zhou. All rights reserved.

This repository and its contents are protected by copyright law. No part of this project may be copied, reproduced, modified, or distributed without prior written permission from the author.

**Commercial use is strictly prohibited.**


*Built with ❤️ for music education* ```



