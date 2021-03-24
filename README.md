# 代码审计之逻辑漏洞

### 熊海CMS v1.0

这次采用黑盒+白盒的方式，使用之前所学知识，对熊海CMS进行审计。

拿到源码先搭建环境安装运行，然后观察目录结构

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193052.png)

使用法师审计系统自动审计一波，先看工具审计到的漏洞点

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193114.png)



#### 文件包含

漏洞点：

- /index.php
- /admin/index.php

扫描结果第一行就是<u>网站入口文件</u>可能存在文件包含漏洞，那先来看 **index.php** 吧

内容很简单只有7行代码，载入文件的函数**include**参数中存在可控变量**$action**，且无任何消毒措施

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193128.png)

是**文件包含漏洞**没错了，这里载入的是**files目录**下的**php**文件，后缀固定死了的

在根目录下创建一个1.php，内容是`<?php phpinfo();?>`，看看能不能包含

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193141.png)

修复建议：

根据include函数的路径，猜测这里只需要包含files目录下的文件，采用白名单限定$action的值即可

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193157.png)

修复后：

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193208.png)

#### SQL注入 - 前台

漏洞点：

- /files/software.php
- /files/content.php
- 存在浏览计数的页面都有此漏洞

在**/files/content.php**中，虽然对**$_GET['cid']**进行了转义处理

但是第19行中的SQL语句条件变量无单引号保护，完全无视了**addslashes**函数

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193222.png)

在新闻页面使用报错注入或盲注都可以成功~

```
sqlmap.py -u "http://127.0.0.1/?r=content&cid=15*"
```

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193235.png)

###### 修复建议：

将第19行的**$id**用单引号括起来

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193301.png)

修复后

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193317.png)



#### SQL注入处的反射性XSS

漏洞点：

- /files/software.php
- /files/content.php
- 存在浏览计数的页面都有此漏洞

还是上一个漏洞的地方，没有过滤XSS字符

必须在xss前加单引号，使原来的页面加载完毕才能执行JavaScript代码

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193336.png)

###### 修复建议：

此处的cid只是数字，使用**intval**函数强制转换类型

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193350.png)

工具审完啦，接下来浏览网站功能点查找漏洞



#### 反射性XSS

漏洞点：

- files\contact.php
- files\download.php
- files\list.php

这里以 **files\contact.php** 文件分析，在第23行输出了可控变量**$pages**，在12行可以看到只对**$_GET['page']**转义了单引号，无法防御XSS攻击

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193409.png)

漏洞验证

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193426.png)

###### 修复建议：

用**htmlentities**函数对输出进行实体化处理

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193447.png)



#### 存储型XSS - 评论处

在 **files\submit.php** 文件中，将评论的昵称、邮箱、内容等插入了数据库

但是一点都没有消毒！除了对**$content**进行了判断

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193504.png)

在评论处可以提交昵称、邮箱、网址、评论内容

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193522.png)

但是显示评论和留言的地方只把昵称提取出来了，所以只有昵称处有存储型XSS

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193537.png)

昵称处插入XSS后，会在后台触发，同时不需要审核就在联系页面的留言处显示

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193548.png)

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193604.png)



###### 修复建议：

用**htmlentities**函数对输出进行实体化处理

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193620.png)

前台看完了，接下来看后台



#### SQL注入 - 后台登录处

在 **admin\files\login.php**中，第10行中SQL语句存在可控变量且未消毒，存在SQL注入漏洞

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193632.png)

抓取登录页面的数据包

```
POST /?r=submit&type=message HTTP/1.1
Host: 127.0.0.1
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; rv:18.0) Gecko/20100101 Firefox/18.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-cn,zh;q=0.8,en-us;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
DNT: 1
Referer: http://127.0.0.1/?r=contact&page=%3Cscript%3Ealert(1)%3C/script%3E
Cookie: UM_distinctid=16bf499773fef-0abf9fe78a00358-49594134-100200-16bf499774114b; CNZZDATA3801251=cnzz_eid%3D1098415049-1563176819-%26ntime%3D1563184667; PHPSESSID=lkur749ro3sn8ut2u370ip55k2
REMOTE_ADDR: 8.8.8.8'
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 87

cid=0&name=111&mail=111&url=111&content=ëÐÐ&save=%E6%8F%90%E4%BA%A4&randcode=&jz=1&tz=1
```

放到sqlmap里跑一下

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193643.png)

###### 修复建议：

使用**mysql_real_escape_string**函数进行过滤

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193654.png)



#### 垂直越权登录后台

查看后台时，发现每个文件头部都引入了checklogin.php文件，一看名字就是检测身份凭证的

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193714.png)

进去一看，只有7行hhh，毫无防护措施

只要cookie中user的值不为空，就可以进入后台

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193726.png)

访问后台时，用burp拦截修改cookie值，成功进入后台耶

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193743.png)

###### 修复建议：

在管理员登陆成功后设置特殊的session值

```
session_start();
$_SESSION['user']= 'admin';
```

然后在需要控制权限的文件头部判断session值是否合法

```php
<?php
	if($_SESSION['user'] != 'admin'){
		alert_href('请重新登录！','login.php');
	};
?>
```



#### SQL注入-  - 后台表单处

这里以发布文章功能点为例

在 **admin\files\newwz.php** 文件中，第83行的SQL语句中存在可控变量且未消毒，存在SQL注入漏洞

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193805.png)

