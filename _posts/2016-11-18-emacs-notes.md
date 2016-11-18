---
layout: post
title: emacs 笔记
description: emacs使用笔记
modified: 2016-11-18
tags: [emacs, elisp, 代理] 
---

# emacs 设置代理 #
```lisp
    (setq url-proxy-services
       '(("no_proxy" . "^\\(localhost\\|10.*\\)")
         ("http" . "proxy.com:8080")
         ("https" . "proxy.com:8080")))
```
