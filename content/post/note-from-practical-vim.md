+++
title = "Practical Vim 读书笔记"
date = "2015-01-12"
slug = "2015/01/12/note-from-practical-vim"
Categories = []
+++

- The Vim Way
    1. `.`命令可以重复做上一次的改变(repeat last change)。这里关键是要理解这个改变的意思，就我现在的理解，这个改变是不包括光标位置的移动的，只是指对文本内容的改变，比如删除 增加等等。理解这一点很重要。
    2. `;` 分号，配合f使用，移动到下一个匹配的地方，反方向移动为`,`
    3. `*` 移动到下一个与当前光标处文本匹配的地方，并高亮显示所有匹配文本
- Normal Mode
    1. `aw`  "a word" `daw`可删除整个单词，不管光标在哪里
    2. `<C - a>`和`<C - x>` commands perform addition and subtraction on numbers.
    3. `g~` swap case 改变大小写    
    `gu` make lowercase    
    `gU` make uppercase    
    4. `=` autoindent  自动缩进
- Insert Mode.   Most of Vim's commands are triggered from other modes, but some functionality is within easy reach from Insert mode.
    1. `<C - h>` Delete back one character
    2. `<C - w>` Delete back one word
    3. `<C - u>` Delete back to start of line
    4. `<C - O>`  execute one command, return to Insert mode. 所谓的Insert Normal Mode，在insert模式下进入此模式后可以执行一次normal模式下才能执行的命令后立即返回insert模式，可以配合zz（使光标处在屏幕中间）使用。如在编辑过程中光标处在最上端或者最下端了，此时想要看一看上下文，可以使用`<C - O>zz`，然后又立即可以继续编辑。
    5. `<C - r>{register}` Paste from a Register Without Leaving Insert Mode. For example, `<C - r>0`  不适合大段文本，因为这种情况下文本是一个字符一个字符粘贴过来的。
    6. `<C - r>=`  expression register. 可以用于在寄存器中evaluate表达式，然后把结果返回到当前光标处。
    7. `<C - v>{code}` Insert unusual characters by character code. For example, `<C -v>065`     
    `<C - v>{123}` Insert character by decimal code    
    `<C - v>u{1234}` Insert character by hexadecimal code    
- Visual Mode
    1. `v` -  character-wise Visual mode    
    `V` - line-wise Visual mode    
    `<C - v>` - block-wise Visual mode    
    `gv` - reselect the last visual selection    
    `o` - go to the other end of highlighted text
- Command-Line Mode
    1. `.` 代表当前行 `%` 代表所有行
    2. `:/<html>/,/<\/html>/p`  Specify a range of lines by patterns
    3. `:/<html>/+1,/<\/html>/-1p`Modify an Address using an offset 
    4. `[range]copy {address}`  Duplicate lines  也可以用`:t`  移动行用`:m`
    5. `@:`  Repeating the last Ex command 
    6. `:%normal A;`  Run Normal Mode Commands across a range  在命令行模式下运行normal模式下的命令
    7. `<C - d>`  ask Vim to reveal a list of possible completions  自动补全
    8. `<C -r><C - w>`  Insert the current word at the command prompt
    9. `:%s//<C-r><C-w>/g`  Leaving the search field of the substitute command blank instructs Vim to reuse the most recent search pattern  重用上次的search pattern
   10. words与WORDS的区别:As users, we can think of them in simpler terms: WORDS are bigger than words.  WORDS用W B E gE来操作，且移动的跨度比words大     
   11. `q:` 进入Command-Line Window
   12. `:shell`   The `:!{cmd}` syntax is great for firing one-off commands, but what if we want to run several commands in the shell? In that case, we can use this command to start an interactive shell session.  可以用exit退出，回到vim
   13. `<C - z>`  Putting Vim in the background  用`fg`返回
   14. `:read !{cmd}` 把cmd运行结果读入当前buffer     `:write !{cmd}`  Use the contents of the buffer as standard input for the specified {cmd}.
   15. Filtering the contents of a buffer through an external command     
   The `:!{cmd}`command takes on a different meaning when it's given a range. The lines specified by `[range]` are passed as standard input for the `{cmd}`, and then the output from `{cmd}` overwrites the original contents of `[range]`.
- Manage Multiple Files
    1. `:args`  argument list, represents the list of files that was passed as an argument when we ran the vim command. 
    2. `<C - w>s` divide the window horizontally   `<C - w>v` divide the window vertically. Each time we use these commands, the two resulting split windows will contain the same buffer as the original window that was divided.    
    或者`:sp {filename}` 水平分割打开文件   `:vsp {filename}` 垂直分割打开文件     
    `<C - w>w` cycle between open windows     
    `:cl[ose]` or `<C - w>c`  close the active window     
    `:on[ly]` or `<C - w>o`  keep only the active window, closing all others
    3. `:tabe[dit] {filename}`  open {filename} in a new tab     
    `:tabc[lose]` close the current tab page and all of its windows.    
    `:tabo[nly]`   keep the active tab page, closing all othersa       
    `tabn[ext]`  or gt  Switch to the next tab page
