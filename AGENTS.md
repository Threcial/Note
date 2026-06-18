---

type: Note
_organized: true
----------------

# AGENTS.md — Tolaria 知识库规则

这是一个 Tolaria 知识库仓库，主要用于整理 Linux 运维、服务部署、故障排查、实验记录、命令用法、技术博客草稿以及相关工具链内容。

本文件用于约束 agent 在这个仓库中的行为。
如果需要了解 Tolaria 本身的通用机制，应优先读取 Tolaria 应用会话中提供的官方 agent 文档；本文件只记录当前知识库的本地规则。

## 核心约定

* 笔记文件统一使用 Markdown。
* 每篇笔记必须使用第一个 H1 作为标题。
* Tolaria 会使用第一个 H1 作为笔记列表、搜索、wikilink 和展示界面的标题。
* 笔记类型统一写在 frontmatter 的 `type:` 字段中。
* 正文和 frontmatter 中都可以使用 wikilink 建立关系。
* 组织知识时优先使用类型、属性和关系，不要依赖文件夹表达语义。
* 文件夹可以用于临时收纳，但不能作为判断笔记含义的主要依据。
* Tolaria 会递归读取所有文件夹中的笔记。
* 新建笔记默认可以放在仓库根目录。
* Saved views 统一存放在 `views/*.yml`。
* `attachments/` 中的文件是附件资源，不是笔记。
* 不要把 `attachments/` 里的文件当作 Note、Type 或 View。
* frontmatter 中以下划线 `_` 开头的字段通常是 Tolaria 管理的状态，除非用户明确要求，否则不要修改。

## 语言与写作风格

本知识库主体使用中文。

创建、整理、重写笔记时，遵守以下规则：

* 不要写成课堂笔记。
* 不要使用明显的教学口吻。
* 不要使用“本文将介绍”“下面我们来学习”这类泛化表达。
* 保留实际操作、实验复盘、排错过程中的个人实践感。
* 保留原始材料中的重要知识点。
* 保留命令、路径、配置项、端口、服务名、报错信息、日志片段和实验现象。
* 可以重构结构，但不能删掉有价值的信息。
* 可以补充原理、场景、排错思路和易错点，但不要为了扩写而堆废话。
* 命令示例必须可以直接复制。
* 标题层级要清晰，但不要堆太多碎片化小标题。
* 避免大段空泛总结。
* 不要在文章末尾添加“检查结果”“是否遗漏”“是否过度教程化”之类的汇报段落，除非用户明确要求。

## 固定字段

新建笔记时，优先使用以下 frontmatter 结构：

```yaml
---
type: Concept
status: Inbox
Belongs to: "[[Linux]]"
Related to:
  - "[[Process]]"
  - "[[File Descriptor]]"
Has:
  - "[[lsof]]"
source: ""
created: ""
updated: ""
---
```

字段说明：

* `type:` 表示笔记类型。
* `status:` 表示整理状态。
* `Belongs to:` 表示所属主题或上级领域。
* `Related to:` 表示相关概念、命令、服务或实践。
* `Has:` 表示包含的子主题、组件、命令或延伸内容。
* `source:` 可以记录来源，例如课程、实验、网页、截图、博客迁移等。
* `created:` 可以记录创建日期。
* `updated:` 可以记录最后整理日期。

规则：

* 新笔记优先使用 `Belongs to:`、`Related to:`、`Has:`。
* 不要随意改成 `belongs_to`、`related_to`、`has`，除非旧文件已经使用这种风格。
* 编辑旧文件时，保留原有字段风格，不要大规模统一字段名。
* 单个关系值使用带引号的 wikilink。
* 多个关系值使用 YAML 列表。
* 不要把字段名翻译成中文，例如不要写 `类型:`、`状态:`、`属于:`。

## 笔记类型

新建技术笔记时，不要默认使用 `type: Note`。
优先从以下类型中选择：

