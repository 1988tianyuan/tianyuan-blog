title: vim快捷键（1）
tags:
  - vim
categories:
  - 工具使用
author: 天渊
date: 2019-01-21 11:18:00
---
vim编辑器有三个模式：一般模式，编辑模式，命令模式：
<!--more-->

- 一般模式：默认模式，可以新增删除复制粘贴，按Esc从当前模式转换到普通模式
- 编辑模式：按i,o,a等字符进入编辑模式，可以编辑文本内容
- 命令模式：按:,/,?三个字符中的一个进入命令模式，可以读取、查找数据、大量替换字符等操作

### 基本操作
vi+文件名 进入文档，按命令键进入编辑或者命令模式，Esc回到一般模式（命令模式和编辑模式不能相互转换），输入:w保存文档，输入:wq保存并离开文档，使用:wq!在没有权限的情况下强制写入，输入:q不保存并退出

#### 文本浏览

- 重新载入文件：

  ```shell
  :e
  :e! #放弃当前修改，强制重新载入
  :bufdo e 或者 :bufdo :e! #重新载入所有打开的文件
  ```

- 光标移动

  ```shell
  0  #数字0）移动光标至本行开头
  $  #移动光标至本行末尾
  ^  #移动光标至本行第一个非空字符
  w  #向前移动一个词
  W  #向前移动一个词 （以空格分隔的词）
  5w  #向前移动5个词
  b  #向后移动一个词
  B  #向后移动一个词 （以空格分隔的词）
  5b  #向后移动5个词
  G  #移动至文件末尾
  gg #移动至文件开头
  ```

- 搜索和替换

  ```shell
  :/search_text  #在文档后面部分检索search_text这个内容
  :?search_text  #在文档前面部分检索search_text这个内容
  n  #移动到后一个检索结果
  N  #移动到前一个检索结果
  :%s/original/replacement  #将第一个检索到的original替换为replacement
  :%s/original/replacement/g	#将所有original替换为replacement
  :%s/original/replacement/gc	 #将所有original替换为replacement，但会先询问
  ```