- Open Files and Save Them to Disk
    1. `:edit %:h<Tab>`  Open a File Relative to the active file directory.  The `%` symbol is a shorthand for the filepath of the active buffer. The `:h` modifier removes the filename while preserving the rest of the path
    2. `:e[dit] .` or `:E[xplore]` Open file explorer for the current working directory or the directory of the active buffer
- Navigate Inside Files with Motions
    1. Real lines and Display lines       
    `gj` `gk`  go down/up one display line.      
    `g0` to first character of the display line      
    也就是所有在display line上的操作都是以g开头的
    2. `f{char}` `;` `,`  行内查找字符  重复  反向       
    小技巧： When using the character search commands, it's better to choose target characters with  a low frequency of occurrences. 尽量选出现频率低的
    3. `vi'` `va'`  区别是：第一个是inside，第二个是all. `cit` stands for "change inside the tag"
    4. `iw`与`aw`的区别  `iw` stands for "current word", while `aw` stands for "current word plus one space". 因此，`aw`对象适合用于删除（d）操作，而`iw`适合用于修改（c）操作。
    5. `m{a-zA-Z}` marks the current cursor location with the designated letter. Lowercase marks are local to each individual buffer, whereal uppercase marks are globally accessible. `'{mark}` moves to the line where a mark was set, positioning the cursor on the first non-whitespace character.```{mark}``moves the cursor to the exact position where a mark was set.      
    The `mm` and `'m` commands makes a handy pair. Respectively, they set the mark m and jump to it.
    6. Surround.vim 插件，非常酷～ 一开始装了不起作用，后来发现是跟.vimrc配置中关于fcitx的设置有冲突，只能暂时不用fcitx的配置了。      
- Navigate Between Files with Jumps
    1. motions and jumps.  motions move around within a file, whereas jumps can move between files. jumps可以在文件之间跳转～
    2. `<C - i>`  jump back     
    `<C - o>` jump forth
    3. `(/)` `{/}`  jump to start of previous/next sentence/paragraph
    4. `H/M/L`  jump to top/middle/bottom of screen
- Copy and Paste
    1. vim中使用`dd`或`yy`命令时，被复制或剪切的内容默认是放在未命名寄存器中的（unnamed register），若要指定特定的寄存器，需要`"{register}`，比如要复制内容到a寄存器，需要`"ayiw`，然后粘贴内容需要`"ap`	
    2. 当我们使用复制命令时（`y{motion}`），特定的文本不仅被复制到未命名寄存器中，它同时也被复制到了复制寄存器（yank register），这个寄存器可以用`"0`来引用。
    3. 黑洞寄存器（The Black Hole Register）。 在删除一些内容时，如果确定以后不会再使用它，那么为防止它污染当前的unamed register，可以用`"_`将它放入黑洞寄存器。 If we run the command `"_d{motion}`, then Vim deletes the specified text without saving a copy of it.
    4. 系统寄存器和选择寄存器。 用于Vim外的程序与Vim共享内存。`"+`可以访问到系统中复制、剪切到剪切板的内容，而`"*`是专门针对Linux平台上的X11系统的，在X11系统中，内容只要被选中即被放在了系统剪切板中了，此时在Vim中可用`"*`寄存器访问。
    5. 避免系统剪切板的内容剪切到Vim中出现太多缩进的情况。  这个问题的出现是由于autoindent特性的开启，如果此时需要在Insert模式下插入系统剪切板的内容，一个方法是先开启paste模式（`:set paste`），然后这时用鼠标中键粘贴就不会有问题了，不过记得用完要关掉paste模式（`:set paste!`）。更简单的方法是：直接在命令行模式下使用`"+p`或者`"*p`，不用管什么模式了。
- Macros
    1. `q{register}` 开始录制，再次按q来停止录制，可以通过`:reg {register}`来查看宏的内容
    2. 通过`@{register}`来执行宏，`@@`来执行最近调用过的宏
    3. Series or Parallel 顺序还是并行。以并行方式执行宏，遇到错误时不会停止。
    4. 录制宏的原则：确保每条命令都可以被**重复**执行
    5. 如何以并行方式执行宏？先在visual模式下选中需要执行宏的所有行，然后在命令行`:normal @a`
    6. 在宏的末尾追加命令。假设之前录制的宏为a，则使用`qA`可以在原来的宏末尾继续添加命令而不会覆盖之前的宏。
- Matching Patterns and Literals
    1. 按正则表达式查找时，使用`\v`模式开关，这会开启very magic搜索模式 eg. `/\v#([0-9a-fA-F]{6}|[0-9a-fA-F]{3})`
    2. 按原义查找文本时，使用`\V`原义开关 eg. `/\Va.k.a`
    3. 界定单词的边界。用`< >`来标识单词的边界  eg. `/\v<the>`
    4. 界定匹配的边界。用`\zs`标识匹配的开始，`\ze`标识匹配的结束，比较有趣，查找还按原来的规则查找，只是显示匹配的时候只把感兴趣的部分显示出来。 eg. `/\v"\zs[^"]+\ze"` 只显示引号内的内容
    5. 统计当前模式的匹配个数。 `:%s///gn`