```yaml
type: Concept
type: Command
type: Practice
type: Troubleshooting
type: Blog
type: Project
type: Tool
type: Reference
type: Type
```

类型含义：

* `Concept`：概念、机制、原理、协议、工作流程。
* `Command`：命令、命令族、参数组合、常用写法。
* `Practice`：实验记录、部署过程、配置实践、操作复盘。
* `Troubleshooting`：故障现象、报错信息、异常行为、排查过程。
* `Blog`：准备发布到 threcial.cn 的博客文章。
* `Project`：长期任务、知识库迁移、自动化流程、工具链规划。
* `Tool`：软件工具、插件、agent、编辑器、客户端、工作流组件。
* `Reference`：稳定参考资料、速查表、对比表、外部资料整理。
* `Type`：Tolaria 类型文档。

只有在内容暂时无法归类时，才使用 `type: Note`。

## 状态流转

统一使用以下 `status:` 值：

```yaml
status: Inbox
status: Active
status: Organized
status: Archived
```

状态含义：

* `Inbox`：刚导入、刚捕获、还没有整理的原始内容。
* `Active`：已经初步整理，但仍在学习、实验、补充或验证中。
* `Organized`：结构清晰、关系明确、内容稳定，可长期使用。
* `Archived`：过期、低频、被替代，或暂时不再维护。

使用规则：

* 导入原始笔记、截图内容、命令输出时，默认使用 `status: Inbox`。
* 整理完成但还需要继续实验验证时，使用 `status: Active`。
* 内容已经成型、能作为长期知识条目使用时，使用 `status: Organized`。
* 不再常用或已经被新内容替代时，使用 `status: Archived`。

## 常用上级主题

根据内容选择合适的 `Belongs to:`。

常用主题包括但不限于：

```yaml
Belongs to: "[[Linux]]"
Belongs to: "[[Shell]]"
Belongs to: "[[Nginx]]"
Belongs to: "[[Tomcat]]"
Belongs to: "[[MySQL]]"
Belongs to: "[[MariaDB]]"
Belongs to: "[[Redis]]"
Belongs to: "[[Zabbix]]"
Belongs to: "[[Docker]]"
Belongs to: "[[Git]]"
Belongs to: "[[Obsidian]]"
Belongs to: "[[Tolaria]]"
Belongs to: "[[WordPress]]"
Belongs to: "[[Network]]"
Belongs to: "[[Systemd]]"
```

规则：

* 能归入已有主题时，不要随意创建新主题。
* 一个笔记通常只设置一个主要 `Belongs to:`。
* 如果确实跨领域，可以把次要领域放入 `Related to:`。
* 不要为了显得复杂而创建大量弱关系。

## 关系字段

Tolaria 会把 frontmatter 中包含 wikilink 的属性视为关系。
本知识库优先使用以下三个关系字段：

```yaml
Belongs to: "[[Parent Topic]]"
Related to:
  - "[[Related Topic A]]"
  - "[[Related Topic B]]"
Has:
  - "[[Child Topic A]]"
  - "[[Child Topic B]]"
```

使用原则：

* `Belongs to:` 表示当前笔记属于哪个更大的主题。
* `Related to:` 表示当前笔记与哪些概念、命令、服务或故障有关。
* `Has:` 表示当前笔记包含哪些子主题、组件、命令或延伸内容。
* 关系必须有实际意义，不要为了建立关系而建立关系。
* 少量准确关系优先于大量模糊关系。
* 编辑已有笔记时，不要破坏原有 wikilink。

## Wikilink 规则

可以在正文和 frontmatter 中使用 wikilink。

可用格式：

```markdown
[[filename]]
[[Note Title]]
[[filename|display text]]
```

使用规则：

* 重要概念、命令、服务、工具、故障类型和上级主题可以使用 wikilink。
* 不要给每个普通词都加链接。
* 可复用的知识点应该链接到独立笔记。
* 一次性短语不需要强行创建笔记。
* 正文中第一次出现重要概念时，可以添加 wikilink。
* 后续重复出现时，不必每次都链接。

