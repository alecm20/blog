# Web缓存简介

前端开发中，页面的加载时间是相当重要的一个指标，加载的时间直接影响着用户的体验。
合理利用前端缓存，能够有效的提升加载速度，优化用户体验。同时，使用缓存还能够减轻服务器压力，
节省网络传输费用。

下面就几个方面对缓存进行讲解。

## 命中率
* 缓存命中率
由缓存提供的请求，占总数的比例，称为缓存命中率，或者文档命中率，一般用百分比描述。缓存命中率介于0%到100%之间。
我们对资源进行设置，希望提高缓存命中率的比例。

* 字节命中率
缓存命中率是根据文件的数量来计算的比率。可能有些文件很大，有些很小，统计缓存的总的内容比例的时候，就不太准确。
字节命中率就是根据内容来计算缓存的比例。能够计算所节省的流量。

这两种命中率在评估缓存的性能时，都是有用的。较少缓存命中率，能够减少和服务器建立连接的数量，对降低时延有利。
我们常用的合并请求等，就是用来降低时延。字节命中率说明节省了多少在网络上传输的流量，提高字节命中率对节省带宽有利。

所以，我们的目的就是尽量提高命中率，包括文档命中率和字节命中率。

## 缓存再验证

服务器上的原始内容可能会发生变化，这时候需要对缓存的副本进行检测，查看副本是不是和服务器上最新的内容一致。
这些新鲜度的检测被称为**HTTP再验证**。为了有效的进行缓存的验证，HTTP定义了一些特殊的请求，使得不需要完整传输整个文件，
就能够检验文件是否是最新的。
下面的图说明了此过程。
![缓存验证](assets/cache-hit.png)
上图中的最后一种情况，检测之后，文件没有被修改，服务器会返回304 Not Modified的响应，代表内容未修改，则缓存副本是有效的，
能够提供给客户端使用。这种情况称为**再验证命中**。

我们一般用HTTP的If-Modified-Since首部来验证缓存的有效性。在Request Headers中加上此字段，服务器会根据此字段进行检验。

* 再验证命中
服务器内容未修改，HTTP响应会返回304 Not Modified响应，此响应不包括内容。
* 再验证未命中
验证之后发现内容已经被修改，服务器会发送一个带有完整内容的200 OK的响应。
* 对象被删除
服务器上的内容如果被删除，返回404 Not Found响应，同时缓存副本也会被删除。

## 缓存控制首部

服务器可以按照HTTP提供的方法来指定文档过期前将其缓存多久。以下内容按照控制度的递减顺序排列：

* 响应中附带 Cache-Control: no-store 到首部中
* 响应中附带 Cache-Control: no-cache 到首部中
* 响应中附带 Cache-Control: must-revalidate 到首部中
* 响应中附带 Cache-Control: max-age 到首部中
* 响应中附带 Expires 到首部中
* 不附带信息，让缓存自己确定过期日期

### no-store与no-cache
```
Pragma: no-cache
Cache-Control: no-store
Cache-Control: no-cache
```
标示为no-store的内容会禁止使用缓存。

标示为no-cache的内容，实际上是可以使用缓存的，但是在使用缓存前需要先对缓存进行验证，验证有效之后，可以使用缓存。

其中，Pragma: no-cache字段是为了兼容HTTP/1.0。HTTP/1.0中不可以使用Cache-Control，所以在HTTP/1.1中，为了兼容提供了
Pragma: no-cache字段。在Cache-Control可用的情况下，不应该使用Pragma: no-cache。

### must-revalidate

Cache-Control: must-revalidate代表，没有和服务器进行验证之前，不能使用缓存副本。


### max-age

Cache-Control: max-age表示服务器传送文档给客户端之后，客户端可以认为此文档是新鲜的状态的秒数。注意此处单位是秒，不是毫秒。

Cache-Control: max-age=3600 表示将内容缓存一个小时。

Cache-Control: max-age=0，代表每次都请求新的文档。

s-maxage与max-age类似，但是只能用于共享缓存，比如CDN。

### Expires

Expires代表缓存过期的时间，不推荐使用此首部，因为服务器的时间和客户端的时间可能不同步。此字段是HTTP/1.0的内容。
应该优先使用Cache-Control首部。

Expires格式如下：

Expires: Sun, 10 Sep 2017 09:55:09 GMT

### 试探性过期
如果响应首部中，没有缓存控制相关内容，则缓存自己计算过期时间。

一般使用LM-Factor算法计算缓存过期时间。
LM算法会计算缓存保存的时间和上次修改的时间的差值，乘上LM-factor的值，得到的时间，就是缓存的有效时间。

比如：一个文档，缓存的时间是17-01-01，最后修改的时间是16-01-01，则差值是1年。我们设置LM-factor=0.2，则缓存时间是两个月，
过期时间则是17-03-01，在此之前的缓存都是有效的。

一般我们可以将LM-factor设置成0。
如果上次修改的时间离现在很久，即使LM-factor很小，计算出来的值，也可能导致用户使用了不新鲜的副本。

### 其他

另外，Cache-Control还有其他选项可以设置。

Cache-Control: public 代表允许任何设备缓存，包括代理等。

Cache-Control: private 只允许用户缓存

Cache-Control: only-if-cached 只从缓存中去数据，不用去检查是否有新数据

