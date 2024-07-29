---
title: "One-Year Summary of Using VIM"
description: "This article is a personal summary of my experience using VIM after one year."
publishDate: "29 Jul 2024"
tags: ["tutorial", "vim", "opensource"]
---

## Introduction

This article is a personal summary of my experience using VIM after one year. It includes the story of how I started with VIM, beginner tips, my frequently used VIM shortcuts, and a share of my ideavim configuration. I hope it will help readers get started with and make better use of VIM.

## My Journey with VIM

The first time I encountered VIM was during my Linux studies. Following the tutorial commands, I used VIM to edit a file and initially found this editor to be quite user-unfriendly. The various strange modes and commands made my head spin.

Around this time last year, VIM’s author, Bram Moolenaar, passed away. After reading an article that shared the story of Bram Moolenaar and VIM, I decided to learn this editor created by such a great programmer.

Today, one year later, VIM has become an indispensable part of my daily development: in my browser, for note-taking, and in my IDE.

## How to Get Started

Many people mention that VIM has a very steep learning curve, but I didn't encounter too many insurmountable difficulties when I started. Instead, as you gradually begin to use this editor and encounter points that feel "awkward," you can learn more shortcuts and configurations to make VIM fit your personal habits rather than forcing yourself to use methods you're uncomfortable with.

To get started, VIM Tutor was the method I initially used. It is a classic tutorial built into VIM.

If you use VIM, you can open it with the following command:

```shell
vimtutor
```

If you use Neovim, you can open it in Neovim's Normal mode with the following command:

```shell
:Tutor
```

VIM Tutor provides an interactive tutorial that takes about 30 minutes and covers the most commonly used shortcuts and modes in VIM.

Once you are familiar with and remember most of the content in VIM Tutor, I believe you will be able to "survive" in VIM. Next, you can read some blogs introducing VIM usage. These blogs usually include some very useful shortcuts not covered in VIM Tutor and basic tutorials, helping you build your own shortcut library.

Here are some excellent blogs that helped me when I started, which you can refer to:

