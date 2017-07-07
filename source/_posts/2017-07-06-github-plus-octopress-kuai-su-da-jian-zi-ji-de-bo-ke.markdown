---
layout: post
title: "Github+Octopress 快速搭建自己的博客"
date: 2017-07-06 18:44:14 +0800
comments: true
categories: 

---

今天完成了在Mac上使用Github+Octopress搭建自己专属的博客，在网上搜了一些相关的资料，趟了一些坑，于是决定把自己搭建博客的过程记录一下。

### Octopress 与 Jekyll & Github Pages 的关系 

1. Octopress 是基于Jekyll的静态博客框架
2. Github Pages 是用于显示托管在Github的静态网页，是Github提供的服务
3. 也就是说我们是使用基于Jekyll的Octopress生成本地的静态网站，然后将静态网站托管到Github为我们提供的Github Pages 服务上，最后访问博客地址就可以显示我们的个人博客网站了

### 安装环境

1. Mac OS X系统（根据个人情况而定）
2. 安装ruby，由于之前安装了cocoaPods，所以已经有ruby，一般来说，Mac OS X都是自带ruby的，这里对于安装ruby的过程不再多做介绍

### 本地安装Octopress

1. #### 将Octopress的项目clone到本地，终端执行命令行

   ```
   $ git clone git://github.com/imathis/octopress.git octopress
   ```

   进入octopress目录

   ```
   $ cd octopress
   ```

2. #### 安装Octopress所需要的依赖库（dependencies）

   安装过程中可能遇到权限问题，所以需要在命令行前面加sudo再执行，并输入登录密码。

   > 注：`sudo` 全称是：`super user do`，也就是以 root 用户身份来执行

   ```
   $ sudo gem install bundler
   $ bundle install
   ```

   这里在不翻墙的情况下，可能会遇到一个问题：`sudo gem install bundler` 执行后，一直没有响应。这是由于国内网络原因（你懂的），导致rubygems.org存放在 Amazon S3 上面的资源文件间歇性连接失败。所以你会遇到`gem install rack`或`bundle install` 的时候半天没有响应的情况。

   但好消息是国内某大神帮我们解决了这一心头大患，我们可以用淘宝的Ruby镜像来替换原来的镜像。只需终端执行下面的命令即可：

   ```
   $ gem sources -a https://ruby.taobao.org/ -r https://rubygems.org/
   ```

   然后执行如下命令查看切换后结果

   ```
   $ gem sources -l
   ```

   如果看到如下输出：

   ```
   *** CURRENT SOURCES ***
   https://ruby.taobao.org
   ```

   这就说明我们切换到淘宝的 Ruby 镜像了，再次安装 Octopress 所需要的依赖库就会发现成功啦。

   当然还有另外两种方法:

   1. 比较原始的方法 -> 动更改：打开 `octopress/Gemfile`文件 -> 将 `source "https://rubygems.org"` 改为 `source "https://ruby.taobao.org"`就可以了

   2. 相对方便点，因为我们使用的是 Gemfile，所以我们可以用 Bundler 的 Gem 源代码镜像命令，这样我们就不用改 Gemfile 的 source 了。命令如下：

      ```
      $ cd octopress
      $ bundle config mirror.https://rubygems.org https://ruby.taobao.org
      ```

3. #### 安装默认主题

   终端执行下面命令，可以安装默认主题，当然也可以安装第三方的主题，这个留到下篇再详解。

   ```
    $ rake install
   ```

4. #### 内容预览

   到这里，我们已经在本地搭建好了一个简版的Octopress博客，是不是迫不及待的想看看效果，那就运行命令吧！

   ```
    $ rake preview
   ```

   此时打开浏览器，输入localhost:4000，你就可以看到效果了，至此，本地Octopress安装过程就结束了。

   ![](../images/2017/07/635689-9a554909effc43fe.jpg)


### 本地的Octopress博客部署到Github Pages

