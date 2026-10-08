# 教学层（teaching layer）约定

这一份说明的是本站（Hugo 官方文档简体中文翻译站）**独有的**结构：在忠实翻译之上，再补一层
「让没有 AI 辅助的普通读者也能照着做完」的内容。它与上游文档的克制风格并存，不替换上游内容。

## 为什么需要它

上游文档默认读者懂命令行、能自己补齐上下文、遇到报错会自己查。中文读者面对的是三重额外成本：
命令行的文化差异、跨平台差异（Windows/macOS/Linux）、以及国内网络环境。上游只给结论的地方，
正是本地化要补的「为什么 / 会怎样 / 怎么验证」。

判断标准只有一条：**读者照做之后，能不能自己确认做对了**。能被检验的断言才有教学价值。

## 数据契约：`[params.teach]`

教学信息放在页面自己的前置元数据里，而不是散落在正文中，这样 HTML 与 Markdown 出口能读同一份数据。

```toml
+++
title = "…"
linkTitle = "…"
description = "…"
date = 2026-10-01
weight = 10
source = "https://gohugo.io/…"      # ← 标量必须在所有表头之前（见下）

[params.teach]
difficulty = "入门"                  # 入门 / 进阶 / 参考
time       = "15–20 分钟"            # 字符串；纯数字会被 TOML 解析成整数/Epoch
prereq     = ["…"]                   # 开始之前需要具备什么（支持行内 Markdown）
outcomes   = ["…"]                   # 读完之后能做到什么
next       = ["/installation/"]      # 接着读（站内根相对路径）
+++
```

- 五个键全部可选；一个都没有时整块不渲染，因此旧页面不受影响。
- `next` 的旧名是 `readAfter`，模板仍兼容。
- 顶层 `[teach]` 表也能被模板读到（兼容早期写法），但新页面一律写 `[params.teach]`。

### TOML 作用域（最容易踩的一处）

`[table]` 之后的裸键属于该表。把 `source` 写在 `[params.teach]` **之后**，它就变成
`params.teach.source`，六字段契约静默失效、页脚不再有原文链接，而 **Hugo 不会报错**。
正确顺序：六个标量字段 → `[params.teach]` → `[params.functions_and_methods]`。同类陷阱见 G21。

## 渲染：两个出口、一份数据

| 出口 | 模板 | 位置 |
| --- | --- | --- |
| HTML 面板（人类） | `themes/hugo-docs-theme/layouts/partials/teach-box.html` | `single.html` 中 `function-meta` 之后、`.doc-body` **之前** |
| Markdown 引用块（机器） | `themes/hugo-docs-theme/layouts/partials/teach-md.html` | `single.md.md` 中摘要之后 |

两个 partial 都从 `.Params.teach` 取值，因此**人类与机器看到同一份事实**，不会分叉（G24）。
面板在 `.doc-body` 之外，所以只抽 `.doc-body` 的抓取器会漏掉它——这一点已写进站点的 `/llms.txt`。

样式在 `themes/hugo-docs-theme/assets/css/main.css` 的 `.teach` 一组：用主题既有的
`--bg-soft` / `--border-soft` / `--brand` 变量，因此深色模式自动生效，无需另写媒体查询。

## 正文增补的写法

按页面角色决定力度，原则是**只增不删**——上游的命令、签名、默认值、表格一行都不能丢：

| 角色 | 增补要求 |
| --- | --- |
| 教程 / 上手（`getting-started`、`installation`） | 目标、前置、分步、每步验证标准、常见坑表、下一步 |
| 流程型章节（`templates`、`render-hooks`、`hugo-pipes` 等） | 每小节说明「在解决什么问题」+ 最小可运行示例 + 结果长什么样 |
| 参考页（`functions`、`methods`、`commands`） | 忠实翻译为主，补「什么时候用 / 别用」与返回值边界 |
| 术语 / 速查 | 保持条目化，不扩写 |

三条硬性要求：

1. **实测与文档分界**：上游没写、由本站实测得出的结论，必须写「实测：……」；**不得把推断写成官方结论**。
2. **首次出现的术语**写「中文（english）」，其后沿用中文。
3. 站内链接一律根相对且**全小写**（Hugo 输出 URL 小写，写驼峰会产生死链）。

## 覆盖度审计

```powershell
pwsh -NoProfile -File .translation/audit-teach.ps1                  # 全站概览 + 按章节明细
pwsh -NoProfile -File .translation/audit-teach.ps1 -Strict          # 教程章节缺教学块即失败
pwsh -NoProfile -File .translation/audit-teach.ps1 -Section getting-started
```

只读、幂等。教程章节（`getting-started` / `installation` / `troubleshooting`）按严格口径要求
教学块覆盖。

## 一致性检查清单

- [ ] `hugo --ignoreCache --renderToMemory --quiet` 退出码为 0；
- [ ] 该页 `public/<path>/index.html` 里有 `<section class="teach">`，且页脚仍有原文链接（`source` 没被表头吞掉）；
- [ ] 该页 `public/<path>/index.md` 里有「**教学信息**」引用块，且前置字段顺序正确；
- [ ] 正文没有未转义的 `{{<` / `{{%`（`note` 短代码除外），没有字面串 `HAHAHUGOSHORTCODE`；
- [ ] 正文从 `##` 开始，站内链接全小写且可解析；
- [ ] `.translation/audit-teach.ps1 -Strict` 通过。

## 状态标注

- **observed**：`[params.teach]` 契约、两个 partial 的行为、`audit-teach.ps1` 的口径，均在本仓库实测；
  反例（`source` 被表头吞掉、HTML 与 md 出口分叉）见 G21 与 G24。
- **documented**：Hugo 的 output format 与 partial 查找规则，见
  <https://gohugo.io/configuration/output-formats/> 与 <https://gohugo.io/templates/partials/>。
