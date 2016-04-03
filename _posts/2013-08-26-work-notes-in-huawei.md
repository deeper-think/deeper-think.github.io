---
layout: post
title: 工作笔记（华为）
category: 工作笔记
tags: [work-notes]
keywords: work-notes
description:
---

> 用正确的工具，做正确的事情

### 1 gdb常用调试命令

	backtrace（或bt）	\\查看各级函数调用及参数
	finish				\\连续运行到当前函数返回为止，然后停下来等待命令
	frame（或f） 		\\帧编号	选择栈帧
	info（或i） 			\\locals	查看当前栈帧局部变量的值
	list（或l）			\\列出源代码，接着上次的位置往下列，每次列10行
	list 行号			\\列出从第几行开始的源代码
	list 函数名			\\列出某个函数的源代码
	next（或n）			\\执行下一行语句
	print（或p）			\\打印表达式的值，通过表达式可以修改变量的值或者调用函数
	quit（或q）			\\退出gdb调试环境
	set var				\\修改变量的值
	start				\\开始执行程序，停在main函数第一行语句前面等待命令
	step（或s）			\\执行下一行语句，如果有函数调用则进入到函数中
	
### 2 source Insight常用快捷键总结

	Ctrl+= :Jump to definition
	Alt+/ :Look up reference
	F3 : search backward
	F4 : search forward
	F5: go to Line
	F7 :Look up symbols
	F8 :Look up local symbols
	F9 :Ident left
	F10 :Ident right
	Alt+, :Jump backword
	Alt+. : Jump forward
	Shift+F3 : search the word under cusor backward
	Shift+F4 : search the word under cusor forward
	F12 : incremental search
	Shift+Ctrl+f: search in project
	shift+F8 : hilight word



Have a fun！！！