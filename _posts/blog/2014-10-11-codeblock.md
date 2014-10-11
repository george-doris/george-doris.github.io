---
layout:     post
title:      codeblocks 使用研究
category: blog
description: 给学弟出了一道题，告诉你n个数，其中只有一个数出现一次，其他的数都出现三次。求出现一次的那个数。
---

## 使用前

使用 codeblocks 前肯定是下载安装 codeblocks 了。

我们可以直接去官网下载 [这里][codeblocks-downloads-windows]。

选择 codeblocks-13.12mingw-setup.exe 下载安装即可。


## 使用时

### 新建文件

我的习惯是直接新建一个文件,然后把模板代码粘进去，保存为cpp文件。

新建文件快捷键：ctrl-shift-n

### 设置代码格式

codeblocks 的默认代码风格和我的代码风格不同，所以我需要修改成我的代码风格。

点击菜单栏的 Setting-> 下拉的editor -> 左侧的Source formatter -> 右侧的Java -> 下侧的 ok.

### 自动提示

敲代码，没有智能提示怎么行呢。

点击菜单栏的 Setting-> 下拉的editor -> 左侧的Code completion

右侧选中 Automatically launch when typed # letters , 后面的输入框内填入1.

### 手动提示

有时候自动提示时没有选需要选的内容，然后提示下拉消失了，这时就需要手动开启提示了。

快捷键：shift - 空格


### 快速注释

选中可若干行，肯定有快捷注释的快捷键了。

快捷键： ctrl-shift-c .


### 快速取消注释

有快速注释了，肯定有快速取消注释了。

快捷键: ctrl-shift-x .


### 快速格式化

默认 codeblocks 可以通过鼠标右键，format use Astyle 来格式化代码。

但是如果我们把它配置成快捷键就爽了。


点击菜单栏的 Setting-> 下拉的editor -> 左侧的keyboard shortcuts中可以配置快捷键。

我们点击plugins,在展开项里可以看到Source code formatter(Astyle), 此时右侧的Current shortcuts 中应该是空的。

我们在下部的 new shortcuts 中按 ctrl-shift-f ,然后点击 add 按钮， 最后点击确定。

这样我们就自定义出格式化的快捷键了。



[codeblocks-downloads-windows]: http://www.codeblocks.org/downloads/26#windows