## 不同类型笔记的推荐结构

以下结构不是强制模板，但新建或整理笔记时应优先参考。

### Concept

适合概念、机制、原理类内容。

```markdown
# 标题

## 概览

## 背景与场景

## 概念拆解

## 工作机制

## 常见误区

## 关联内容
```

### Command

适合命令、参数、常用写法类内容。

```markdown
# 命令名称

## 核心用途

## 常用写法

## 参数拆解

## 使用场景

## 易错点

## 关联内容
```

### Practice

适合部署、实验、配置过程类内容。

```markdown
# 实践标题

## 实践目标

## 环境背景

## 配置过程

## 命令记录

## 现象与验证

## 问题与处理

## 复盘
```

### Troubleshooting

适合故障排查类内容。

```markdown
# 故障标题

## 现象

## 环境

## 初步判断

## 排查过程

## 根因

## 解决方式

## 延伸理解

## 关联内容
```

### Blog

适合发布到 threcial.cn 的文章。

```markdown
# 博客标题

## 背景

## 正文

## 实践过程

## 问题与处理

## 复盘
```

## 博客整理规则

当 `type: Blog` 时，按发布级技术博客处理。

规则：

* 输出完整 Markdown。
* 不要写成课堂讲义。
* 不要使用“教别人”的口吻。
* 保留个人实践复盘感。
* 不要遗漏原始材料中的知识点。
* 可以重构结构，让文章更适合发布。
* 可以补充原理、场景、坑点和排错思路。
* 不要把所有内容都写成列表。
* 命令、配置、日志片段必须格式化为代码块。
* 文章应适合发布到 threcial.cn。
* 不要在末尾额外汇报“已检查”。

## 原始材料整理规则

当用户提供课堂笔记、实验记录、截图说明、命令输出、报错日志或文档摘录时：

* 保留所有有意义的技术点。
* 保留原始命令。
* 保留路径、端口、配置文件名、服务名、错误信息。
* 保留实验现象。
* 可以调整顺序，使逻辑更清晰。
* 可以合并重复内容。
* 不要因为内容混乱就删除细节。
* 对明显错误的地方可以纠正，但要保留纠正后的解释。
* 对不确定的地方要标注“不确定”或“需要验证”，不要编造。
* 能补充现代命令时，可以同时补充，例如 `ip addr`、`ip route` 对应旧的 `ifconfig`、`route`。
* 不要创建 Mermaid 图。
* 不要使用 Obsidian callout，例如 `> [!summary]`。
* 不要把原始内容整理成过度基础的教程。

## Type 文档

Type 文档本身也是普通 Markdown 文件，但 frontmatter 中使用：

```yaml
type: Type
```

示例：

```yaml
---
type: Type
_icon: terminal
_color: "#3b82f6"
_order: 0
_list_properties_display:
  - status
  - Belongs to
  - Related to
_sort: "property:status:asc"
---

# Command
```

规则：

* Type 文档用于定义某一类笔记的默认属性、展示方式和建议关系。
* Type 文档中的空属性可以作为新笔记的占位字段。
* Type 文档中的非空属性可以作为该类型笔记的默认值。
* 可用元数据包括 `icon` / `_icon`、`color` / `_color`、`order` / `_order`、`sidebar label`、`_list_properties_display`、`_sort`、`template`、`view`、`visible`。
* 编辑已有 Type 文档时，保留原有字段风格。
* 不要在没有用户要求的情况下批量修改 Type 文档。

## Saved Views

Saved views 统一存放在：

```text
views/*.yml
```

Tolaria 会扫描 `views/` 目录下的 `.yml` 文件。
文件名就是稳定的 view id。

文件名使用 kebab-case，例如：

```text
inbox.yml
active-troubleshooting.yml
mysql-notes.yml
blog-drafts.yml
```

