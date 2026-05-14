# Ext27：Hermes-Wiki——让知识库通过双链自己长出网络

> 前几篇解决了"AI 能操作知识库"的问题。但跑了一段时间发现：Agent 帮你建了一堆页面，页面之间是孤立的。图谱打开一看——散的。这次升级的核心目标：让 Agent 通过 Obsidian 的 wikilink 能力，把一篇文章自动编译成互相关联的知识节点，在知识库里自己长出网络。

---

## 核心流程升级

```
Command 文章 → Agent 拆解 → 概念页 / 实体页 / 比较页 / MOC
       ↓
  每个页面通过 [[wikilink]] 互相关联
       ↓
  Obsidian 图谱中形成可视化知识网络
```

从「存 + 搜」到：**让知识自己长起来**。

---

## 一、文件夹结构——让 Agent 区分节点类型

```
Hermes-Wiki/
├── SCHEMA.md
├── index.md
├── log.md
├── raw/
│   ├── articles/
│   ├── papers/
│   ├── transcripts/
│   └── assets/
├── concepts/
├── entities/
├── comparisons/
├── queries/
├── moc/
└── drafts/
```

**核心原则只有一条：不同类型的内容不要混在一起。**

| 目录 | 用途 | 原则 |
|------|------|------|
| `raw/` | 原始资料 | 只追加只读，不修改原文。raw 是证据层，源头坏了后面全乱 |
| `concepts/` | 概念页 | RAG、Agentic RAG、LLM Wiki、MOC。不是文章摘要，是跨资料沉淀的长期知识节点 |
| `entities/` | 实体页 | Obsidian、Hermes、Claude Code、某家公司。概念和实体必须分开 |
| `comparisons/` | 比较页 | Hermes vs Claude Code、RAG vs Agentic RAG。适合长期更新 |
| `queries/` | 值得保留的问答 | 不是每次聊天都存，解决重要问题的才沉淀 |
| `moc/` | 主题地图 | 理解路线图，不是目录。告诉阅读顺序 |
| `drafts/` | 输出草稿 | 知识库的终局是输出，不是收藏 |

---

## 二、三个根目录文件——这才是真正的核心

### SCHEMA.md：规则文件

定义整个 Wiki 的规则，Agent 的所有行为受它约束：

```
• raw 是原始资料区，只能追加和读取，不要改写原文
• concepts 存放概念页
• entities 存放实体页
• comparisons 存放比较页
• queries 存放值得长期保留的问答结果
• moc 存放主题地图
• drafts 存放输出草稿
• 重要概念使用 [[wikilink]]
• 关键结论尽量绑定来源
• 不确定内容必须标记为"待验证"
• 每次重要修改后更新 log.md
• 新增重要页面后更新 index.md
```

**不给规则，Agent 就会自作主张。** 今天建 summary，明天建 note，后天换一种命名——最后得到的不是知识库，是一堆 AI 生成的新垃圾。

特别强调：**「重要概念使用 wikilink」是整个知识网络能形成的底层前提。** 没有双链，页面再多也是孤岛，Obsidian 图谱里什么都看不到。

### index.md：总入口

早期三块就够：

```markdown
# LLM Wiki Index

## 核心概念
- [[RAG]]
- [[Agentic RAG]]
- [[LLM Wiki]]
- [[Obsidian]]
- [[MOC]]
- [[双链交叉引用]]
- [[Source-first]]

## 主题地图
- [[RAG 技术地图]]
- [[LLM Wiki 地图]]

## 最近更新
待更新。
```

### log.md：更新日志

每次新建/更新页面、标注来源、标记待验证项，都记进 log.md。

没有 log，AI 知识库就是黑箱。页面多了你不知道哪来的、为什么创建、哪些是原文结论、哪些是 AI 归纳。这对长期知识库是致命的。

---

## 三、Obsidian 打开文件夹

打开 Obsidian → `Open folder as vault` → 选择整个 `Hermes-Wiki` 文件夹（不要选子目录）。

打开成功后侧边栏应该看到：
- SCHEMA.md / index.md / log.md
- raw / concepts / entities / comparisons / queries / moc / drafts

---

## 四、设置 WIKI_PATH

告诉 Hermes：Wiki 文件夹在哪里。

**Windows + WSL 用户**（最容易出错）：
```bash
export WIKI_PATH="/mnt/c/Users/你的用户名/Hermes-Wiki"
```

**Mac 用户**：
```bash
export WIKI_PATH="$HOME/Hermes-Wiki"
```

验证：
```bash
echo $WIKI_PATH
ls "$WIKI_PATH"
```

应该看到：`SCHEMA.md  index.md  log.md  raw  concepts  entities  comparisons  queries  moc  drafts`

