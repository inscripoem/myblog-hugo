---
title: WordPress 的 Docker 迁移
description: 
slug: wordpress-to-docker
categories:
    - 开发/服务器踩坑记录
tags:
    - WordPress
    - Docker
    - 博客
date: "2024-06-15T00:00:10+08:00"
image: cover.jpeg
math: 
license: 
hidden: false
comments: true
draft: false
---

WordPress 的迁移问题对我来讲一直是个大难题。

其实也不是对我，这东西迁移虽然麻烦，但也只要复制个目录然后给数据库写个 SQL（不换域名甚至不用写）就搞定了，现在甚至有现成的插件（如 [Duplicator](https://wordpress.org/plugins/duplicator/)）可以一键搞定，我担心的是风蓝（前情提要见[当我们谈论风蓝官网时我们在谈论什么](https://diary.aoikaze.moe/article/001)）。

对于风蓝来讲其实需要一个更加傻瓜更加一键（虽然这不一定是好事）的解决方案，作为多年 Docker 用户自然想到了容器化。用 Docker Compose 可以把网站和数据库打包在一个目录里，移动到新机器不用修改，并且实现一行命令的启动或停止。

实际操作起来还遇到了蛮多坑（包括之前不使用 Docker 时来不及记录的），在这里一并写一下。

# 操作步骤

[Compose 文件](https://github.com/docker/awesome-compose/blob/master/official-documentation-samples/wordpress/README.md)来自 [docker/awsome-compose](https://github.com/docker/awesome-compose)：

```yaml
services:
  db:
    # We use a mariadb image which supports both amd64 & arm64 architecture
    image: mariadb:10.6.4-focal
    # If you really want to use MySQL, uncomment the following line
    #image: mysql:8.0.27
    command: '--default-authentication-plugin=mysql_native_password'
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=somewordpress
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
    expose:
      - 3306
      - 33060
  wordpress:
    image: wordpress:latest
    volumes:
      - wp_data:/var/www/html
    ports:
      - 80:80
    restart: always
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
      - WORDPRESS_DB_NAME=wordpress
volumes:
  db_data:
  wp_data:
```

实际操作的时候只是把挂载的两个卷 `db_data` 和 `wp_data` 提到本地目录中，其他基本不变。

这一步会创建两个容器，一个 mariadb 数据库和一个 nginx+wordpress 的 web 服务器。 `wordpress` 映射出来的端口如果宿主机上没装 web 服务器可以直接 `80` ，我是比较习惯映射到别的端口之后做反代的（宝塔新版本可以直接创建反代项目了，这点好评~~究竟是谁2024年了还在用宝塔~~），此处不再赘述。

`docker compose up -d` 之后访问端口或域名就是一个全新的wordpress，如果需要迁移的话可以用上文提到的 Duplicator 直接打包放到 `wp_data` 里安装。

# 踩坑

终于到了~~我最喜欢的环节~~最折磨的排坑环节，现在看来大多数问题都不是容器化而是 WordPress 本身的一些麻烦特性，更加坚定了我虽然 WordPress 很好但是还是要坚决舍弃的信念。

需要注意的是，下面的几乎所有问题**我都没有进行刨根问底了解原因**，只是简单推测+搜索解决方案罢了。归根结底是我认为如果需要用这种奇技淫巧才能保证在现代环境下的使用的话，也不需要浪费大量的时间在折腾这些东西上，能用就行了。

### **ERR_TOO_MANY_REDIRECTS**

这个问题在我将设置中的 WordPress 地址和站点地址均改为 `https://example.com` 后出现了，访问网站本身没有问题，但是在访问管理站点的时候却会报 `ERR_TOO_MANY_REDIRECTS` 。在多次重启以及苦等后都没有改善后，我找到了 StackExchange 里的[这篇帖子](https://wordpress.stackexchange.com/questions/302965/too-many-redirects-only-when-trying-to-access-wp-admin-page)，一下子就想明白了。

原生 WordPress 本身是不支持在站点内设置 HTTPS 的，因此我的 SSL 设置在反代位置的 Nginx。而WordPress 的地址设置又掌管着其重定向的 Base URL，所以在访问带 https 的站点后，外层 Nginx 将我带向了内层的 WordPress，而此时根据设置，WordPress又要将我重定向到 https 站点，这样一来二去重定向就进入了死循环。

我自己的解决方案有点类似[这楼](https://wordpress.stackexchange.com/a/318028)，只是我明确知道没有在 Docker 中的 Nginx 里设置 SSL，所以索性把地址改回 `http` 开头好了。

有很多种改法，如果可以访问到数据库的话，可以修改 `[sitename]_options` 表里的 `home` 和 `siteurl` 字段。也可以在文件根目录的`wp-config.php` 里加上：

```php
define('WP_HOME','http://example.com');
define('WP_SITEURL','http://example.com');
```

同时，在修改了 `wp-config.php` 之后，站点设置中被覆盖的项会变灰不能修改，也即数据库中的值无用，这点请务必注意~~我就疑惑了好久为什么改不了~~。

## 插件安装与目录权限

对于直接在宿主机安装的 WordPress，插件安装是很顺滑的，不需要特殊的操作就可以在插件市场一键安装。但是 Docker 镜像有一个问题，就是 WordPress 的官方镜像是以 `www-data` 用户运行的，而复制进去/自动创建的文件权限不对，这就导致在安装插件时需要填写 ftp 账号云云。

解决方法参考了[这个 issue](https://github.com/docker-library/wordpress/issues/298) 以及[回复中提到的 gist](https://gist.github.com/dianjuar/1b2c43d38d76e68cf21971fc86eaac8e) ，首先还是熟悉的在 `wp-config.php` 中添加：

```php
define('FS_METHOD', 'direct');
```

这会改变插件的安装模式。接着在终端中改变 WordPress 文件夹的所有者及权限：

```bash
docker exec -u root -it {CONTAINER_ID} /bin/bash
chown -R www-data wp-content
chmod -R 755 wp-content
```

当然我没有改所有者而是直接把文件夹权限改到 `777` 了，别学我，会被打的（）~~一说好像风蓝官网上次下线就是因为被黑客打掉来着~~

# 优缺点

私以为把 WordPress 这种古早框架迁移到 Docker 里还是有一些优点的：

1. **易于迁移**
    
    我最喜欢的优点，整个网站都在一个文件夹里，真要搬家不需要备份恢复数据库，不需要改设置，直接一整个目录打个压缩包就带走了。
    
2. **易于部署**
    
    整个系统的启动只需要一句 `docker compose up -d` ，关闭和重启也都是一句解决，也可以方便地用 `docker compose logs` 同时看两个容器的日志。虽然原则上并不推荐数据库进容器，但是容器！爽！
    
3. **更加安全（？）**
    
    在 Docker Compose 中数据库和 WordPress 本体全都在虚拟网络中，平常数据库对外不会暴露端口，降低了被打的风险。不过这点比较存疑，因为只要 WordPress 能访问数据库，直接攻破WordPress 现有漏洞的成本也很低。
    

当然，缺点也很明显：

1. **数据库以及文件访问更加繁琐**
    
    放进 Docker 之后文件和端口都需要映射才能访问，有些操作更是需要进入容器内进行，就数据库和文件管理上确实比之前繁琐了。当然也可以在 Compose 的配置中加入 PHPMyAdmin 这种数据库管理程序（[成品](https://github.com/nezhar/wordpress-docker-compose)），但是端口管理有点麻烦而且我没有成功部署，可以自行尝试。
    
2. **需要处理权限问题**
    
    正如前文所言，Docker 也同样意味着容器内外更容易出现所有者和权限不一致的问题，而且通常解决也要耗费一定的时间（可能要回去看 Dockerfile 或者进入容器）
    
3. **访问速度变慢**
    
    这是我最不能接受的缺点，本来 WordPress 就以其孱弱的性能臭名昭著，进 Docker 之后每次文件访问要经过目录挂载，每次数据库请求也要经过目录挂载，体感上网站的加载时间甚至增加了100%以上。
    

# 总结与展望

总体而言算是还比较愉悦的体验，但是在折腾这个的时候我发现自己服务器上的面板突然无法读出 Nginx 配置，在极慢的访问速度和时不时挂掉的宿主机 Nginx 双重压力下，我觉得我还是会转向更加新的技术栈。都2024年了，为什么要花这么长时间来折腾一个甚至不原生支持 HTTPS 的博客框架呢（但事实证明确实 WordPress 有其存在的必要，它还是目前社区最大可定制程度最高且支持多作者写作的全功能CMS），该进行新的选型了！

今天看到了一些新的优化方法（[Redis](https://sleele.com/2020/03/29/wordpress%E6%90%AD%E9%85%8Dredis%E5%8A%A0%E9%80%9F%E7%BD%91%E7%AB%99%E8%AE%BF%E9%97%AE%E9%80%9F%E5%BA%A6/), [jsDelivr](https://sleele.com/2020/05/09/wordpressjsdelivr-%E4%BC%AA%E5%85%A8%E7%AB%99cdn/)）， 如果之后还有精力的话可能会尝试折腾一下，毕竟现在风蓝几乎全部的 post 都在 WordPress上，丢了属实是有一些可惜。