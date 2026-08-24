---
# 文章标题："hugo new" 会自动把文件名中的 "-" 替换为空格并转为标题格式
# （例如 my-first-post -> My First Post），也可以之后手动修改。
title: "{{ replace .Name "-" " " | title }}"

# 文章摘要：显示在首页/归档的文章卡片和文章页顶部，同时用于 SEO 和 RSS。
# 留空时 Hugo 会自动截取正文开头作为摘要。
description:

# 自定义 URL 路径（本站 permalinks 配置为 /post/<slug>/）。
# 留空时默认使用文章所在文件夹的名称。
slug:

# 发布时间："hugo new" 时自动填入当前时间，决定文章的排序与页面显示的时间。
date: "{{ .Date }}"

# 封面图：填写与本文件同目录下的图片文件名（如 cover.jpg），
# 会显示在文章卡片和文章页顶部；留空则不显示封面。
image:

# 分类：一般每篇文章一个，点击后进入对应的分类归档页。
categories:

# 标签：可以写多个，比分类更自由，用于细化标注文章主题。
tags:

# 是否为本文启用 KaTeX 数学公式渲染（$...$ 行内、$$...$$ 块级）；
# 留空时跟随站点配置（params.toml 中的 article.math）。
math:

# 是否显示目录（TOC）：文章至少包含一个标题时才会渲染；
# 留空时跟随站点配置（params.toml 中的 article.toc）。
toc:

# 草稿开关：true 时生产构建（hugo）不会生成该页面，
# 仅本地预览（hugo server -D）可见；正式发布时改为 false 或删除此行。
draft: true
---
