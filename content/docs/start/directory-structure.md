+++
title = '目录结构'
linkTitle = '目录结构'
description = '仓库里每个目录负责什么，以及哪些目录是构建产物、手改等于白改。'
date = 2026-01-10
weight = 20
difficulty = 'beginner'
estimatedTime = 10
prerequisites = ['/docs/start/quick-start/']
outcomes = ['说出每个顶层目录的用途', '分清源码目录和构建产物目录']
tags = ['Hugo']
+++

改这个站点之前，先花两分钟认清目录。这个仓库里的大部分目录你都可以直接编辑，但有一小撮是构建写出来的：改了它们，要么在下一次构建时被覆盖，要么让构建读到自相矛盾的状态。这两类目录混在一起看，是新手最常见的返工原因。

## 仓库全景

下面是当前仓库的目录树。缩进就是实际层级，标注说明的是每个目录的角色。

{{< filetree >}}
hugo-scratch/
├── archetypes/                 新建内容用的模板
│   └── default.md              `hugo new` 的默认骨架
├── assets/                     Hugo Pipes 处理的源文件
│   ├── css/custom.css          站点自己的样式，覆盖主题
│   └── jsconfig.json           js.Build 的解析配置
├── config/
│   ├── _default/               站点配置本体
│   │   ├── hugo.toml           主配置
│   │   ├── languages.toml      语言与 locale
│   │   ├── menus.zh-cn.toml    中文菜单
│   │   ├── menus.en.toml       英文菜单
│   │   └── params.toml         模板读取的站点参数
│   └── production/
│       └── hugo.toml           仅生产环境叠加
├── content/                    全部页面，按 URL 组织
│   ├── _index.md               首页
│   ├── about/index.md          /about/
│   ├── blog/                   博客章节与文章
│   ├── changelog/_index.md     /changelog/
│   ├── docs/                   文档章节
│   │   ├── _index.md           /docs/
│   │   └── start/              起步子章节
│   └── legal/                  法律声明章节
├── data/                       模板可读的结构化数据
├── i18n/                       界面文案的翻译目录
├── layouts/                    站点自己的模板覆盖
├── static/                     原样拷贝的输出文件
├── themes/
│   └── hugo-scratch-theme/     主题，一个 git 子模块
├── .gitignore                  忽略构建产物
└── .hugo_build.lock            构建锁，由 Hugo 写入
{{< /filetree >}}

## 每个目录的职责

| 目录 | 里面放什么 | 能不能手改 |
| --- | --- | --- |
| `content/` | 所有页面。目录结构直接映射到 URL，`docs/start/quick-start.md` 就是 `/docs/start/quick-start/` | 能，这是你日常改的地方 |
| `config/_default/` | 站点配置。拆成 `hugo.toml`、`languages.toml`、`params.toml` 和两个菜单文件 | 能 |
| `config/production/` | 只在生产构建时叠加的配置 | 能 |
| `layouts/` | 站点自己的模板，覆盖主题里的同名文件 | 能，但优先改主题 |
| `assets/` | 需要经 Hugo Pipes 处理的 CSS 和 JS | 能 |
| `static/` | 原样复制到输出根目录的文件，如站点图标 | 能 |
| `data/` | 模板通过 `site.Data` 读取的结构化数据 | 能 |
| `i18n/` | 界面文案的键值翻译；缺失的键由主题的 `i18n/` 兜底 | 能 |
| `archetypes/` | `hugo new` 生成新页面时用的骨架 | 能 |
| `themes/` | 主题，作为 git 子模块挂载。改这里等于改另一个仓库 | 本地调试可以，提交要回到主题仓库 |

`themes/` 这一行值得多看一眼：`themes/hugo-scratch-theme` 有自己的 `.git`，所以在这里做的修改属于主题仓库的改动，不属于站点。如果你想改主题的模板给本站用，正确的位置是本站的 `layouts/`——它优先于主题，并且随站点一起提交。

## 绝对不能编辑的目录

下面四个名字出现在仓库里，但它们不是源码。它们由 Hugo 写出，被 `.gitignore` 忽略，手动修改的后果各不相同：

- **`public/`** ——正式构建的输出。默认情况下 `hugo` 把整个站点写到这里，每次构建开始时这里的内容都可能被重写。你在这里改的任何东西都不会进入下一次构建。
- **`resources/`** ——Hugo 的资源缓存，用来存放处理过的图片、拼接后的 CSS/JS 等中间产物。它存在的意义就是让你不用重复计算；手工编辑它，轻则下次构建覆盖，重则让缓存与源文件不一致。
- **`.hugo_build.lock`** ——Hugo 构建时创建的锁文件，用来防止两个构建同时写同一个输出目录。它应该在 `.gitignore` 里，不应该被提交，也不应该被手工编辑。构建异常中断后如果它残留，通常直接删掉即可。
- **`hugo_stats.json`** ——由 `[build.buildStats] enable = true` 在每次构建时写出，内容是本次构建实际用到的标签和类名；Tailwind 靠它只生成真正出现过的工具类。它同样是产物，改了等于改一份会被重新生成的报告。
- **`node_modules/`、`package.json`、`package-lock.json`** ——Tailwind v4 的 CLI 与库，用 `npm ci` 按锁文件还原。它们不是构建产物，但 `node_modules/` 不进版本库。

判断一个陌生的目录属于哪一类，有一个简单的办法：看 `.gitignore`。被忽略的是产物，没被忽略的是源码。这个仓库的 `.gitignore` 里就是这四项加上编辑器与系统的杂项文件。

## 找文件的两个习惯

第一，用页面路径反推文件路径。URL `/docs/start/quick-start/` 对应的就是 `content/docs/start/quick-start.md`；章节首页 `/docs/` 对应 `content/docs/_index.md`。`_index.md` 和 `index.md` 不是同一个东西：前者是章节节点，后者是叶子包，一个目录不能同时拥有两者。

第二，用 Hugo 自己报告内容清单，而不是在文件树里数文件：

```bash
hugo list all
```

它一次列出 Hugo 认可的全部内容文件及其路径、日期和类型。如果你刚新建了一页却发现它不在列表里，通常是文件名不对（例如把 `_index.md` 打成了 `index.md`），或者页面被标成了草稿。

弄清楚这两件事之后，就可以进入下一章，看这些配置文件是怎么被合并成一份生效配置的。
