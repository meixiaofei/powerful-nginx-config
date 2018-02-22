# powerful-nginx-config
Self use powerful nginx configuration

nginx try_files的一个坑  
原来的配置是酱紫的
```ini
location / {
        try_files $uri $uri/ /index.php$is_args$args /app_local.php$is_args$args /app_dev.php$is_args$args;
    }
```
本以为它会首先找`$root/app_local.php$is_args$args`这个地方的东东，可没想到的是：它直接当文件读取然后返回了。。

解释：
---
这要从 nginx try_files 的工作原理说起。以 try_files $uri $uri/ /app_local.php 为例，当用户请求 http://domain/example 时，这里的 $uri 就是 /example。try_files 会到硬盘里尝试找这个文件。 
如果存在名为 /$root/example（其中 $root 是 项目目录）的文件，就直接把这个文件的内容发送给用户。显然，目录中没有叫 example 的文件。 
然后就看 $uri/，增加了一个 /，也就是看有没有名为 /$root/example/ 的目录。又找不到，就会 fall back 到 try_files 的最后一个选项 /app_local.php， 发起一个内部 “子请求”，也就是相当于 nginx 发起一个 HTTP 请求到 http://domain/app_local.php。  
这个请求会被 location ~ \.php$ { ... } catch 住，也就是进入 FastCGI 的处理程序。而具体的 URI 及参数是在 REQUEST_URI 中传递给 FastCGI 和 PHP应用 程序的，  
因此不受 URI 变化的影响。

配置修改之后，/app_local.php 就不在 fall back 的位置了，也就是说 try_files 在文件系统里找到 app_local.php 之后，就会直接把文件内容（也就是 PHP 源码）返回给用户。这里的坑就是 try_files 的最后一个位置（fall back）是特殊的，它会发出一个内部 “子请求” 而非直接在文件系统里查找这个文件。
