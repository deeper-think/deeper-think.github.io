---
layout: post
title: HTML页面禁止鼠标右键
category: 技术
tags: [HTML, JavaScript]
keywords: Multipath,TCP
description:
---

> 用正确的工具，做正确的事情


## 1 HTML禁止鼠标右键代码

	<body oncontextmenu='return false' ondragstart='return false' onselectstart ='return false' onselect='document.selection.empty()' oncopy='document.selection.empty()' onbeforecopy='return false' onmouseup='document.selection.empty()'>
	...
	</body>



Have a fun！！！