View 文件必须使用 YAML，不要创建 JSON view 文件，也不要使用 `.view.json` 文件名。

示例：

```yaml
name: Inbox
icon: null
color: null
sort: "modified:desc"
filters:
  all:
    - field: status
      op: equals
      value: Inbox
```

View 规则：

* `name` 必填。
* `icon`、`color`、`sort` 可选。
* `sort` 使用 `option:direction` 格式。
* 内置排序字段包括 `modified`、`created`、`title`、`status`。
* 自定义属性排序使用 `property:<Property Name>:<direction>`。
* `filters` 根节点必须是一个 `all:` 或一个 `any:`。
* 每个过滤条件使用 `field`、`op`，通常还需要 `value`。
* `field` 可以使用 `type`、`status`、`title`、`favorite`、`body` 等内置字段，也可以使用实际存在的 frontmatter 字段。
* 支持的操作符包括 `equals`、`not_equals`、`contains`、`not_contains`、`any_of`、`none_of`、`is_empty`、`is_not_empty`、`before`、`after`。
* `any_of` 和 `none_of` 的 `value` 应使用 YAML 列表。
* `equals`、`not_equals`、`contains`、`not_contains` 可以配合 `regex: true` 使用。
* 关系过滤可以使用 wikilink，例如 `value: "[[MySQL]]"`。

## 文件命名

Markdown 笔记文件名使用 kebab-case。

示例：

```text
mysql-binlog-recovery.md
nginx-root-alias.md
redis-sentinel-auth.md
zabbix-active-check-timeout.md
docker-compose-service-error.md
```

规则：

* 一个文件只存一篇笔记。
* 文件名优先使用稳定英文。
* H1 标题可以使用中文。
* 不要依赖文件夹表示笔记含义。
* 不要随意重命名已有文件。
* 只有在用户明确要求，或文件名明显错误时，才重命名。

## 附件规则

`attachments/` 中的文件都是附件资源。

规则：

* 不要把附件当作笔记。
* 不要给附件添加 `type:`。
* 不要把附件纳入 Type 或 View。
* 可以在 Markdown 正文中引用附件。
* 不要随意移动或重命名附件，除非用户明确要求。

## Agent 应该做的事

* 按本文件规则创建和编辑 Markdown 笔记。
* 新笔记优先使用固定 `type:`，不要默认写成 `type: Note`。
* 使用 H1 作为笔记标题。
* 使用 frontmatter 记录类型、状态和关系。
* 使用 wikilink 建立概念、命令、服务、故障和项目之间的连接。
* 创建或编辑 Type 文档时，遵守 Tolaria 的 Type 规则。
* 创建或编辑 Saved View 时，使用 `views/*.yml`。
* 整理原始材料时，保留技术细节。
* 重构内容时，优先提高结构清晰度，而不是删减信息。
* 需要了解 Tolaria 产品行为时，读取 Tolaria 官方 agent 文档。
* 组织知识时优先采用“类型 + 属性 + 关系”模型。
* 对新捕获内容使用 `Inbox`，对整理中的内容使用 `Active`，对稳定内容使用 `Organized`，对过期内容使用 `Archived`。

## Agent 不应该做的事

* 不要根据文件夹推断笔记类型。
* 不要把 `attachments/` 中的文件当作笔记。
* 不要静默覆盖已有自定义 `AGENTS.md`。
* 不要在用户没有要求时修改安装相关配置。
* 不要批量统一旧文件的 frontmatter 字段风格。
* 不要把已有 `Belongs to:`、`Related to:`、`Has:` 改成别的字段名。
* 不要删除原始命令、报错、路径、配置项和重要技术细节。
* 不要把技术笔记写成泛泛的入门教程。
* 不要创建 Mermaid 图，除非用户明确要求。
* 不要使用 Obsidian callout，除非用户明确要求。
* 不要把博客草稿写成课堂笔记。
* 不要凭空补充未经验证的结论。
