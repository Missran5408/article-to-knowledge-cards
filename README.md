# 文献翻译 · literature-translation

一个开源 Agent Skill：把英文文献**逐自然段**译成中文，每段附一张四字段知识卡片，并用术语表保证译名在学科内正确。

- **输入**：直接粘贴的英文正文，或本地 PDF 文件路径
- **输出**：术语表 + 每段「原文摘录 / 译文 / 知识卡片」
- **知识卡片四字段**：标题、核心知识、简明解释、例子或自测问题
- **适配**：任何支持 `SKILL.md` 的 Agent 运行时（Claude Skills、Codex、DSH 等）

## 目录结构

```
literature-translation/
├── SKILL.md          # 技能本体：工作流、分段规则、译文规则、卡片规则、输出模板、自检清单
└── agents/
    └── openai.yaml   # UI 元数据（display_name / short_description / default_prompt）
```

技能内不含脚本，因此没有需要维护的可执行文件；PDF 提取用运行时现成的 `pdftotext` 或 `pypdf` 一行命令完成。

## 安装

把 `literature-translation/` 整个文件夹放进运行时的技能目录：

| 范围 | 放置位置 |
|---|---|
| 项目级 | `<项目根>/.agents/skills/literature-translation/` |
| 用户级 | `~/.agents/skills/literature-translation/`（Windows 为 `C:\Users\<用户名>\.agents\skills\`） |
| DSH | 上述任一位置，或 `<DSH_HOME>/skills/literature-translation/` |

放好后多数运行时会自动发现新技能（无需重启）；若目录未刷新，重开一次会话即可。

想在 Windows 上保持「仓库里改、技能目录立刻生效」，可以建一个目录联接代替复制：

```powershell
cmd /c mklink /J "%USERPROFILE%\.agents\skills\literature-translation" "D:\path\to\literature-translation"
```

## 使用方法

直接用自然语言提出请求即可，常见触发说法：

- 翻译这篇文献 / 把这段英文论文译成中文 / 逐段英译汉 / 精读翻译
- 翻译这个 PDF：`D:\papers\robotics.pdf`
- translate this paper into Chinese, paragraph by paragraph

技能被触发后会自动执行：识别输入 → 判定学科与文体 → 建术语表 → 逐段出卡片 → 交付前自检。

可选的附加要求（技能会遵从）：`只要译文不要卡片`、`给大一学生看的解释`、`输出成 Markdown 文件`、`只翻译第 2–3 节`。

## 输入输出示例

### 示例 1：粘贴一段英文文本

**输入**

> Chimeric antigen receptor (CAR) T-cell therapy has transformed the treatment of relapsed or refractory B-cell malignancies, yet durable responses are observed in only a subset of patients. The mechanisms that govern whether infused T cells persist and remain functional in the tumour microenvironment remain incompletely understood.

**输出**（节选自实际运行结果）

**学科**：肿瘤免疫学 / 细胞治疗 ｜ **文体**：研究论文

| 原文 | 译名 | 依据 |
|---|---|---|
| chimeric antigen receptor (CAR) T-cell therapy | 嵌合抗原受体（CAR）T 细胞治疗 | 免疫学名词审定名词 |
| relapsed or refractory | 复发或难治性 | 肿瘤学通行译名 |
| tumour microenvironment | 肿瘤微环境 | 肿瘤免疫学通行译名 |

> **原文摘录**
> Chimeric antigen receptor (CAR) T-cell therapy has transformed the treatment of relapsed or refractory B-cell malignancies, yet durable responses are observed in only a subset of patients. …

**译文**

嵌合抗原受体（CAR）T 细胞治疗已经改变了复发或难治性 B 细胞恶性肿瘤的治疗格局，然而仅在部分患者中可观察到持久缓解。决定输注后的 T 细胞能否在肿瘤微环境中持续存在并保持功能的机制，目前仍未完全阐明。

**知识卡片**

- **标题**：CAR T 疗法的持久缓解难题
- **核心知识**：CAR T 治疗显著改善复发或难治性 B 细胞恶性肿瘤疗效，但持久缓解只发生于部分患者；输注 T 细胞能否在肿瘤微环境中存活并维持功能，其机制尚不清楚。
- **简明解释**：CAR T 是取出患者 T 细胞、用能识别肿瘤抗原（如 CD19）的嵌合抗原受体改造后回输。疗效关键不只是「杀伤一次」，而是回输细胞能否长期存活、持续增殖并保持杀伤力；肿瘤微环境的抑制因素会削弱该过程，故患者结局差异大。段末「机制不清」正是后文用单细胞测序切入的动机。
- **例子或自测问题**：自测题——若患者输注后 1 个月肿瘤明显缩小、6 个月后复发，按本段逻辑最可能的失效环节是什么？（答案要点：不是初始杀伤力不足，而是输注细胞未能持久存在或功能维持失败。）

### 示例 2：本地 PDF

**输入**

```
帮我翻译这个 PDF：D:\papers\robotics.pdf
```

**输出**

1. 自动提取文本（优先 `pdftotext -layout`，其次 `pypdf`），合并 PDF 硬换行、剔除页眉页脚与行号、恢复段落边界；
2. 判定学科（如 流体力学 / 仿生机器人）并给出术语表（Reynolds number → 雷诺数、Strouhal number → 斯特劳哈尔数…）；
3. 按自然段逐段输出「原文摘录 + 译文 + 知识卡片」，公式、变量、单位、引用编号原样保留；
4. 若该 PDF 没有文本层（扫描件或图片型），会明确告知无法可靠提取，并给出三条出路——先做 OCR、直接粘贴正文、或提供页面截图，**不会编造原文**。

## 设计要点

- **先定领域再动笔**：学科判定决定术语选择，同一篇文献术语全篇一致；译名存疑时给候选并加「译者注」，不生造术语。
- **学科内正确**：内置跨学科高频易错术语表（`significant`、`control`、`precision/accuracy/recall`、`stress/strain`、`cable-driven`、`torque/moment`、`compliance`、`turbulence`、`power` 等），并规定公式、变量、单位、基因名、算法专名、引用与图表编号原样保留；统计量不得添加原文没有的方向词，不得把「相关」写成「因果」。
- **分段可复现**：以自然段为单位，章节标题只作分组、列表整体成卡、参考文献默认跳过；PDF 无空行时按版面与语义信号恢复段界。
- **卡片可检验**：四个字段必须齐全且紧扣该段，「例子或自测问题」必须能验证本段知识，禁止「请思考……」式空问题。
- **诚实边界**：输入已是中文、扫描版 PDF、领域无法判定等情况都有明确处置，宁可停下确认也不臆造内容。

## 测试情况

开发时用四类真实输入跑过端到端测试，并根据结果迭代了 SKILL.md：

| 用例 | 学科 | 覆盖的规则分支 |
|---|---|---|
| 粘贴摘要 | 肿瘤免疫学 / CAR T | 术语表、卡片字段、统计量（HR / 置信区间）忠实性 |
| 粘贴摘要 | 深度学习 / 神经机器翻译 | 括号附注密度、符号还原（`q.k`→`q·k`、`dk`→`d_k`） |
| 文本型 PDF | 流体力学 / 仿生机器鱼 | PDF 提取、段界恢复（提取结果无空行）、失真的希腊字母与上标还原 |
| 边界输入 | — | 输入已是中文、无文本层的扫描版 PDF、只要译文不要卡片 |

另外用官方 `quick_validate.py` 校验通过。测试中暴露并已修复的问题包括：PDF 提取丢失段界、卡片把「相关」写成「因果」、统计量方向被反推、假设值被当成原文数据、括号附注成灾等。

## 校验

技能结构与 frontmatter 用官方 `skill-creator` 的校验脚本检查：

```bash
python quick_validate.py literature-translation
# Skill is valid!
```

## 许可

MIT