接下来，我们就需要将本地Octopress部署到Github Pages上，一起来看看具体步骤吧

1. 新建Github repository

   登录个人的Github账号，新建Github repository，项目名称命名格式为username.github.io，username是你的Github用户名，例如你的Github用户名为XXXgithub，那么项目名称应该为XXXgithub.github.io，然后点击创建即可。

2. 配置 Github Pages

   在终端运行下面命令

   ```
   $ cd octopress
   $ rake setup_github_pages
   ```

   该命令会要求我们输入github仓库的url，复制粘贴一下新建仓库的SSH 或 HTTPS URL 即可，因为SSH需要注意配置SSH Key的问题，所以在此处我选择了HTTPS URL。

   很多人也许会疑问，Octopress生成的静态页面是怎么与GitHub绑定的呢？

   对的，就是rake setup_github_pages这行代码，那么它是怎么实现的呢？

   用户的 Github Pages使用master分支作为Web服务的公开目录，为我们的博客主页（username.github.io）提供内容显示，因此，我们需要将与博客相关的源码存到某个分支上（这里是source分支），而master分支commit生成的博客内容供Web访问，Octopress就是通过这行代码帮我们搞定这些工作的。

3. 创建一篇文章

   终端执行命令

   ```
   $ rake new_post["title"]    #title为你的文章名，可随意更改
   ```

   生成的新文章在source/_post目录下，我们可以用Markdown编辑器对文章进行编辑修改。

   打开目录下的markdown文件（我目前用的Typora），会发现头部有如下内容（千万不可删除！！）

   ```
   layout: post
   title: "Github+Octopress 快速搭建自己的博客"
   date: 2017-07-06 18:44:14 +0800
   comments: true
   categories: 
   ```

   下面我们就可以编辑自己的正文内容了。

   正文编辑完成以后，终端运行如下命令即可生成静态网页

   ```
   $ rake generate
   ```

   如果你想预览内容的话，则终端运行命令

   ```
   $ rake preview
   ```

   此时，你可以在浏览器输入localhost:4000查看实际效果，如果没有问题的话，就可以将静态网页同步到GitHub上，终端运行命令

   ```
   $ rake deploy
   ```

   > 注意了，在这里我遇到了一个坑，每次执行rake deploy的时候，就会提示
   >
   >  ! [rejected]        master -> master (non-fast-forward)
   >
   > error: failed to push some refs to 'https://github.com/stoneqi2007/stoneqi2007.github.io.git'
   >
   > hint: Updates were rejected because the tip of your current branch is behind
   >
   > hint: its remote counterpart. Integrate the remote changes (e.g.
   >
   > hint: 'git pull ...') before pushing again.
   >
   > hint: See the 'Note about fast-forwards' in 'git push --help' for details.
   >
   > 在网上找了好久，最后找到解决办法：
   >
   > $ cd octopress
   >
   > 打开Rakefile文件，找到system "git push origin #{deploy_branch}"，在#前面加+，然后保存文件
   >
   > 重新执行$ rake deploy即可解决。

   你会看到静态页面已经被push到GitHub的master分支，然后等待几秒钟，访问博客主页username.github.io，就可以看到你创建的博客了。

   如果你还想把自己的本地资源也同步到GitHub上，那么终端运行指令

   ```
   $ git add .
   $ git commit -m "comment"  #comment可随意更改
   $ git push origin source
   ```

   这样本地的资源文件就上传到GitHub的source分支了。

   > 注意：`rake preview` 会自动监视文件的变化，重新生成静态页面。因此修改markdown文件后，只需要在浏览器里刷新一下页面，就立刻可以看到效果。不过如果修改了_config.yml的话，则需要`Ctrl+C`终止，用 `rake generate` 重新生成，才能看到效果。

   至此，我们就完成了个人博客的搭建，当然此时的博客略显简陋，我们可以根据自己的喜好重新包装自己的博客，至于怎么个性化定制自己的博客，会在后续的文章中给出。


