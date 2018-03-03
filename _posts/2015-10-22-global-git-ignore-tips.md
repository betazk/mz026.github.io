---
layout: post
title: global git ignore 小撇步
comments: true
tags:
  - git
  - gitignore
  - 我要成為效率王
---

在和別人合作開發 project 的時候，常常會有一些只有自己想要 git ignore 的檔案，
像是每個 project 的 `.vimrc`，或是編輯器的暫存檔 (`.swp`, `.swo`)，或是自己要看的筆記等等。

這些檔案有時候不太適合 check in 進去 repository，好比說並不是每個人都有用 [Vim](http://mz026.logdown.com/posts/216100-vim-and-i)，那這樣把 `.vimrc` check in 進去就不太合理。

這時候可以在家目錄的 `.gitconfig` 檔裡面的 `core.excludesfile` 來指定一個 user-based 的 `.gitignore`。

例如說：

```
# ~/.gitconfig
[core]
	excludesfile = ~/.gitignore_global
```

```
# ~/.gitignore_global
*.swo
*.swp
.vimrc
_todo*
_note*
```

這樣就可以不用在每個 project 都 ignore 這些檔案，同時又可以避免他們被不小心 check in 進去惹。
