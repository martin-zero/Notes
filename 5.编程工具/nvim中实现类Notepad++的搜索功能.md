---
tags:
  - neovim
---
	Notepad++的搜索功能在日常工作中查看日志排查问题时非常方便，但是由于Notepad++只有Win版本，通过偶然的机会了解到vimgrep命令，所以想在nvim中实现Notepad++的搜索功能。

## vimgrep
`vimgrep` 是 Vim/Neovim 自带的“正则全文搜索命令”，用于在一个或多个文件中查找内容，并把结果放进 QuickFix（或 Location List）里。

>[!note] 命令格式
>vim 关键字 文件路径

关键字：想要搜索的关键字，将关键字用一对`/`括起来可以进行正则匹配。
文件路径：关键字的搜索范围，若为当前文件则可以使用%，或者使用`*.c`范围匹配。

