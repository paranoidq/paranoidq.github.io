title: Vim配置  
tags: [linux, vim]
categories: linux
---


### 文件浏览

NERDTree 插件：`Plugin 'scrooloose/nerdtree'`
如果你想用tab键，可以利用vim-nerdtree-tabs插件实现：`Plugin 'jistr/vim-nerdtree-tabs'`

<!--more-->

一些设置：
```
let g:nerdtree_tabs_open_on_console_startup=1           " 开启vim即打开nerdtree
nnoremap <silent> <F4> :NERDTreeToggle<CR>              " F4打开关闭nerdtree
let NERDTreeIgnore=['\.pyc$', '\~$', 'node_modules', '.*']    " 不显示的文件
let NERDTreeHightCursorline=1                           " 高亮显示当前文件或目录
let NERDTreeShowLineNumbers=1                           " 显示行号
let NERDTreeWinPos='left'                               " 窗口位置
let NERDTreeWinSize=31                                  " 窗口宽度

" git-nerdtree增强
let g:NERDTreeIndicatorMapCustom = {
    \ "Modified"  : "✹",
    \ "Staged"    : "✚",
    \ "Untracked" : "✭",
    \ "Renamed"   : "➜",
    \ "Unmerged"  : "═",
    \ "Deleted"   : "✖",
    \ "Dirty"     : "✗",
    \ "Clean"     : "✔︎",
    \ "Unknown"   : "?"
    \ }



```



### 参考
[https://www.sdk.cn/news/1344](https://www.sdk.cn/news/1344)
[http://codingpy.com/article/vim-and-python-match-in-heaven/#rd?sukey=fc78a68049a14bb27b8377074f320e3640973a7b44a89bf8e337dea7195bc10f379ccd4ed115dd80a316467379eec995](http://codingpy.com/article/vim-and-python-match-in-heaven/#rd?sukey=fc78a68049a14bb27b8377074f320e3640973a7b44a89bf8e337dea7195bc10f379ccd4ed115dd80a316467379eec995)

