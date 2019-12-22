### 记一次关于nginx配置升级的的问题
本次升级的目的是使得nginx代理服支持TLSv1.3，关于TLSv1.3的优点简要说明就是删除了不安全的密码组件，优化HTTPS握手时间来提高网站访问速度。具体介绍可以查看[知乎专栏文章](https://zhuanlan.zhihu.com/p/28850798)，TLSv1.3 标准已于 2018 年 8 月正式发布为 RFC 8446 。

> TLSv1.3使用了复杂的密钥导出过程，增强了最终使用的密钥的安全性。同时简化了所使用的加密算法，废弃了RC4、3DES、MD5、SHA1、AES-CBC等加密算法，删除了压缩、重协商等具有漏洞的机制，大大精简了协议。



#### 准备：

openssl-1.1.1b

nginx-1.16.0

Brotli（压缩算法）

注：Brotli 也是一种压缩算法，它是由谷歌开发的一个更适合文本压缩的算法，具有更好的压缩比。现在主流浏览器都已经支持了，而且也能与 gzip 共存，如果浏览器支持 Brotli 就会优先使用。



#### 升级：

<https://www.ilouis.cn/practice/nginx_enable_tls_1_3.html>

具体升级方式详见git上的[文档](http://10.32.64.222/common/server-deployment/blob/master/README.md#nginx%E5%AE%89%E8%A3%85)。

同时在原有nginx-1.14.0基础上进行升级的脚本在文档下方。



### 问题1：

#### 问题描述：

在nginx.conf中开启TLSv1.3配置后，在浏览器并未生效，在chrome控制台 -> Security中可以看到对应网站依然启用的是TLSv1.2。

```bash
# nginx.conf
ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
ssl_ciphers ...;#略
```

通过include方式下引入了子目录下(eg: include/*)其他业务有关配置文件，在其配置文件中也引入了对应的配置，但是并未开启TLSv1.3，如下所示。

```bash
# service1.conf
ssl_protocols TLSv1.1 TLSv1.2;
ssl_ciphers ...;#略
```

在启动nginx服务后，业务配置会将`nginx.conf`中的配置覆盖掉，在业务配置文件中对应的配置删除重启服务即可。



### 问题2：

#### 问题描述：

在按上述方法更新海外服时，在`./configure ...`时出现了如下参数无效，(`–with-ld-opt`表示传递给C链接器的其他参数)

```bash
--with-ld-opt='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' 
```

去掉该段参数重新`configure`，提示如下错误，导致后续的操作失败。根据提示可能和内核版本有关。

```bash
./configure: no supported file AIO was found
Currently file AIO is supported on FreeBSD 4.3+ and Linux 2.6.22+ only
```

通过`cat /proc/version`查看内核信息：

海外服的内核信息：

```bash
Linux version 3.2.0-4-amd64 (debian-kernel@lists.debian.org) (gcc version 4.6.3 (Debian 4.6.3-14) ) #1 SMP Debian 3.2.63-2
```

国内服内核信息

```bash
Linux version 3.16.0-4-amd64 (debian-kernel@lists.debian.org) (gcc version 4.8.4 (Debian 4.8.4-1) ) #1 SMP Debian 3.16.43-2+deb8u2 (2017-06-26)
```

根据查询得到，该部分和AIO（异步IO操作有关）。

将参数名`--with-ld-opt`改成`--with-cc-opt`即可通过编译。（`-with-cc-opt` 设置将添加到CFLAGS变量的其他参数。 ）

```bash
--with-cc-opt='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' 
```

具体问题发生原因不确定，问题可能是海外服的Linux内核版本过低导致编译链接的方式有些许不同导致。