此处正是后台发布文章的地方，来到该功能点，随意输入一些内容提交然后抓取数据包，保存为txt

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193819.png)

使用sqlmap进行测试

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193831.png)

###### 修复建议：

使用**mysql_real_escape_string**函数进行过滤



#### 存储型XSS - 后台发布内容处

在 **admin\files\wzlist.php** 文件中,184行未经消毒就输出了查询的变量

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193842.png)

两者一结合，先将xss payload存到数据库中，然后在wzlist.php将查询结果显示在页面中，即存在存储型XSS

漏洞点在后台的发布文章处，随意输入一些内容提交

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193856.png)

用burp拦截修改成XSS payload

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193908.png)

然后访问内容管理 --》文章列表，触发XSS

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193922.png)

在前台浏览文章时也会触发XSS，但是标题不能插payload，否则文章不能显示在列表中（可以插到作者处）

###### 修复建议：

用**htmlentities**函数对所有输出进行实体化处理

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193933.png)



#### CSRF - 后台资料设置处

在 **admin\files\manageinfo.php** 文件中，表单没有设置token，也没有校验referer值，对操作的来源没有限制，存在CSRF漏洞

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193943.png)

这里以名称为例进行展示

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716193956.png)

用burp生成临时poc，修改管理员的名称为test

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194008.png)

当管理员点击恶意链接时，操作已经完成

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194026.png)

###### 修复建议：

- 对每次操作增加csrf token验证
- 对每次操作判断Referer头
- 对操作页面增加验证码

-----

熊海CMS v1.0作为入门级代码审计肥肠合适啊，代码几乎没有防护措施。

后台表单处的漏洞原理都差不多，就没必要都去审计啦。



### 大米CMS 5.5.2

#### 支付逻辑漏洞

大米CMS有购物功能，且支持站内支付。一般自己写的支付功能都不太成熟嘛，这就有了可乘之机。

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194040.png)

寻找到订单支付模块的代码，看看有没有防护措施

在**Web/Lib/Action/MemberAction.class.php**文件中，第585行-第640行之间是站内支付的功能代码

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194101.png)

可以看到在第608行对价格进行了重新查询，但没有对数量进行处理，可以在**$_POST['qty']**参数上做文章

来到前台随便选择一个产品进行购买，在下单页面选择站内扣款

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194113.png)

提交订单，使用burp抓包拦截

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194128.png)

提交的内容被URL编码了，解码后可以看到提交的参数都是数组，圈住的地方就是数量和价格了

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194146.png)

修改数量为 -1 后放行，订单提交成功

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194157.png)

在会员中心的订单页面可以查询到

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194212.png)

站内余额也增加了

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194225.png)

###### 修复建议：

在第594行计算的总金额中，对qyt参数替换掉负号，禁止商品数量为负数

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194237.png)

在后台将刚才加的6000元减掉，现在余额还有100。

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194251.png)

再尝试购买 -1 个产品，来看下效果

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194304.png)



### 热剧CMS2.1

#### 验证码失效导致暴力破解

在**/system/verifycode.php**验证码生成页面，直接生成四个随机数然后赋给SESSION，验证码使用后不会过期，相当于没有验证码hhh

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194315.png)

这是登录检测页面**/admin/cms_login.php**，也没有限制错误次数。

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194327.png)

这样一来可以进行暴力破解啦~

抓取后台登录页面的数据包，直接爆破

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194346.png)

###### 修复建议：

使用完**$_SESSION['verifycode']**后就关闭session

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194359.png)



#### 垂直越权登录后台

后台页面都是通过 **/admin/cms_check.php** 文件校验身份是否合法

但是该文件只检测cookie中admin_name值是否为空，而cookie是可控的

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194412.png)

使用burp修改cookie值进行访问，可以看到成功登录后台

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194424.png)

###### 修复建议：

在 **/admin/cms_login.php** 中，登陆成功后设置特殊的session值

```
session_start();
$_SESSION['user']= 'admin';
```

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194439.png)

将 **cms_check.php** 文件判断cookie的方法改为判断session

```php
<?php
	if($_SESSION['user'] != 'admin'){
		alert_href('请重新登录！','cms_login.php');
	};

?>
```

修复成功~

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194454.png)



### FeiFeiCms4.0.181010

#### 无限刷影币

用户注册成功后会收到赠送的10影币，这里可以通过注册功能刷影币

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194504.png)

首先来看注册赠送影币的逻辑代码，在**Lib\Lib\Action\Home\UserAction.class.php**文件第198行的post函数中

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194516.png)

首先观察下**ff_update**函数是做什么的，在**\Lib\Lib\Model\UserModel.class.php**文件

是新增或更新用户的功能，但是判断**$data['user_id']**存在的话进行save函数处理

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194526.png)

跟进**save**函数，在**\Lib\ThinkPHP\Lib\Think\Core\Model.class.php**文件中就是负责更新数据

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194538.png)

这样一来，如果注册的时候**$data['user_id']**参数存在的话，只是更新数据，而且返回true

**$info**就会为true，就可以无限送注册积分

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194549.png)

影币成功到手~

![](https://raw.githubusercontent.com/bigbighippo/hippoBed/master/img/20190716194606.png)

###### 修复建议：

将**ff_update**函数的新增和更新功能分开写，功能混用就会导致逻辑问题
