---
layout: post
title: My Vim setup.
permalink: /2011/12/04/my-vim-setup.html
categories:
- blog
---

I've finally come around to documenting how I customized vim to be my primary IDE. When coding I use vim about 90% of the time when writing the code itself.  The rest of the time I use a dedicated IDE - primarely for refactoring, file creation, and automatic imports. For all the beef - editing code, or nontrivial text transformations it's Vim all the way.

The biggest and most important thing about vim is mapping caps lock to the escape key you can do this quite `xmodmap -e 'clear Lock' -e 'keycode 0x42 = Escape'`. This alone is a big productivity booster - switching out of input mode using the escape key can be unwieldy at times.

The second thing is being consistent, although you may say it is taking it too far but I like to the readline options of my shell to be vi-like (adding `set -o vi` in my `.profile`), I use the [vimperator](http://vimperator.org/vimperator) plug-in for browsing and use a tiling window manager called [awesomewm](http://awesome.naquadah.org/) for my daily tasks which lets me be "in the mode" almost all the time. With this setup I don't really need to touch the mouse at all (or almost at all), some applications also support vi like navigation and search - try it some times I was amazed that evince supported `hjkl` navigation and finding by `/` (although I was displeased that `^b` and `^g` don't work the way they should).

vimrc 
====

The beginning of my vimrc is quite vanilla, just setting the nocompatible mode,
enabling file type detection and syntax.

{% highlight vim %}
set nocompatible 
filetype plugin on
syntax on
{% endhighlight %}

Normal mode maps.
-----------------

Next come my normal maps: I like to switch around buffers using similar
mappings as in normal mode I map `^h` to switch to the left buffer and `^l` to
switch to the right buffer. My `<Leader>` is (left) as `\` and I use `\d` to
delete the current buffer. 

Some of my toggles include: 
* line numbering with `<F2>`
* incremental search with `<F6>`. I leave incremental search mostly on, but when I'm distracted I'm and feeling ``dizzy'' from the amount of information I like to turn of incremental search to lessen the cognitive load of keeping track of the cursor.
* search highlight `<F7>` mostly I find it useful to have search highlight on when there isn't a lot of tokens, or when dealing with similar structures (like searching for semicolons in a java file) to make sure everything matches
* spellchecker with `\s`
* cd to the current file with `\cd` - I use this a *lot* with the combination of vimgrep or manually editing a file by feeding vim the full path.

Finally I use `<F9>` to call make and `<F10>`, `<F11>` to switch between previous and next errors.

{% highlight vim %}
"-----------------------------------------------
" normal mode maps 
"-----------------------------------------------

" scroll to next buffer
map <C-h> :bprev<CR>

" scroll to previous buffer
map <C-l> :bnext<CR>

" delete buffer
map <Leader>d :bd <CR>

" Line numbering
nmap <silent> <F2> :set number! <CR>

" Incremental search 
noremap <F6> :set incsearch!<CR> 

" Highlight search
noremap <F7> :set hls!<CR>

" Spelling 
nmap <silent> <Leader>s :set spell!<CR>

" Change directory to the current file
nmap <silent> <Leader>cd :cd %:h<CR>

" compiling 
" <F9> make
" <F10> next error
" <F11> previous error

noremap <F9> <Esc>:mak<CR>
noremap <F10> <Esc>:cnext<CR>
noremap <F11> <Esc>:cprev<CR>

" Disable annoying help
map <F1> <Esc>
{% endhighlight %}


Input mode maps.
----------------

I don't have much input mode maps I'll go into detail what SuperCleverTab does later. Apart from that the most important mapping is to `^S`. It inserts my signature with the current date - quite handy when writing FIXME or TODO notes.

{% highlight vim %}
""""""""""""""""""""
" input mode maps  "
""""""""""""""""""""

" Disable annoying help
imap <F1> <Esc> 

" Tab-completion 
inoremap <Tab> <C-R>=SuperCleverTab()<CR>

" Time stamp
inoremap <C-S> <C-R>=strftime("(%Y-%m-%d Bartosz Witkowski)")<CR>

" compiling 
" <F9> make
" <F10> next error
" <F11> previous error

inoremap <F9> <Esc>:mak<CR>
inoremap <F10> <Esc>:cnext<CR>
inoremap <F11> <Esc>:cprev<CR>
{% endhighlight %}

Visual mode maps.
-----------------

The only thing I map in visual mode is disabling the default behavior that shifts in visual mode turn off the visual highlight.

{% highlight vim %}
"""""""""""""""""""""
" visual mode maps  "
"""""""""""""""""""""

" Disable unhighlighting after shift
vnoremap > ><CR>gv
vnoremap < <<CR>gv
{% endhighlight %}

Settings.
---------

Some options that I like to set as on by default are incremental search, search highlighting, syntax coloring, wild menu (wild menu shows alternatives when tabbing through alternatives of files, settings etc).

Although I have spell checking disabled by default I like to have English as as the language and a maximum of 5 spelling suggestions when using.

In case you didn't know when enabling the spell checker Vim will underlined the misspelled world in red and by pressing `z=` the possible spelling suggestions will appear. You can traverse misspelled words by using `[z` and `]z` which will put your cursor on the previous and next misspelled word respectively.

One of the important mappings for me is adding the "unnamed" parameter to the clipboard option - it causes vim to yank every string to the unnamed (`*`) register (the register which is shared with the system clipboard), with the combination of the `a` flag of guioptions all text highlighted in visual mode is also yanked to `*` which comes in handy to keep consistency with other X applications and the default "select to copy" behavior.

Color schemes and gui options.
------------------------------

The most readable color scheme for me turned out to be the desert color scheme. The desert is one of the default color schemes ones binded with Vim. The low contrast and grayish tones really help to calm the tired eyes.  As a font **DejaVu Sans Mono** is super readable and crisp the difference between Il1 and O0 really stand out so I chose it as my font.

I disable almost all gui options in vim leaving only the [`i`]con and, [`a`]utoselect yanking. The `f` flag stands for foreground I have it set so that vimperator (a plugin for that firefox that I like which that emulates vim behavior)- you can learn more about this option reading the vim help file (`:help guioptions`).

Check out how the default look and feel compares to my custom one below:

Default color scheme, and options.

![vim normal](/images/vim-normal.png)

The desert color scheme with my custom options.

![vim custom](/images/vim-custom.png)

{% highlight vim %}
" incremental search
set incsearch 

" default text width
set textwidth=80

" keep at least 5 lines above/below
set scrolloff=5    

" 1000 is a nice number
set undolevels=1000

" TODO: disable in ssh
set ttyfast

" default spell checker language
set spelllang=en_us

" spelling suggestions
set spellsuggest=5

" Yanks go to clipboard
set clipboard+=unnamed

" highlighting 
set hls 

" Set syntax highlighting as default 
syntax enable

" scrollable menu
set wildmenu 

" exporting to html
let use_xhtml         = 1
let html_number_lines = 0
let html_use_css      = 1
let html_no_pre       = 1

" gui
if &guioptions =~# 'T'
	" color scheme 
	colorscheme desert
	" font
	set gfn=DejaVu\ Sans\ Mono\ 8
	set cursorline
	" guioptions
	set guioptions=ai
	" make vimperator behave nicely
	set guioptions+=f
endif

if has("multi_byte")
   if &termencoding == ""
		let &termencoding = &encoding
	endif
	set encoding=utf-8jkk
	setglobal fileencoding=utf-8
	set fileencodings=uds-bom,utf-8,latin1
endif
{% endhighlight %}

File specific options.
======================

Vim has wonderful support for many file types with its various syntax files, additionally you can set text width, tab size, and if tabs should be expanded to spaces for every file format that vim knows about. The parameter `tw` controls the text width (how many columns a line has), shift width `sw` determines how many columns will the text "shift" either using the `<<` `>>` commands or by auto indentation. The `ts` or tab stop parameter controls how many spaces an `<Tab>` button inserts. The tabs can be expanded into spaces using the `expandtab` option. I set `ts` the same as `sw` and I like to set all options only local to the current buffer using the `setlocal` command instead of the usual `set`.

| File type | Text width | Tab space  | Expand tab |
|-----------|:-----------|------------|:-----------|
| asm       | 80         | 3          | no         |
| cpp, java | 80         | 4          | yes        |
| make      | 80         | 8          | no         |
| scala     | 110        | 2          | yes        |
| sql/sh    | 80         | 2          | no         |
| tex       | 80         | 3          | no         |

java
----

* java `matchpairs+=<:>` adds pair matching to of triangle brackets to
highlighting.
* `makeprg=ant\ -emacs` 

python
------

* `autocmd FileType python noremap <F9> <Esc>:!python %<CR>`

gnuplot
-------

* `autocmd BufNewFile,BufRead *.gp noremap <F12> <Esc>:!gnuplot -persist % <CR>`

plugins
=======

minibufexplorer
---------------

This plugin provides an list of buffers in a separate window (a vim window that is) located at the top. For each buffer it's number and corresponding filename is displayed which in addition to mappings to `:bprev` and `:bnext` makes buffer navigation a snap.

bccalc 
------

Bcccalc adds its mapping `<Leader>bc` which sends the current line to `bc` (the unix calculator) which enables arbitrary calculations straight from vim.  Entering the equations `2 + 2` and pressing `<Leader>bc` will show the result `4` in the status line adding the equals sign `2 + 2 =` pressing `<Leader>bc` adds the result to the current line.

camelcasemotion
---------------

Adds mappings to various vim-motions enabling vim-motions with camel case words - `,w` will move to the next camel case word `,b` will move to the previous and `,e` will move to the end of the camel case word. The movements work normally so `d,w` will delete the first camel case word.

vicle
-----

Vicle provides `screen` and `tmux` integration to vim and I use it when programming in languages that provide a repl (lisp, scala, bash, sql). `^c^c` sends the current line to the screen session in normal mode and in visual mode it sends the whole highlighted text. Very useful especially with conjunction with vim marks.

supertab
--------

Supertab maps the `<tab>` button to the usual `^x` vim-completions (omni-, dictionary-, file etc) and toggles through the searches, you can change the current mode of completion (eg. from dictionary to file) by using the usual vim completion, you can lookup the completion modes by hitting `:SuperTabHelp`.

vimcommander
------------

Remember {norton,midnight} commander? Vimcommander lets you navigate, copy, remove files with vim. It's awesome for quick file manipulation without need to switch windows or fire up a new console. The beef I have with this plugin is that I couldn't find any way to invoke a shell command in the current directory of vimcommander.

git-vim
-------

A git plugin for vim. Unfortunately it isn't so well rounded, particularly, adding and removing files in `:GitStatus` seems kind of buggy, I hadn't tried fugitive but if git-vim will irk me enough I'll check it out. 

Apart from some usability shortcomings the rest of the plugin is quite solid.  `:GitBlame` opens an additional window on the left which shows the input of `git blame %` (sans the text content), `:GitDiff` (mapped to `<Leader>gd) shows the diff of the current file which is particularly nice with vim syntax highlighting. 

`:GitStatus` or `<Leader>gs` shows everything that `git status` would, additionally you can navigate the entries and by pressing `<CR>` you can `git add` a commit, I think I modified the plugin a bit so that pressing `<CR>` on a removed entry will do `git rm`. 

`:GitCommit`/`<Leader>gc` works as expected - giving you a screen ready to edit the commit message. After writing and closing buffer ale changes will be committed.

Fuzzy Finder
------------

For me fuzzy-finder provides two very useful mappings, one `:FufFile` loads a special buffer that lets you navigate the file system with ``fuzzy'' completion assuming a file path `/etc/apache2/conf.d/javascript-common.conf` - `:FufFile` `/e<CR>a2<CR>c<CR>j<CR>` will go to that file. The fuzzy-finder will match any partial patterns along the way.

`:FufBuffer` will let you match any pattern in a buffer with the same mechanism as fuzzy file finding.

NERD tree
---------

I left the best for the last, NERD tree lets you navigate the filesystem showing it as a tree structure it has mappings to "reroot" the tree structure using `c` opening files with new tabs, and other numerous mappings.

`<Leader>n` opens nerd tree other mappings can be looked up at anytime using `?` when in the nerd-tree buffer.
