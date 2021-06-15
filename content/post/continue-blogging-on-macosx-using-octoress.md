+++
title = "continue blogging on Mac OSX using octoress"
date = "2015-07-12"
slug = "2015/07/12/continue-blogging-on-macosx-using-octoress"
Categories = ["mac", "octopress"]
+++

换了Mac，终于可以继续用Octopress写blog了，但首先得把原来的blog从git上同步过来，纪录下在此过程中遇到的坑。

1. clone source分支。     

        git clone -b source git@github.com:cs50Mu/cs50mu.github.com.git octopress
2. clone master分支到`octopress/_deploy`文件夹中

        git clone git@github.com:cs50Mu/cs50mu.github.com.git _deploy 
3. 重新安装Octopress环境，这里出现了一堆坑。。。
    - ruby gems官方源被墙，这个当时在Archlinux上安装时已经遇到，解决办法不再重复
    - 当执行`brew install ruby-build`提示找不到GCC。这个是因为mac上的gcc用的是苹果自己的编译器llvm，不是GNU版本的gcc，而octopress使用的特定版本的ruby（1.9.3）需要用GNU版的编译器来编译，按照homebrew的提示安装GCC即可

            $ brew update
            $ brew tap homebrew/dupes
            $ brew install autoconf automake apple-gcc42
    - 使用rbenv安装`Ruby 1.9.3`时，安装的ruby版本不能生效。表现的现象是：当执行`ruby --version`时仍然给出的时mac系统自带的ruby版本。解决办法就是在`.bash_profile`中添加

            # Initialize rbenv
            if which rbenv > /dev/null; then eval "$(rbenv init -)"; fi
      然后**注意**在zsh shell中需要在`.zshrc`中添加`source ~/.bash_profile`，因为zsh shell是不读`.bash_profile`文件的（或者是因为我安装了oh-my-zsh），还有要记住添加的这一行一定要在`.zshrc`文件中路径声明的后边！！否则，加这一句也不会起作用。切记，一开始在这个上面被坑了很久。
    - 执行`$ rake new_post["something to say"]`，zsh会报错：`zsh: no matches found: new_post[something to say]`，原因是`[ ]`在zsh中是文件名通配符，解决办法：
      在`.zshrc`中加入`alias rake="noglob rake"`

### 参考
- [Installing Ruby With Homebrew and Rbenv on Mac OS X Mountain Lion](http://blog.zerosharp.com/installing-ruby-with-homebrew-and-rbenv-on-mac-os-x-mountain-lion/)
- [rbenv-guide](https://ruby-china.org/wiki/rbenv-guide)
- [Octopress Setup](http://octopress.org/docs/setup/)
- [not compatible with zsh](https://github.com/imathis/octopress/issues/117)
- [Mac上安装octopress](http://liuyix.org/blog/2013/mac-install-octopress/)
- [在多台电脑上写Octopress博客](http://boboshone.com/blog/2013/06/05/write-octopress-blog-on-multiple-machines/)
