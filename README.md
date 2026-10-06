# knowledge-guide

**面向 DeepSeek Harness（DSH）的「检索路由 + 回答规程」skill** —— 把「先检索再作答、强制标来源、置信度标注、未命中先做正对照」从每次会话的临场发挥，变成一份可复用、可审计的规程。

> 起因：DSH **没有自动上下文检索**。除了 `AGENTS.md`，任何知识都得我主动调工具才会进上下文。
> 于是同一个问题在不同会话里会得到不同答案 —— 有时检索、有时凭记忆编。
> 这不是提示词写得不够好，是**缺一条纪律**。

---

## 目录

- [它解决什么问题](#它解决什么问题)
- [安装](#安装)
- [规程长什么样](#规程长什么样)
- [依赖：AnySearch 检索通道](#依赖anysearch-检索通道)
- [诚实披露：这不是向量 RAG](#诚实披露这不是向量-rag)
- [本机踩过的坑](#本机踩过的坑)
- [License](#license)

---

## 它解决什么问题

| 现状 | 后果 |
|---|---|
| 模型有训练数据截止日期 | 问「最新现状」「某公司某产品」时，凭记忆作答等于编造 |
| 知识不进上下文 | 不主动调工具，就等于没有检索 —— 而「忘了调」是最常见的失败模式 |
| 回答无来源 | 无法验证，错误无法定位，对外材料一旦引用错数字就是事故 |
| 「查不到」不可信 | 检索通道故障与内容不存在，两种情况都会表现为「搜不到」 |

规程针对的就是这四点：**路由**（该查什么去哪儿查）、**标注**（来源 + 置信度）、
**证伪**（「没有」这个结论必须先自证通道有效）、**降级**（查不到就说查不到，不许填空）。

## 安装

DSH 只发现 `~/.dsh/skills/<name>/SKILL.md`（或平铺的 `<name>.md`），**嵌套的 `**/SKILL.md` 刻意不发现**。

```powershell
git clone https://github.com/Jav1es/dsh-knowledge-guide.git
$env:USERPROFILE\.dsh\skills\knowledge-guide\SKILL.md   # 放到这里才被 skill 工具识别
```

装完必须**实调 `skill knowledge-guide` 验收** —— frontmatter 缺 `name` 或 `description`
会**静默失效**（文件还在，但工具报 unknown，目录里也看不到，不给逐 skill 诊断）。

`name` 必须是 kebab-case（`^[a-z0-9]+(?:-[a-z0-9]+)*$`）；
frontmatter 只有白名单字段（`name` / `description` / `whenToUse` / `metadata` /
`disable-model-invocation` / `user-invocable`），未知字段不报错、只是被忽略。

本仓库是**纯 Markdown** —— 零依赖、零构建、不联网也能读。

## 规程长什么样

```
第一步  判断问题类型 ─→ 本地事实 / 外部现状 / 深度调研 / 推理写作
第二步  检索路由     ─→ 本地优先于联网；外部检索走 AnySearch 垂直库
第三步  回答格式     ─→ 结论先行 + 每点标来源 + 证据强度（高/中/低）
```

核心断言：**没检索过就不作答。**

例外写死在规程里 —— 纯技术概念解释、纯代码任务、用户明确说「就聊聊」不需要检索，
避免把「凡事必查」变成另一种浪费。

## 依赖：AnySearch 检索通道

外部检索走 DSH 的 web seam，当前钉死 `searchProvider: anysearch` / `fetchProvider: anysearch`。

- **垂直库优先**：`anysearch_capabilities` 查可用 vertical tag，再 `anysearch_search` 指定
  （`academic` / `code` / `business` / `finance` / `ip` / `legal` / `security`）。
- **时间精度按需给**：当前时间从 runtime context 读，**不要凭记忆**；问「最新」要给到月 + 日。
- **命中高价值来源要 `web_fetch` 读全文**，不只读 snippet。
- **跨源验证**：任何「数字 / 版本 / 日期」断言至少两个独立来源，否则标注「单一来源，未验证」。
- **批量上限**：`web_search` 1–4 个查询；`anysearch_batch_search` 1–5 个。

⚠️ 若你的环境未钉 `fetchProvider`，`web_fetch` 可能被 SSRF 守卫拦下（部分环境用 fake-ip 代理，
所有域名解析到 `198.18.0.0/15`）。本机是显式钉成服务端抓取来绕开的。

## 诚实披露：这不是向量 RAG

DSH 自带的 `knowledge` 库**当前是空的**，本 skill 也没用它：

- `~/.dsh/knowledge/knowledge.sqlite` 共 **18 张表**，但 `knowledge_documents` **0 篇**、
  `knowledge_entries` **0 条**、`knowledge_mounts` **0 个挂载**（`knowledge_bases` 表有 1 行占位）；
- 对应插件**未安装**。

所以这个 skill 走的是**三路文件式检索**，**不是向量 RAG、不做 embedding、不做语义召回**：

| 路 | 通道 | 覆盖 |
|---|---|---|
| 文件 | `read` / `grep` / `glob` | 项目约定、HANDOFF、配置事实 |
| TiddlyWiki | `tiddlywiki_search` 等 15 个工具 | DSH 运维与集成案例（长期记忆） |
| 外部 | AnySearch | 当前、外部、变化的现状 |

本地知识是**文件式**的 —— 这正是规程的设计前提。

## 本机踩过的坑

写进 `SKILL.md` 的「别重踩」清单，摘几条最容易复发的：

1. **不要装 `dsh-knowledge` 插件** —— peerDeps 带 5 个 `@deepseek-ai/*` 运行时内置包，
   实体化即双实例，直接崩。长期记忆交给已装的 TiddlyWiki。
2. **web seam 没有 provider 兜底链** —— `resolveProvider()` 配置命中即用，不可用就抛错，
   不会自动 fallback。
3. **换模型代价被低估** —— 切换 = KV cache 全失效 + 上下文重算，且目标档位倍率还要再乘一遍。
4. **别为「看图」切 glm-5.2/5.3** —— 它们是昂贵通用档，**且看不了图**（投递问题，不是模型能力问题）。
5. **改配置要在会话开始前做** —— 会话中途改 patch 等于摧毁缓存前缀。
6. **「不存在 / 没有」先做正对照** —— 拿一个已知存在的项验证检索通道本身有效，再下结论。

## License

[MIT](./LICENSE) © 2026 黄家辉 (Jav1es)