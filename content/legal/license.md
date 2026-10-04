+++
title = '许可'
linkTitle = '许可'
description = '正文用 CC BY 4.0，主题与代码用 MIT：可以把什么拿走、需要保留什么。'
date = 2026-01-05
weight = 20
+++

## 两套条款，各管一半

这个仓库里有两类东西，条款不同：

| 内容 | 位置 | 许可 |
| --- | --- | --- |
| 正文与配图 | `content/` 下的文字与图片 | CC BY 4.0 |
| 主题、模板、样式与脚本 | `themes/hugo-scratch-theme`、`layouts/`、`assets/`、`i18n/` | MIT |
| 正文里出现的代码片段 | 文章中的围栏代码块 | MIT |

分界线的依据是「这段东西是拿来读的，还是拿来跑的」：文章是作品，代码是工具。
同一个页面里两者都有，所以一页上的内容可能同时受两套条款约束。

## 正文：CC BY 4.0

`content/` 下的文字与图片采用
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/)。
说得直白一点：你可以复制、翻译、改写、印出来、拿去做培训材料，甚至商用，
**唯一的要求是署名**。

署名的写法没有规定格式，够用即可，例如：

```text
来源：Hugo Scratch — https://hencter.github.io/hugo-scratch/
许可：CC BY 4.0
```

加上许可链接和「是否做过修改」的说明就完整了。
中英两版不是逐句互译，所以引用时请注明引用的是哪一版，并给出该语言页面的地址。

## 代码与主题：MIT

主题仓库、站点里的 `layouts/` 与 `assets/`，以及文章里的代码块，
采用 MIT 许可：

```text
MIT License

Copyright (c) 2026 Hencter Lew

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

完整文本也在主题仓库根目录的 `LICENSE` 文件里，`theme.toml` 的 `license = "MIT"`
与它一致。MIT 的实际要求只有一条：**保留版权声明与这份许可文本**。
没有「必须开源你的改动」这类条件——那是 copyleft，MIT 不是。

## 第三方与依赖

站点本身没有引入第三方的前端代码：没有 CDN 上的脚本、没有字体、没有分析。
Hugo 二进制与主题里内嵌的 esbuild 各自受它们自己的许可约束，
既不在这个仓库里分发，也不受这里两套条款影响。

引用外部资料时，那部分内容的权利仍然归原作者；本页的许可只覆盖这个仓库自己的产出。

至于站点在浏览器里会请求什么，以及为什么默认没有分析，
写在[隐私](/legal/privacy/)一页——两页描述的是同一份实现。
仓库地址：<https://github.com/hencter/hugo-scratch>，
主题：<https://github.com/hencter/hugo-scratch-theme>。