**关键：Obsidian 和 Hermes 必须指向同一个文件夹。路径不对，Obsidian 里永远看不到变化。**

---

## 五、先只读，不要急着改

第一次启动 Hermes 后，先只做初始化检查：

```
请读取我的 LLM Wiki。Wiki 路径使用环境变量 WIKI_PATH。

先不要修改任何文件，只做初始化检查：
1. 读取 SCHEMA.md
2. 读取 index.md
3. 读取 log.md
4. 总结当前 Wiki 结构
5. 告诉我下一步应该收录哪些资料
```

**验证标准**：Hermes 能准确说出：
- raw/ = 原始资料
- concepts/ = 概念页
- moc/ = 主题地图
- SCHEMA.md = 规则文件
- index.md = 总入口
- log.md = 更新日志

说对了，再进入下一步。

---

## 六、收录第一篇文章

在 Obsidian 进入 `raw/articles/`，新建 Markdown 文件粘贴文章内容。

然后 Hermes 输入：

```
请收录这篇文章：

${WIKI_PATH}/raw/articles/你的文章标题.md

注意：这是一篇 raw 原始资料，请把它作为原始资料处理，不要修改原文。

要求：
1. 读取文章内容，提取核心主题
2. 根据内容创建或更新对应的 Wiki 页面
3. 重要概念使用 [[wikilink]]
4. 关键结论必须标注来源：raw/articles/你的文章标题.md
5. 如果某些内容是总结归纳或待验证判断，请明确标记
6. 更新 index.md
7. 更新 log.md
8. 如果适合，请创建或更新 moc/LLM Wiki 地图.md
9. 完成后列出本次新增和更新了哪些文件
```

**四个要点：**
1. raw 原文不要改
2. 关键结论标注来源
3. 要求列出新增/更新文件
4. 更新 log.md

这四条确保这是一次**可检查的知识编译**，不是随手总结。

---

## 七、回到 Obsidian 检查五个地方

### 1. concepts — 概念页是否生成

应该能看到从文章中拆出的长期知识节点：
- LLM Wiki / RAG / Agentic RAG / MOC / Source-first / 知识飞轮

### 2. entities — 实体页是否生成

概念和工具/实体应该分开：
- Obsidian / Hermes / Claude Code / NotebookLM

### 3. moc — 主题地图是否生成

不是列清单，而是理解路线图——哪些是入口概念，哪些是进阶，哪些资料值得反复读。

### 4. log.md — 更新是否有记录

必须记录：新增/更新了哪些页面、来源文章、待验证内容、下一步建议。

### 5. 图谱 — 是否形成网络（关键验收点）

打开 Obsidian 图谱视图。如果 wikilink 规则生效了，应该看到：
- concepts 里的概念页互相链接
- entities 里的实体页和概念页交叉关联
- moc 作为中枢节点把多个页面串在一起

**这才是核心验收点：一篇文章 → Agent 拆出 5-10 个页面 → 每个页面用 [[wikilink]] 互相链接 → Obsidian 图谱中形成可见的知识网络。**

文章越多，网络越密。这就是「知识库自己长大」的真正含义。

---

## 八、本质变化

和基础流程比，这次升级的关键差异不在「操作」，在「关联」：

| 变化 | 之前 | 现在 |
|------|------|------|
| 从存资料到 | 存资料 | 编译资料——拆成可被引用的独立节点 |
| 从页面孤立到 | 孤立页面 | 自动关联——Agent 强制用 [[wikilink]] 把新页面和已有页面关联 |
| 从线性输出到 | 一篇文章 → 一篇总结 | 一篇文章 → 多个节点 → 和已有节点自动关联 → 图谱越来越密 |

**文章越多，网络价值越大。不是线性叠加，是指数增长。**

---

## 九、启动建议（不要一上来全自动）

1. 只让 Agent 操作独立的 `Hermes-Wiki`，不要直接动主力 Vault
2. raw 原始资料只追加只读，不让 AI 改写
3. 每次生成后看 log.md，确认改了什么
4. 不确定内容一律标"待验证"
5. 先用 3-5 篇文章测试，命名、双链、MOC 规则稳定后再扩大到论文、转录稿

**AI 知识库最怕的不是不够自动，是自动生成一堆你自己都不信的东西。**

---

## 总结

关键就三点，缺一个都不行：

- **wikilink 规则**：让 Agent 强制建双链，没有这一步，图谱永远是空的
- **节点类型分离**：概念、实体、比较、MOC 各司其职，才有结构化网络
- **log 追踪**：知道每个节点从哪来、为什么创建，网络才可信

人负责判断和审阅，Obsidian 负责存储和可视化，Agent 负责拆解和建链——三者打通，知识库自己就长起来了。
