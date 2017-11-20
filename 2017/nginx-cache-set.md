# Nginx 常用配置

Nginx安装好之后会有一个默认的nginx.conf配置，我们保持Nginx的默认配置，然后在底部用include方法把我们的设置添加进去。

## 缓存设置

关于缓存的讲解上一篇文章中有讲解，这里只给出具体的配置。

设置缓存可以通过expires字段和cache-control字段设置，但是nginx里面的expires字段只能设置时间，
不能设置cache-control中的其他字段。因此我们设置通过```add_header Cache-Control "max-age=0"```的方式设置。

下面是Nginx设置静态资源缓存的设置项。

```
server {
   listen       8080;
   server_name  name;
   root   /path;
   location / {
       try_files $uri /index.html;
   }
    error_page   500 502 503 504  /50x.html;

    location = /50x.html {
       root   /etc/nginx/html;
    }

    location ~* \.(?:manifest|appcache|html?|xml|json)$ {
      add_header Cache-Control "private,max-age=0";
    }

    location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|mp4|ogg|ogv|webm|htc)$ {
      access_log off;
      add_header Cache-Control "public,max-age=31536000";
    }

    location ~* \.svgz$ {
      access_log off;
      add_header Cache-Control "public,max-age=31536000";
    }

    location ~* \.(?:css|js)$ {
      add_header Cache-Control "public,max-age=31536000";
      access_log off;
    }

    location ~* \.(?:ttf|ttc|otf|eot|woff|woff2)$ {
     add_header Cache-Control "public,max-age=31536000";
     access_log off;
    }
    
}
```

listen是端口号，server_name是服务名，可以自己设置。

我们要注意```root /path```这一行。

root设置静态资源的根目录。

root可以放在location下面和server下面。默认是在location内部，在设置缓存的时候，我们需要把root放在server下面，和listen同级，否则的话会找不到静态资源。

```location ~* \.(?:manifest|appcache|html?|xml|json)$```中的~*后面接正则表达式，这一行是找以这些内容结尾的文件，然后给这些内容设置```Cache-Control: private,max-age=0```。

其他内容设置的```max-age=31536000```代表缓存一年。

## 压缩设置

压缩需要在编译安装的时候添加一个参数，--with-http_gzip_static_module，否则gzip是无效的。yum安装的nginx默认带有这个参数。

压缩设置这里我们在server下添加对应的选项：

```
server {
   gzip on;
   gzip_proxied any;
   gzip_comp_level 3;
   gzip_min_length 1024;
   gzip_types text/css text/xml application/javascript text/plain application/json;
}
```

gzip_types 对对应的文件类型开启gzip压缩。text/html已经包括在默认项目中，不需要在列出。我们只对文本进行压缩，不需要对图片类型进行压缩。
对图片进行压缩太消耗CPU资源。
gzip_min_length 1024; 只对1k以上的内容开启压缩，太小的效果不明显甚至增大体积。
