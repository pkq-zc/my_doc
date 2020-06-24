# win下安装redis5

目前最新版本redis已经到了6.x了.但是以前官网提供的windows版本只到3.x,对于后面的版本需要我们自己编译.  
本文安装环境:```Win10```+```redis-5.0.5```+```cygwin-x86_64```

## 安装cygwin

下载地址:```https://cygwin.com/setup-x86.exe```  
下载好了安装包直接下一步安装即可.在途中需要选择安装哪些包.这里我们需要下面这几个:
![gcc相关](https://i.loli.net/2020/06/24/mNRJAEFMjKvuq3t.png)
![make](https://i.loli.net/2020/06/24/RH4fLJo2hBS5jwv.png)
如果上面有的包漏选了也没关系,再次运行安装包,选择遗漏的安装即可.

## 下载redis5并编译

下载地址:```http://download.redis.io/releases/redis-5.0.5.tar.gz```

- 下载完成解压到任意目录
- 删除```redis-5.0.5\deps\```中的文件夹```hiredis```,从github上下载新的```hiredis```.github地址:```https://github.com/redis/hiredis```
- 在```redis-5.0.5\deps```目录下执行```make hiredis jemalloc linenoise lua```
- 上一步完成之后,在```redis-5.0.5```下执行```make```

经过上面几个步骤.在```redis-5.0.5\src```目录下会生成多个```.exe```文件.这些就是在win下编译好的程序了.在```Cygwin Terminal```中运行redis服务和redis客户端.  
![服务端](https://i.loli.net/2020/06/24/pfSxYhWLRXBVsyq.png)
![客户端](https://i.loli.net/2020/06/24/7BtFcmlTn1uqehj.png)  

上面编译好的```.exe```并不能直接点击直接运行.它目前只能在```Cygwin Terminal```中才能运行.我们将```redis-5.0.5\src```下生成的```.exe```文件拷贝到一个文件夹,同时将根目录下```redis.conf```也拷贝同一个文件夹中.最后将```cygwin1.dll```和```cyggcc_s-1.dll```这两个文件拷贝到这个目录即可.关于这两个文件在哪,这个要看你安装```Cygwin```在哪.在你安装的目录```cygwin\bin```下,能找到上面所提示的两个文件.  
![文件列表](https://i.loli.net/2020/06/24/PXk7VbfpYMn9WxE.png)
个人分享地址:```链接: https://pan.baidu.com/s/1N8DJBlv_t3rttmNANARpEg 提取码: rcat ```