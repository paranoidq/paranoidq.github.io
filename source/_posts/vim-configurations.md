title: Vim配置  
date: 2015-12-25 22:55:51
tags: [linux, vim]
categories: linux
---


### 文件浏览

NERDTree 插件：`Plugin 'scrooloose/nerdtree'`
如果你想用tab键，可以利用vim-nerdtree-tabs插件实现：`Plugin 'jistr/vim-nerdtree-tabs'`

打开文件浏览：`:NERDTreeDTreeTabs on console vim startup, put this into .vimrc:

<!--more-->

To run NERDTreeTabs on console vim startup, put this into .vimrc: (自动打开NERDTree)
```
let g:nerdtree_tabs_open_on_console_startup=1
```




