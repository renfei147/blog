---
title: "使用 Markdown 编写 LaTeX 文档"
subtitle: ""
date: 2023-10-05T14:23:01+08:00
author: "teralem"
authorLink: ""
authorEmail: ""
description: ""
keywords: ""
license: ""
comment: true
weight: 0

tags:

categories:


hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: featured-image.jpg
- name: featured-image-preview
  src: featured-image-preview.jpg

toc:
  enable: true
math:
  enable: true
lightgallery: false
seo:
  images: []

# See details front matter: /theme-documentation-content/#front-matter
---

Markdown 编辑器直接导出的 pdf 文档的质量往往欠佳，直接写 LaTeX 又太麻烦。本文简要记录了两种使用 Markdown 编写 LaTeX 文档的方法。

# 方法一：Pandoc

Pandoc 是一个非常强大的文档转换器，可以在各种文档格式之间转换。

## 基本用法

使用

```
pandoc report.md -o report.tex
```

就可以将 Markdown 转换为 LaTeX 源文件。不过这样得到的 LaTeX 源文件是不完整的，可以加上 `-s` 或 `--standalone` 来生成完整的可以直接编译的 LaTeX 源文件。注意 Pandoc 对 Markdown 的格式要求比较严格，需要确保每个块（正文段、标题、列表、代码块等）之间都有空行分隔。

```
pandoc report.md -o report.tex -s
```

也可以直接转换成 pdf 文件，这时候不需要 `-s`，可以用 `--pdf-engine=` 来指定渲染引擎。

```
pandoc report.md -o report.pdf --pdf-engine=xelatex
```

可以直接在 Markdown 中使用 tex 命令，pandoc 会将其原样保留，这样就可以插入无法用 Markdown 表示的内容。如果觉得这样不够清晰，也可以使用 pandoc 的 generic raw attribute 功能， 把 tex 命令写在标记为 ` ```{=tex}` 的代码块中。

## 设置变量

在 Markdown 文件开头可以加入 yaml 格式的 Metadata，这些 Metadata 会被用作模板的变量。典型的 Metadata 如下：

```yaml
---
title: '\textbf{标题}'
author:
- Eternal Dream
date: '2023-10-5'
documentclass: ctexart
geometry:
- margin=1in
papersize: a4
numbersections: true
indent: true
toc: true
---
```

首先可以设置标题、作者和日期。然后将 `documentclass` 设置为 `ctexart` 来支持中文。`geometry` 设置页面边距，`papersize` 设置纸张大小。

默认模板没有段落编号，且禁用缩进。可以用 `numbersections: true` 打开编号，用 `indent: true` 恢复 `ctexart` 的首行缩进。`toc: true` 可以生成目录。更多变量可以参考[Pandoc 文档](https://pandoc.org/MANUAL.html#variables-for-latex)。

## 模板

如果默认模板提供的变量还是不能满足自定义的需要，就需要使用自定义模板了，具体见[Pandoc 文档](https://pandoc.org/MANUAL.html#templates)。可以仿照默认模板里的格式添加自己的变量。

## 代码块

Pandoc 生成的 LaTex 源文件中的代码块默认是由 Pandoc 进行语法着色的。可以用 `--listings` 选项来使用 listings 宏包进行语法着色，再在自定义模板用 `\lstset` 就可以更加灵活地设置代码块的样式。

注意 `listings` 宏包不支持 UTF-8，实测在引入 `ctex` 并使用 xelatex 引擎的情况下是可以在代码块中包含中文的。

## SVG

如果在 Markdown 中使用了 SVG 图片，转换成 pdf 的时候 Pandoc 会尝试调用 `rsvg_convert` 来转换 SVG 图片。Windows上的 `rsvg_convert` 二进制文件可以在[这里](https://opensourcepack.blogspot.com/2012/06/rsvg-convert-svg-image-conversion-tool.html)找到。

# 方法二：Markdown 宏包

Markdown 宏包是一个可以在 TeX 中引用 Markdown 文件的宏包。可以在[这里](https://witiko.github.io/markdown/)找到使用手册。这个宏包比较新，实测在 texlive2022 中功能还不全，但在 texlive2023 中可以比较好地使用了。

演示：

```tex
\usepackage{markdown}
\markdownSetup{
  fencedCode = true, % 启用 ``` 代码块
  texMathDollars = true, % 启用行内数学公式
  pipeTables = true, % 启用表格
  rawAttribute = true, % 启用 raw attribute
}
\begin{document}

\begin{markdown}
These are just three regular dots ...
\end{markdown}
```

注意使用 Markdown 宏包后需要在编译命令中加上 `--shell-escape` 选项。

设置 `rawAttribute = true` 后也可以使用 pandoc 的 ` ```{=tex}` 代码块，其中的内容会被直接插入到 LaTeX 源文件中。