Cache-Control: max-stale[=<seconds>] 允许的缓存最大过期时间

Cache-Control: min-fresh=<seconds> 保证在这段时间内，缓存不能过期

Cache-Control: no-transform 禁止代理转换内容的格式等

## 缓存验证相关首部

HTTP定义了5个条件请求首部，If-Modified-Since和If-None-Match是常用的验证缓存有效性的首部。

### If-Modified-Since

If-Modified-Since: Date是最常见的验证缓存的首部。

如果自指定日期后，文档被修改了，则If-Modified-Since为真，响应中会发送新的内容，并有一个新的过期日期。

如果自指定日期之后，文档没被修改，则会返回304的响应。不发送内容主体，会返回新的过期日期。

If-Modified-Since可以和服务端的Last-Modified配合工作。Last-Modified代表文档的最后修改日期。在响应的首部中，可以包含Last-Modified字段。
如果文档没被修改，则这两个字段应该一致。如果不一致，则代表文档被修改了，需要返回新的内容。

### If-None-Match

有些情况只基于日期比较可能不能给满足要求。
比如：
* 修改日期变化，但文件内容未变化；
* 文件内容变化，但变动不重要；
* 文件变动在一秒内，最后修改时间的精确度不够；
* 服务器无法确定最后修改日期。

为了解决这些问题，加入了If-None-Match首部。该首部是基于标签进行比较的。
发送请求时，请求首部中带上一个If-None-Match首部，该首部中是特定的标签，服务器接收到请求后，会将此标签和服务端内容比较，
如果一致，则内容有效，返回304响应，否则返回新的200响应。响应中会带上ETag首部，指明资源的标签。
可以在If-None-Match中带上多个标签，例如：

If-None-Match: "v1.0", "v1.1", "v1.2"

表示带有这些标签的内容在缓存中已经有了。

## 缓存方案制定

上面讲的是缓存的原理，下面说下具体的设置。

比如我们有一个网页，页面中有一个文件：

```html
<html>
<head>
<meta charset="UTF-8">
             <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
                         <meta http-equiv="X-UA-Compatible" content="ie=edge">
             <title>Document</title>
</head>
<body>
  <script src="http://a.com/a.js"></script>
</body>
</html>
```

上面的文档中，有一个JS文件。

对于HTML文档，我们希望每次使用前都要向服务器验证。因为我们必须保证HTML文档是新鲜的，否则其他资源的请求可能是错误的。

因此我们可以设置HTML的请求头的Cache-Control: max-age=0, private，这样每次打开网页的时候，都会验证页面的有效性。

对于上面页面中的js文件，则不太一样。这种属于页面中的静态资源，我们希望文件内容没有变动的时候，不要重新请求。

我们可以先给它设置：Cache-Control: 604800 一周的时间。但是一周内文件变动的话，文件也不会被请求。

则我们可以给它加上版本号a.js?v=1.2，内容修改的话，修改版本号即可。但是，修改版本号一般是对所有文件都设置，如果有的文件没有变动，
也需要重新请求，这样则会浪费没必要的流量和带宽。所以现在的方案都是基于文件内容，生成hash值放在文件中，例如这样：a.4c127bb5.js。

这样设置的话，文件内容变化之后，HTML文档中的资源路径也会变化，所以不用担心缓存的问题。如果内容不变，则可以利用缓存。

加上hash之后，我们则可以对带上hash的静态资源设置较长的缓存时间。

我们可以这样设置：Cache-Control:public,max-age=31536000,immutable 这样就给该资源设置了一年的过期时间。

public我们上面说过，代表允许代理缓存该内容。immutable表示该内容如果不过期，是不会在服务器上改变的。因此可以不用给服务器发送请求验证此文件的新鲜度。
因为我们的内容是基于hash的，所以我们能够保证同一个路径的资源是不会变化的，因此也就没必要去重新验证了。此项参考的facebook的设置。这种情况下，
这种资源是完全不会发起新的请求的。对于一般的304请求，也需要验证的时间，这种则省略了验证的时间，是**缓存命中**的，304的称为**再验证命中**。

上面的对于js的设置，同样也适用于css以及图片等静态资源。我们都可以为其设置一个较长的过期时间，但是我们要保证他们的文件名都是基于文件内容的hash值的。

HTML内的静态资源能够加hash戳，我们可以对其设置较长的过期时间，而HTML文档则不能给这样设置，也有这个的原因。

给文件名加上hash值，用现在的打包工具进行设置已经很简单了。我们用webpack可以对资源名设置chunkhash，来为文件名加上MD5戳。
具体自行查阅官方文档即可。

## Nginx配置

我们的前端资源使用的Nginx服务器，下面是静态资源的样例设置：

基础Nginx配置不再列出，下面都是针对静态资源的设置项：

端口号、server_name、root目录根据自己的目录设置做修改。

~* 后面加正则表达式，表示匹配root下面的资源类型，对其做相应的设置。

下面的配置参考的这里：[缓存头设置](https://github.com/h5bp/server-configs-nginx/blob/master/h5bp/location/expires.conf)

```
server {
   listen       8080;
   server_name  name;
   root   /;
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
      gzip off;
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


有一点要注意，设置max-age=31536000的几项内容，要确保文件内容变动时，文件名会被修改。

这样设置之后，就给对应的资源设置了缓存。


