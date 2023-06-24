---
layout: post
title: "Nginx - alias 和 root 的配置差异"
subtitle: "介绍alias 和 root 的配置差异"
author: "Vascent"
header-img: "//w.wallhaven.cc/full/5g/wallhaven-5gk859.jpg"
tags:
  - Nginx
---

# Nginx

[Nginx Built-in Variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)

### `alias` 和 `root` 配置的差异

```json
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
    
    location /static/ {
        alias /var/www/static/;
    }
}
```

在这个例子中，**`root`** 指令将 **`/var/www/html`** 目录设置为服务器上静态文件的根目录。而 **`alias`** 指令将 **`/static/`** 替换为 **`/var/www/static/`**，并将请求映射到 **`/var/www/static/`** 目录下的文件或目录。

例如，如果用户请求 **`http://example.com/index.html`**，Nginx 将会直接映射到 **`/var/www/html/index.html`**。如果用户请求 **`http://example.com/static/images/logo.png`**，Nginx 将会将 **`/static/`** 替换为 **`/var/www/static/`**，并映射到 **`/var/www/static/images/logo.png`**。

<aside>
💡 需要注意的是，在使用 **`alias`** 指令时，路径后面必须添加 **`/`** ，否则可能会出现匹配错误。例如，如果将 **`alias`** 指令修改为如下形式：

</aside>

```json
location /static/ {
    alias /var/www/static;
}
```

当用户请求 **`http://example.com/static/images/logo.png`** 时，Nginx 将会将 **`/static/`** 替换为 **`/var/www/static`**，并映射到 **`/var/www/staticimages/logo.png`**，而不是 **`/var/www/static/images/logo.png`**。

***差异示例：***

可以在 Nginx 的配置文件中使用 **`alias`** 指令将 URI 中的 **`/conf`** 路径部分映射到实际的文件路径 **`/usr/local/nginx/conf/`** 上。具体配置如下：

```
location /conf/ {
    alias /usr/local/nginx/conf/;
}
```

这样，当用户访问 **`http://example.com/conf/nginx.conf`** 时，Nginx 将会返回实际的文件 **`/usr/local/nginx/conf/nginx.conf`**。注意，**`alias`** 指令需要以斜杠（**`/`**）结尾，以确保 Nginx 能够正确地解析 URI 路径。

如果要使用 **`root`** 指令来实现同样的功能，可以这样配置：

```
location /conf/ {
    root /usr/local/nginx/;
}
```

这样，当用户访问 **`http://example.com/conf/nginx.conf`** 时，Nginx 将会在 **`/usr/local/nginx/`** 目录下寻找 **`conf/nginx.conf`** 文件并返回。与 **`alias`** 不同的是，**`root`** 指令需要在 URI 路径后面添加要访问的文件路径，因此需要在 **`root`** 指令后面添加 **`/conf/`** 路径。