- [Learn Vim Progressively](https://yannesposito.com/Scratch/en/blog/Learn-Vim-Progressively/)

- [Vim Best Practices For IDE Users](https://medium.com/better-programming/50-vim-mode-tips-for-ide-users-f7b525a794b3)

If you find these blogs insufficiently official and want to learn in a more formal way, I suggest reading the VIM User Manual. It is written by Bram Moolenaar and community contributors and is very comprehensive and detailed. You can open it in VIM or Neovim's Normal mode with the following command:

```shell
:help user-manual
```

## Common Shortcuts

This section mainly shares some shortcuts I commonly use in VIM. I hope you find them helpful.

- `ci"`: Delete all content inside the quotes and enter Insert mode.

These kinds of shortcuts are very convenient when you need to operate on content within various types of quotes or parentheses. You can also use other operators to build similar shortcuts, such as:

- `yi(`: Copy all content inside the parentheses.
- `va'`: Select the single quotes and all content inside them, entering Visual mode.
- `di<`: Delete all content inside the angle brackets.

For easier understanding and memorization, `i` stands for "inner," meaning to operate on the content inside the quotes or parentheses, and `a` stands for "around," meaning to operate on the content around the quotes or parentheses.

---

- `daw`: Delete the word under the cursor.

This type of shortcut is very useful when you need to operate on a word, sentence, or paragraph. Since commands like `cw` and `dw` need to be used at the beginning of a word to operate on the whole word, you might have to use shortcuts like `b` and `w` to move to the beginning first. The `daw` command allows you to operate on the entire word as long as the cursor is within the word, making it very convenient. Similar shortcuts include:

- `caw`: Delete a word and the space after it, entering Insert mode.
- `das`: Delete a sentence.
- `cap`: Delete a paragraph and enter Insert mode.

---

- `zz`: Center the current line in the middle of the screen.
- `zt`: Move the current line to the top of the screen.
- `zb`: Move the current line to the bottom of the screen.

---

- `H`: Move the cursor to the top line of the screen.
- `M`: Move the cursor to the middle line of the screen.
- `L`: Move the cursor to the bottom line of the screen.

---

- `<C-y>`: Scroll the content of the current window up one line while keeping the cursor position unchanged.

- `<C-e>`: Scroll the content of the current window down one line while keeping the cursor position unchanged.

---

- `<C-w>v`: Split the current window vertically.
- `<C-w>s`: Split the current window horizontally.
- `<C-w>w`: Switch between windows.
- `<C-w>c`: Close the current window.
- `<C-w>h`: Move focus to the window on the left.
- `<C-w>j`: Move focus to the window below.
- `<C-w>k`: Move focus to the window above.
- `<C-w>l`: Move focus to the window on the right.

## IDEAVIM Configuration

As a JetBrains user, after learning VIM, I configured the ideavim plugin to enjoy the convenience of both JetBrains IDE and VIM in my IDE. Below is a set of ideavim configurations tailored to my personal habits. I hope it can provide you with some reference:

You can also find it here: [https://github.com/justlorain/euphonium](https://github.com/justlorain/euphonium)

```vim
" Vim mode toggle
nmap <leader>vim <Action>(VimPluginToggle)

" --- Basic Configuration ---

" leader key
let mapleader = " "

" Move to the previous/next line when pressing h/l at the beginning/end of a line
set whichwrap=b,s,<,>,h,l,[,]

"" visual shifting (builtin-repeat)
vnoremap < <gv
vnoremap > >gv

" Vertical scroll offset
set scrolloff=5

" Search
set incsearch
set nohls
set ic
set smartcase
nnoremap <leader>ss :set invhlsearch<CR>

" Clipboard mapping
set clipboard+=unnamed

" Show line numbers
set number
" Set relative line numbers
set relativenumber

" Don't use Ex mode, use Q for formatting.
map Q gq

" --- Plugin Configuration ---

" Highlight copied text
Plug 'machakann/vim-highlightedyank'
" Commentary plugin
Plug 'tpope/vim-commentary'
" vim-surround
set surround
" easymotion
set easymotion
" nerdtree
set NERDTree
nnoremap <leader>nf :NERDTreeFind<CR>
" quickscope
set quickscope
let g:qs_highlight_on_keys = ['f', 'F', 't', 'T']
" which-key
set which-key
set notimeout

" --- Coding Configuration ---

" Show file structure
let g:WhichKeyDesc_FileStructure = "<leader>fs FileStructure"
nmap <leader>fs <action>(FileStructurePopup)
let g:WhichKeyDesc_FindFile = "<leader>ff FindFile"
nmap <leader>ff <action>(GotoFile)
" Close tab
let g:WhichKeyDesc_CloseCurrentTab = "<leader>xx CloseCurrentTab"
nmap <leader>xx <action>(CloseContent)
let g:WhichKeyDesc_CloseOtherTabs = "<leader>xo CloseOtherTabs"
nmap <leader>xo <action>(CloseAllEditorsButActive)
let g:WhichKeyDesc_CloseAllTabsOnTheLeft = "<leader>x[ CloseAllTabsOnTheLeft"
nmap <leader>x[ <action>(CloseAllToTheLeft)
let g:WhichKeyDesc_CloseAllTabsOnTheRight = "<leader>x] CloseAllTabsOnTheRight"
nmap <leader>x] <action>(CloseAllToTheRight)
" Scroll page
let g:WhichKeyDesc_EditorScrollUp = "<C-k> EditorScrollUp"
nmap <C-k> <action>(EditorScrollUp)
let g:WhichKeyDesc_EditorScrollDown = "<C-j> EditorScrollDown"
nmap <C-j> <action>(EditorScrollDown)
" Go to definition or reference
let g:WhichKeyDesc_GotoDeclaration = "gd GotoDeclaration"
nmap gd <action>(GotoDeclaration)
" Go to usage
let g:WhichKeyDesc_FindUsages = "<leader>gr FindUsages"
nmap <leader>gr <action>(FindUsages)
" Go to superclass
let g:WhichKeyDesc_GotoSuperMethod = "<leader>gs GotoSuperMethod"
nmap <leader>gs <action>(GotoSuperMethod)
" Go to implementation
let g:WhichKeyDesc_GotoImplementation = "<leader>gi GotoImplementation"
nmap <leader>gi <action>(GotoImplementation)
" Jump to method
let g:WhichKeyDesc_MethodUp = "<M-k> MethodUp"
nmap <M-k> <Action>(MethodUp)
let g:WhichKeyDesc_MethodDown = "<M-j> MethodDown"
nmap <M-j> <Action>(MethodDown)
let g:WhichKeyDesc_ExtractMethod = "<leader>em ExtractMethod"
vmap <leader>em <Action>(ExtractMethod)
" Jump tab
let g:WhichKeyDesc_PreviousTab = "<M-h> PreviousTab"
nmap <M-h> <action>(PreviousTab)
let g:WhichKeyDesc_NextTab = "<M-l> NextTab"
nmap <M-l> <action>(NextTab)
" Translation
let g:WhichKeyDesc_EditorTranslate = "<leader>t EditorTranslate"
vmap <leader>t <action>($EditorTranslateAction)
" Cursor back
let g:WhichKeyDesc_Back = "<C-i> Back"
nmap <C-i> <action>(Back)
" Cursor forward
let g:WhichKeyDesc_Forward = "<C-o> Forward"
nmap <C-o> <action>(Forward)
" Open recent project
let g:WhichKeyDesc_OpenRecentProject = "<leader>p OpenRecentProject"
nmap <leader>p <action>($LRU)
" Replace
let g:WhichKeyDesc_ReplaceInFile = "<leader>rif ReplaceInFile"
nmap <leader>rif <action>(Replace)
vmap <leader>rif <action>(Replace)
let g:WhichKeyDesc_ReplaceInProject = "<leader>rip ReplaceInProject"
nmap <leader>rip <action>(ReplaceInPath)
vmap <leader>rip <action>(ReplaceInPath)
" Find
let g:WhichKeyDesc_FindInFile = "<leader>fif FindInFile"
nmap <leader>fif <action>(Find)
vmap <leader>fif <action>(Find)
let g:WhichKeyDesc_FindInProject = "<leader>fip FindInProject"
nmap <leader>fip <action>(FindInPath)
vmap <leader>fip <action>(FindInPath)
" New line
let g:WhichKeyDesc_NewLine = "<M-o> NewLine"
nnoremap <M-o> :normal o<CR>
" Toggle breakpoint
let g:WhichKeyDesc_ToggleLineBreakpoint = "<leader>bb ToggleLineBreakpoint"
nmap <leader>bb <action>(ToggleLineBreakpoint)
" Show expression type
let g:WhichKeyDesc_ExpressionTypeInfo = "<leader>et ExpressionTypeInfo"
nmap <leader>et <action>(ExpressionTypeInfo)
" Show method parameters
let g:WhichKeyDesc_ParameterInfo = "<leader>et ParameterInfo"
nmap <leader>pp <action>(ParameterInfo)
" Recent files
let g:WhichKeyDesc_RecentFiles = "<leader>ee RecentFiles"
nmap <leader>ee <action>(RecentFiles)

sethandler <C-j> a:vim i:ide
sethandler <C-k> a:vim i:ide
```

## Summary

That's all for this article. I hope this personal summary helps you use VIM more effectively.

Once again, thanks to Bram Moolenaar for bringing us such a wonderful software. R.I.P.

If there are any mistakes or issues, feel free to comment or message me privately. Thank you.

## Reference

- https://yannesposito.com/Scratch/en/blog/Learn-Vim-Progressively/
- https://medium.com/better-programming/50-vim-mode-tips-for-ide-users-f7b525a794b3
- https://github.com/justlorain/euphonium