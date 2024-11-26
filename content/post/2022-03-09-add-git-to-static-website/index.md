---
date: "2022-03-09"
categories:
  - 折腾记录
title: 给你最爱的静态网站加上方便上传管理的git属性吧！
slug: add-git-to-static-website
image: https://www.loliapi.com/acg/?id=1
---


搞这个的前提是有一天我在编辑[主页](https://www.inscripoem.com)的时候忘记事先备份，然后之前调完的渐变就全丢了orz。这里要cue一下罪魁祸首[HoYoverse的关于我们页面](https://www.hoyoverse.com/zh-cn/about-us)，这个文字渐变实在太让人想抄了！（当然最后以没抄到动画代码还不得不猜一个之前用的渐变颜色的结果收场）

<!--read more-->

正好之前看hexo本地部署的方法就是利用一个git用户创建仓库的形式来管理代码，把云端当成一个repo来部署，就想是不是可以把中间的一部分借鉴来完成静态网站的管理，这样我既有一个本地上传的渠道不用天天ssh，也可以方便退回到从前的版本。那么事不宜迟，我们马上开始。

## 1.添加git用户

安装git与nginx这一步我们就此跳过，从添加git用户开始

```Bash
useradd git
passwd git

# 给git用户配置sudo权限
chmod 740 /etc/sudoers
vim /etc/sudoers
# 找到root ALL=(ALL) ALL，在它下方加入一行
git ALL=(ALL) ALL

chmod 400 /etc/sudoers
```

## 2.为git用户添加git密钥

```Bash
su - git
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorzied_keys
chmod 700 ~/.ssh
vim ~/.ssh/authorized_keys    #将ssh密钥粘贴进去
```

这一步的ssh密钥可以参考[《生成SSH密钥》](https://www.jianshu.com/p/7aba9b127cb8)来进行生成，总体上是运行`ssh-keygen -t rsa -C "your_email@example.com"`后，在对应用户的`.ssh`目录下生成的`id_rsa.pub`文件就是公钥，进行粘贴即可

## 3.创建git仓库并使用git-hooks进行自动部署

```Bash
sudo mkdir -p /home/git/repos    #新建目录，这是git仓库的位置
sudo mkdir pp /www/wwwroot/homepage #homepage为你创建网站的位置
cd /home/git/repos  #转到git仓库的文件夹
sudo git init --bare homepage.git #创建一个名叫homepage的仓库
sudo vim /home/git/repos/homepage.git/hooks/post-update
```

将`post-update`的内容改为如下：

```Bash
#!/bin/bash
git --work-tree=/www/wwwroot/homepage --git-dir=/home/git/repos/homepage.git checkout -f
```

**给post-update授权**：

```Bash
cd /home/git/repos/homepage.git/hooks/
sudo chown -R git:git /home/git/repos/
sudo chown -R git:git /www/wwwroot/homepage
sudo chmod +x post-update  #赋予其可执行权限
```

下面一步是**修改git用户的默认shell环境**，但我本人没有进行这一步就完成了，所以暂且贴在这里，可以自行展开

{% folding 修改git用户的默认shell环境 %}

```Bash
vim /etc/passwd 
#修改最后一行 
#将/bin/bash修改为/usr/bin/git-shell 
```
{% endfolding %}

## 4.进行git仓库的拉取测试及初始化

接下来可以试着在本地电脑上（当然需要安装git）运行

```Bash
git clone git@xxx.xx.xxx.xxx:/home/git/repos/homepage.git
```

如果成功，恭喜！这表明你前面的步骤没有出现错误，下面如果你试图开始往里面添加内容并推送，就会发现形如

```plaintext
error: src refspec master does not match any.
error: failed to push some refs to 'git@xxx.xx.xxx.xxx:/home/git/repos/homepage.git'
```

的错误，这是由于没有初始化git仓库造成的，因此需要在本地端运行

```Bash
git add . #将文件添加到仓库
git commit -m "注释内容" #进行提交
git push -u origin master #将更改推送至远程仓库
```

后续即可正常提交

锵锵！以后在本地端就可以直接把网站当成git仓库来改动及进行版本管理了

注：由于本文章在撰写时笔者对于Git的理解并不深刻，所以在以上步骤运行后依然无法正常使用，或是关于代码之外的文件的问题，可以参阅文末的参考资料及网络等进行进一步探寻

**参考文章：**

[将Hexo部署到自己的服务器上 StaryJie](https://www.cnblogs.com/jie-fang/p/13445939.html)

[error: src refspec master does not match any. 错误的解决办法 oliverhoo](https://blog.csdn.net/qq_38198952/article/details/82792279)