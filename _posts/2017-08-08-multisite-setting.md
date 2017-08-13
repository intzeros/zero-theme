---
layout: post
title:  "利用Apache进行多站点配置"
date:   2017-08-08 08:30:00
categories: coding4fun
tags: [jekyll, blog, multisites]
---

用了静态网页之后，主机就空闲了，当初辛辛苦苦跟工信部折腾了好几个月连案都备好了。。总感觉特别亏（其实就是舍不得wordpress的搜索功能。。）转念一想，拿wp当图片上传服务也是不错啊，这样就省去找第三方图床了。

我的目标：

* www.lszero.com 显示主页面。
* blog.lszero.com 对应静态site。
* wp.lszero.com 对应wordpress。

我这里用的是apache来实现的，也可以用nginx。

因为我的测试平台是Ubuntu，故跟网上的配置文件有些不一样。如网上说的apache配置文件`http.conf`，在我的平台下是`/etc/apache2/apache2.conf`。<!-- more -->

##  Apache下多站点配置

首先修改默认的VirtualHost，然后再添加其他的VirtualHost。

1.sudo vi /etc/apache2/apache2.conf，设置：

```
ServerName 'www.lszero.com'
<Directory "your_home_page_path">
...
</Directory>
```

注意该文件的最后两行为：

```
IncludeOptional conf-enabled/*.conf
IncludeOptional sites-enabled/*.conf
```

故把新加的站点配置文件放在`sites-enabled`目录下。

继续，sudo vi /etc/apache2/sites-available/000-default.conf，设置：

```
DocumentRoot your_home_page_path
```

2.添加新站点配置文件，并放在`sites-available`中。

```shell
$ cd /etc/apache2/sites-available
$ sudo touch wp-lszero-com.conf
$ sudo touch blog-lszero-com.conf
```

sudo vi blog-lszero-com.conf，添加：

```
<VirtualHost *:80>
    ServerAdmin intzero@outlook.com
    ServerName blog.lszero.com
    DocumentRoot your_jekyll_site_path/_site

    <Directory "your_jekyll_site_path/_site">
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

sudo vi wp-lszero-com.conf，添加：

```
<VirtualHost *:80>
    ServerAdmin intzero@outlook.com
    ServerName wp.lszero.com
    DocumentRoot your_wordpress_site_path

    <Directory "your_wordpress_site_path">
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

然后建立软链接到`sites-enabled`目录：

```shell
$ cd /etc/apache2/sites-enabled
$ sudo ln -s /etc/apache2/sites-available/www-lszero-com.conf /etc/apache2/sites-enabled/www-lszero-com.conf
$ sudo ln -s /etc/apache2/sites-available/blog-lszero-com.conf /etc/apache2/sites-enabled/blog-lszero-com.conf
```

重启：

```shell
$ sudo service apache2 restart
```

参考：

- [Ubuntu/CentOS下Apache多站点配置](http://www.linuxidc.com/Linux/2017-05/143590.htm)



## 修改数据库

最后还要修改wordpress数据库里的链接：

```shell
$ mysql -u[user_name] -p[your_passwd]
(1) UPDATE wp_options SET option_value = replace( option_value, 'your old web_site','your new web_site') WHERE option_name = 'home' OR option_name ='siteurl';
(2) UPDATE wp_posts SET post_content = replace( post_content, 'old.com','new.com') ;
(3) UPDATE wp_comments SET comment_content = replace(comment_content, 'old.com','new.com') ;
(4) UPDATE wp_comments SET comment_author_url = replace(comment_author_url, 'old.com','new.com') ;
```



---

额外说一点，

Wordpress自带有多站点功能，前提是你的站点都是wordpress。当然我就不属于这种情况了。

可参考：[WordPress多站点的创建](http://www.smyx.net/create-wpmu.html)

首先开启多站点功能（默认是关闭的），修改wordpress配置文件：vi wp-config.php。找到

```
/* That's all, stop editing! Happy blogging. */
```

在上面这行代码的前面加上：

```
define('WP_ALLOW_MULTISITE',true);
```

回到wordpress后台界面，点击`工具 -> 配置网络`。

因为不是新创建的wordpress，故只能用`子域名`模式，而不是`子目录`模式。点击安装，然后根据提示，修改`wp-config.php`和`.htaccess`。

然后在Dashboard的左上角就可以设置不同的站点，如新建一个站点。

说明：`主站点`还是原来的lszero.com，`子站点`为你后来新加的。

对于删除多站点：

除了要恢复上述修改的文件之外，还有删除数据库多出来的表。

