---
layout: post
title: DrawIt部署及使用
category: 工具使用
tags: [systemtap]
keywords: systemtap
description: 
---

> 我思故我在 -- 笛卡尔

DrawIt是Vim画图的插件，可以使用该插件通过vim来方面的画矢量图，下面记录DrawIt的安装和部署以及常用的使用命令。

## DrawIt部署及测试

在vim的 --[官方站点](http://www.vim.org/scripts/script.php?script_id=40) 来选择合适的版本合适的DrawIt插件版本。注意对于各DrawIt版本都注明的对应的vim版本，你必须根据你linux系统上vim的版本来选择匹配版本的DrawIt， 否则解压并安装该DrawIt插件的时候会报错。

选择并下载合适的DrawIt版本后，得到的是一个gz文件：DrawIt.vba.gz，用vim 打开该文件会得到提示：

	Source this file to extract it! (:so %)

根据提示在vim输入:so %, 如果没有报错一般会得到下面的输出，表示解压安装成功：

	Vimball Archive
	extracted <plugin/DrawItPlugin.vim^I[[[1>: 76 lines
	wrote /usr/share/vim/vim70/plugin/DrawItPlugin.vim      [[[1
	extracted <plugin/cecutil.vim^I[[[1>: 536 lines
	wrote /usr/share/vim/vim70/plugin/cecutil.vim   [[[1
	extracted <autoload/DrawIt.vim^I[[[1>: 2884 lines
	wrote /usr/share/vim/vim70/autoload/DrawIt.vim  [[[1
	extracted <doc/DrawIt.txt^I[[[1>: 474 lines
	wrote /usr/share/vim/vim70/doc/DrawIt.txt       [[[1
	Press ENTER or type command to continue

从上面的输出信息 可以看到 DrawIt的安装过程无非就是拷贝一些脚本和说明文件到vim的安装目录。之后随便用vim编辑一个文件，输入：DIstart 用来在vim里面启动DrawIt插件，报错：

	E492: Not an editor command: DIstart

到vim plugin目录检查安装的脚本文件以及安装过程上面的输出，发现如下可疑之处：

![drawit安装](http://7u2rbh.com1.z0.glb.clouddn.com/drawit.png)

根据上面安装过程输出信息，到各个目录下修正已安装文件的文件名，然后重新测试，可疑正常使用。


## DrawIt使用

下面DrawIt说明文档DrawIt.txt中摘抄出来常用的命令：

	/===============+============================================================	\
 	|| Starting &   |                                                           ||
 	|| Stopping     | Explanation                                               ||
 	++--------------+-----------------------------------------------------------++
 	||  \di         | start DrawIt                     |drawit-start|             ||
 	||  \ds         | stop  DrawIt                     |drawit-stop|              ||
 	||  :DIstart    | start DrawIt                     |drawit-start|             ||
 	||  :DIstart S  | start DrawIt in single-bar mode  |drawit-start|             ||
 	||  :DIstart D  | start DrawIt in double-bar mode  |drawit-start|             ||
 	||  :DIsngl     | start DrawIt in single-bar mode  |drawit-start| |drawit-sngl| ||
 	||  :DIdbl      | start DrawIt in double-bar mode  |drawit-start| |drawit-dbl|  ||
 	||  :DIstop     | stop  DrawIt                     |drawit-stop|              ||
 	||  :DrawIt[!]  | start/stop DrawIt                |drawit-start| |drawit-stop| ||
 	||              |                                                           ||
	++==============+===========================================================++
	||   Maps       | Explanation                                               ||
	++--------------+-----------------------------------------------------------++
	||              | The DrawIt routines use a replace, move, and              ||
	||              | replace/insert strategy.  The package also lets one insert||
	||              | spaces, draw arrows by using the following characters or  ||
	||              | keypad characters:                                        ||
	||              +-----------------------------------------------------------++
	|| <left>       | move and draw left                         |drawit-drawing| ||
	|| <right>      | move and draw right, inserting lines/space as needed      ||
	|| <up>         | move and draw up, inserting lines/space as needed         ||
	|| <down>       | move and draw down, inserting lines/space as needed       ||
	|| <s-left>     | move cursor left                              |drawit-move| ||
	|| <s-right>    | move cursor right, inserting lines/space as needed        ||
	|| <s-up>       | move cursor up, inserting lines/space as needed           ||
	|| <s-down>     | move cursor down, inserting lines/space as needed         ||
	|| <space>      | toggle into and out of erase mode                         ||
	|| >            | insert a > and move right    (draw -> arrow)              ||
	|| <            | insert a < and move left     (draw <- arrow)              ||
	|| ^            | insert a ^ and move up       (draw ^  arrow)              ||
	|| v            | insert a v and move down     (draw v  arrow)              ||
	|| <pgdn>       | replace with a \, move down and right, and insert a \     ||
	|| <end>        | replace with a /, move down and left,  and insert a /     ||
	|| <pgup>       | replace with a /, move up   and right, and insert a /     ||
	|| <home>       | replace with a \, move up   and left,  and insert a \     ||
	|| \>           | insert a fat > and move right    (draw -> arrow)          ||
	|| \<           | insert a fat < and move left     (draw <- arrow)          ||
	|| \^           | insert a fat ^ and move up       (draw ^  arrow)          ||
	|| \v           | insert a fat v and move down     (draw v  arrow)          ||
	||<s-leftmouse> | drag and draw with current brush            |drawit-brush|  ||
	||<c-leftmouse> | drag and move current brush                 |drawit-brush|  ||
	||              |                                                           ||
	||==============+===========================================================++
	||Visual Cmds   | Explanation                                               ||
	||--------------+-----------------------------------------------------------++
	||              | The drawing mode routines use visual-block mode to        ||
	||              | select endpoints for lines, arrows, and ellipses. Bresen- ||
	||              | ham and Bresenham-like algorithms are used for this.      ||
	||              |                                                           ||
	||              | These routines need a block of spaces, and so the canvas  ||
	||              | routine must first be used to create such a block.  The   ||
	||              | canvas routine will query the user for the number of      ||
	||              | lines to hold |'textwidth'| spaces.                         ||
	||              +-----------------------------------------------------------++
	|| \a           | draw arrow from corners of visual-block selected region   ||  |drawit-a|
	|| \b           | draw box on visual-block selected region                  ||  |drawit-b|
	|| \c           | the canvas routine (will query user, see above)           ||  |drawit-c|
	|| \e           | draw an ellipse on visual-block selected region           ||  |drawit-e|
	|| \f           | flood figure with a character (you will be prompted)      ||  |drawit-f|
	|| \l           | draw line from corners of visual-block selected region    ||  |drawit-l|
	|| \s           | spacer: appends spaces up to the textwidth (default: 78)  ||  |drawit-s|
	||              |                                                           ||
	++==============+===========================================================++



enjoy the life !!!
