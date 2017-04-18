---
layout: post
title: 如何使用jekyll在github上构建自己的博客来分(zhuang)享(B)
date: 2017-04-14 14:30
comments: true
external-url:
categories: Tech-Frontend
---

> 说实话，我考虑了很久写博客究竟用中文还是用英文，考虑再三，鉴于母语毕竟不是英语，做技术分享的话，很有可能会引起歧义，造成不严谨，所以对于技术分享类的文章，我还是先用中文完成，之后会尽可能的自己去翻译，然后想办法做个双语的版本，一方面可以练习英语，一方面，只是我自己的痴心妄想，也许哪天会有国际友人跟我有更深♂入♂的♂交♂流~

好了，那么该开始我的第一篇文章了~ 怎样用jekyll构建我的blog！

其实当你搜索互联网，你可以发现很多文章都在介绍你怎么用jekyll来搭建博客，但是实际上，我周围很多朋友在构建博客的过程中，都遇上了各种各样奇怪的问题，大家会因为操作系统不同，工具不同，甚至平时的习惯不同，碰见各种各样的问题，虽然无法保证我能把过程写的尽善尽美，但是至少，如果你是一个mac用户，我的文章应该会对你有一定的帮助

## 环境预备
由于macOS本身集成了很多开发小伙伴们平时使用的工具，所以安装过程可能会比使用windows的少年们轻松很多（像我这样的懒人一直没在我的windows电脑上装……），但是实际上，当你真正安装的时候会发现其实会遇上各种版本问题和依赖问题，所以不妨参考下我的准备列表
- ruby 2.3.0+ 由于一般mac系统都是集成ruby的，只是版本相对较低，所以还需要安装下面的东西
- rvm 由于在我构建博客的时候，发现有些安装会需要以依赖ruby更高的版本，所以非常建议安装rvm，因为这个工具可以作为本机ruby管理的工具，尤其对于一些全栈开发的小伙伴们，日常工作恐怕也会用上，分享我学习安装的教程：[rvm安装教程](http://www.cnblogs.com/bigshow1949/p/5642775.html)
- Homebrew 这个我可能说的都有点多余，我相信很多mac的开发人员都会使用这个在我看来神一样的工具(虽然原作者因为转二叉树悲剧过，然而我自己有时候手写代码也很垃圾，看上去没什么联系……)，直接上[官网地址](https://brew.sh)
- git 由于时间确实有点长了，我是在不记得我是否对mac的git做过什么，不过向我这种喜欢装x的人一项都喜欢最新的版本～直接使用homebrew来装一个还是挺好用的：  
```bash
# 查看当前版本
$ git version
# 旧版本Git的地址
$ which git
# 利用brew安装新版本的Git
$ brew install git
# 移除旧的版本
sudo mv /usr/bin/git /usr/bin/git-x.x.x
```
- RubyGem   
  > 这个我需要特别说明一下RubyGem这个东西在国内访问非常的困难，我曾在update的时候脑残的等了15分钟，当然我不会告诉你期间我在跟人家聊天……，后来才知道需要使用国内的镜像

  · 首先从[官网](https://rubygems.org/pages/download)下载相应的压缩版到你的指定路径  
  · 进入解压后的目录运行 ```$ sudo ruby setup.rb```  
  · 检查是否安装成功```$ gem -v ```
  · 若需要更新版本，请见如下方法：
  ```shell
  # 在我安装的时间，淘宝的镜像已不再支持，使用ruby-china
  $ gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
  $ gem sources -l
  https://gems.ruby-china.org # 确保只有 gems.ruby-china.org
  # 正式update，时间不会太长
  $ gem update --system
  # 检查当前版本
  $ gem -v
  ```

## 安装jekyll
> 虽然说网上管用github构建博客的框架很多，但是对于楼主这样的懒人，最喜欢的就是这种简单明快粗暴直接的东西，如果有哪位仁兄有强烈的理由特别的想安利我一个其他的框架，非常欢迎给我写邮件一起交流！

好了现在可以正式开始我们安装jekyll的工作了
- 首先你需要在你的github上建立一个新的Repository，由于github本身对展示页面的支持，这里你只要把repo的名称改成username.github.io即可，另外在项目的setting中将Features下面的☑️Wikis选项勾选。
- 之后使用git clone将仓库拉至本地
- 敲开你的终端输入如下命令安装jekyll ```$ gem install jekyll ``` ⚠️注意这个时候可能需要root权限
- 在安装的时候可能会提示如下错误
```
ERROR:  While executing gem ... (Errno::EPERM)
    Operation not permitted - /usr/bin/sass
```

> 非常不幸的是，这其实是一类错误，不瞒你说，在[stackoverflow](http://stackoverflow.com/questions/31443530/sass-error-installing)早就有老司机们发现了  
于是乎这个这个错误直接使用如下命令即可```sudo gem install -n /usr/local/bin sass```  
其实此类错误都可用该命令，其实是安装来解决```$ sudo gem install -n /usr/local/bin missed_gem_package_name```

- 相信至此，你的jekyll也已经完成安装，但是别着急，离运行还差最重要的一步，那就是拷贝一份jekyll的模板至之前你clone到本地的那个仓库，当然你也可以自己去搞，你需要的是按照[官方给出规则](http://jekyll.com.cn/docs/structure/)来建立目录
- 当然如果你跟我一样也是个懒人或者只是想先熟悉一下，你大可直接下载一个模板到本地，想我就是用了这个[极简的风格](https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll)，当然我也可能会换啦～
- 完成内容之后，cd至你的项目目录执行如下命令```$ jekyll serve``` 执行完命令，在输出中你就可以看到本地模拟路径了，类似我是127.0.0.1:4000，ok页面出来了！
- 确保这个就是你满意的结果之后，只需要继续git命令，将这个Repository进行提交并push到你的github远端，这时候，在浏览器访问你这个项目的名称 yourname.github.io 你的博客从今天开始上线了～
- 稍作说明的是，虽然模板千变万化，但是jekyll的目录结构其实都类似，在_posts文件夹里面写你的具体内容，_config.yml中是你网站目录，展示等基本信息

#### 好了到此，你肯定也已经上道儿了～ 如果你觉得有任何问题或者觉得在下哪里写的不好，欢迎发邮件给我，我会即使改进，我也会在之后加入评论功能。写的不好多有包含
