> 用过很多博客程序了，有动态的，也有静态的，只能说各有所长吧。

今天这个程序，基于serverless，运行在cloudflare workers上，除了域名需要付费(也可以去[Freenom](https://freenom.com)申请免费的，但是不推荐)其余都是完全免费的。
## 演示
![iloli.icu](https://i.loli.net/2019/11/23/6EFQW7DmPa9s52z.png "iloli.icu")

## 源代码
[github ](https://github.com/kasuganosoras/cloudflare-worker-blog "github ")
这份代码是未经修改的，你也可以选择自行修改
## 搭建过程
1. 进入[cloudflare官网](https://cloudflare.com "cloudflare官网")，登录你的账号
2. 进入workers设置，创建一个新的worker
![workers](https://i.loli.net/2019/11/23/BhgxGZia1R7o9Lf.png "workers")
![创建完毕](https://i.loli.net/2019/11/23/7W2a3RJy5l6SDNj.png "创建完毕")
3.  点击editor，将worker.js内容粘贴到左边，save，就完事了
## 自定义域名
获取到自己的****.workers.dev 
域名 Cname 到(****.workers.dev)
然后去 Workers 点击 ADD route
输入自定义域名，例如 gd.mjj.com/* 后面要加上 /*，下面的 workers 选择刚才的项目
