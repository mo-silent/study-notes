# Linux 的 vimrc 配置解析
```bash
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 显示相关
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
set encoding=utf8               " vim 内部编码
set fileencodings=utf8          " 当前编辑的文件的字符编码方式
set fileencoding=utf8           " 探测即将打开的文件的字符编码方式, 并且将 fileencoding 设置为最终探测到的字符编码方式
set termencoding=utf-8          " Vim 所工作的终端的字符编码方式，通常不需要改变
set number                      " 显示行号
set tabstop=2                   " 一个tab键显示为 2 空格，只是看起来是2空格
set hlsearch                    " 搜索逐字符高亮
set incsearch                   " 搜索逐字符高亮
set showmatch                   " 高亮显示匹配的括号
set showmode                    " 命令行显示vim当前模式
set showcmd                     " 输入的命令显示出来
set scrolloff=5                 " 光标移动到vim窗口的顶部和底部时保持5行距离
set wrap                        " 一行不能显示完成时，自动换行显示（还是同一行数据）
autocmd InsertLeave * se nocul  " 用浅色高亮当前行，离开时关闭高亮
autocmd InsertEnter * se cul    " 用浅色高亮当前行，进入是开启高亮
set cursorline                  " 突出显示当前行
set backspace=2                 " 使回格键（backspace）正常处理indent, eol, start等
syntax on                       " 语法高亮
set ruler                       " 打开状态栏标尺
if version >= 603
    set helplang=cn             " 显示中文帮助
endif
set statusline=%F%m%r%h%w\ [FORMAT=%{&ff}]\ [TYPE=%Y]\ [POS=%l,%v][%p%%]\ %{strftime(\"%d/%m/%y\ -\ %H:%M\")}   "状态行显示的内容
set laststatus=1                " 启动显示状态行(1),总是显示状态行(2)
set novisualbell                " 警告响铃时不要闪烁
set go=                         " 去掉图形按钮
colorscheme koehler             " 应用 koehler 主题
set cmdheight=2                 " 命令尾行的高度


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
""实用设置
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
set shortmess=atI               " 启动的时候不显示那个援助乌干达儿童的提示
set paste                       " 进入 paste 模式，防止粘贴代码格式错乱
set expandtab                   " 使用空格代替制表符
set softtabstop=2               " 统一缩进为2
set shiftwidth=2                " 统一缩进为2
set autoindent                  " 使用自动对齐，也就是把当前行的对齐格式应用到下一行
set cindent                     " 使用 C 语言风格的缩进
set ignorecase                  " 忽略大小写的查找
set wildmenu                    " 尾行模式中自动补全命令
set textwidth=79                " 设置最大文本宽度
set nocompatible                " 去掉讨厌的有关 vi 一致性模式
set history=1000                " 历史记录数
set mouse=r                     " 默认 a 模式下，鼠标左键无法复制
set nofoldenable                " 关闭折叠功能
set foldmethod=manual           " 手动折叠
set noeb                        " 去掉输入错误的提示声音
set nobackup                    " 禁止生成备份
set noswapfile                  " 禁止生成临时文件
set autowrite                   " 自动保存
if has("autocmd")               " 记录上次编辑/查看位置
  au BufReadPost * if line("'\"") > 1 && line("'\"") <= line("$") | exe "normal! g'\"" | endif
endif
```