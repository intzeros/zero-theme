---
layout: post
title:  "搭建Isso评论服务"
date:   2017-08-09 10:30:00
categories: coding4fun
tags: [jekyll, blog]
---

与其借助第三方服务天天担心挂，还不如将自己的服务器资源利用起来。[Isso](https://posativ.org/isso/)就是自建评论系统的一种方式。（官网需要fq.）

## Install Isso

测试自Ubuntu服务器。本来官网文档里没那么多戏，但是我一直报错。。又安了很多其它包。

```
$ sudo apt-get install python3-dev
$ sudo pip install isso
$ sudo apt-get install sqlite3

# 可能需要：
$ sudo pip install werkzeug
$ sudo apt-get install libffi-dev
$ sudo pip install misaka
```

<!-- more -->

## Configuration

首先新建一个isso配置文件，`vi isso.conf`，放哪儿都行。

```
[general]
; database location, check permissions, automatically created if not exists
; your website or blog (not the location of Isso!)
dbpath = xxx/comments.db
name = example
; you can add multiple hosts for local development
; or SSL connections. There is no wildcard to allow
; any domain.
host = http://lszero.com/
notify =
log-file = xxx/isso.log

[server]
listen = http://localhost:8090/

[moderation]
enabled = false
purge-after = 30d

[smtp]
username = username@example.com
password = password
host = smtp.example.com
port = 587
security = starttls
to = your email address
from = "Isso Comment"<username@example.com>
timeout = 10

[markup]
options = strikethrough, superscript, autolink
allowed-elements =
allowed-attributes =

[guard]
enabled = true
ratelimit = 3
direct-reply = 3
reply-to-self = true
require-author = true
require-email = true

...
```

其中，

* `dbpath` 是数据库文件，会自动生成。
* `host` 可以添加多个。
* `notify` 选择新评论的通知方式。`stdout`或者`smtp`。
* `listen` 设置端口。
* `moderation` 是否开启评论审核。若开启，新的评论不会显示，除非你activate。
* `smtp` 即设置邮件服务的配置信息。
* `guard` 评论防火墙。
* `ratelimit` 每个访客一分钟最多可以评论的次数。
* `direct-reply` 评论回复次数。
* `reply-to-self` 是否可以回复自己的评论，需要配合 JS 引用，下面会说

更多设置选项参见[官网文档](https://posativ.org/isso/docs/configuration/server/?utm_source=sb.sb)。

运行isso：

```
$ isso -c /path/to/isso.conf run
```

## 反向代理

根据上述配置，isso运行在http://localhost:8090下。我们现在需要通过反向代理功能，目的是让发送到comments.lszero.com的请求被转发到本地isso的端口。

官网给了nginx的配置，我这里也用apache测试了。

### with nginx

修改nginx配置：`vi /etc/nginx/sites-enabled/default`，在开头添加：

```
server {
    listen 80;
    server_name comments.lszero.com;

    location / {
        proxy_pass http://localhost:8090;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

启动nginx：

```
sudo service nginx start
```

有时候nginx配置文件错误，上述命令并没有任何错误提示。

可以通过`sudo service nginx status`查看是否真的启动了。或者通过`sudo nginx -t`来测试配置是否正确。

### with apache2

配置环境是Ubuntu apache2.

首先要通过`a2enmod`加载命令加载代理模块：

```
a2enmod proxy proxy_http
```

然后添加一个VirtualHost，为了方便，我这里就直接在 /etc/apache2/sites-available/000-default.conf 上改了。

```
<VirtualHost *:80>
        ServerAdmin intzero@outlook.com
        ServerName comments.lszero.com
        # off表示开启反向代理，on表示开启正向代理
        ProxyRequests Off
        # 将这个虚拟主机跳转到本机的8090端口
        ProxyPass / http://localhost:8090/
        ProxyPassReverse / http://localhost:8090/
        <Proxy *>
                Order Deny,Allow
                Allow from all
        </Proxy>
</VirtualHost>
```

## Test

将下列代码插入你想显示评论功能的位置：

```
<script data-isso="//comments.lszero.com/"
        src="//comments.lszero.com/js/embed.min.js"></script>

<section id="isso-thread"></section>
```

这里的更多设置选项参见[官网文档](https://posativ.org/isso/docs/configuration/client/)。比如开启支持/反对功能，一页显示评论数等。

然后启动web服务和isso：

```
$ sudo service nginx start   # or 'sudo service apache2 start'
$ isso -c /path/to/isso.conf run
```

这样就可以了。

小提示：

如果你在发评论自己测试的时候遇到403 Forbidden的错误，你可能是需要修改isso的配置文件，设置`reply-to-self = true`，让自己可以回复自己。