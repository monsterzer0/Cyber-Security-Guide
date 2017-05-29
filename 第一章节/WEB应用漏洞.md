
### WEB应用漏洞:
  
  现在面向用户提供的服务大多数都使用WEB技术开发.WEB技术分为前端与后端,WEB前端技术与客户端相比更加适合移动端,WEB后端开发则更方便快速,两者还有一个共同点是支持跨平台.对于WEB前端技术来说,各个版本的浏览器的实现不同,但是具体的渲染过程和逻辑相同,于是开发出来的页面能够在不同的平台上的不同浏览器都能够正常浏览(虽然某些浏览器会存在些特性,在开发的时候还需要针对该版本做一些适配,但是不影响整体);对于WEB后端技术来说,同样的代码能够在不同的平台上运行,也更方便运维人员部署生产环境,WEB后端技术相比于传统的系统开发,更容易开发者学习和使用,大大提高开发效率.WEB技术普及之后,越来越多的WEB开发者找到工作,但是这些开发者的水平参差不齐,导致写出来的代码存在严重的安全问题,下面挑选一些比较重要的漏洞来讨论<br/>
  
  *远程代码执行之动态语言的eval
  
  动态语言与静态语言的区分是:静态语言需要通过编译器编译,编译出来的文件能够直接执行;动态语言则是需要依赖于解析器来解析文本中的代码.在大多数的动态语言(比如:Python,JavaScript,PHP等)下都能看到`eval()`的身影,`eval()`提供了一个入口,让解析器可以执行指定的代码.有些场景里面,`eval()`函数能够让程序更加灵活,**但往往因为这些灵活性,没有对代码做严谨的逻辑设计,那么很有可能会产生漏洞**,举个例子:<br/>
  
```javascript
//  付款系统前端代码

submit.onclick = function () {
    var upload_select = '';

    for (select_index in select_list)
        upload_select += select_list[select_index].get_price() + '+';  //  把所有商品的价格

    if (upload_select.length)
        upload_select = upload_select.substr(0,upload_select.length - 1)

    if (take_discount)
        upload_select = '(' + upload_select + ')*' + discount_number;  //  构造中括号计算表达式

    if (upload_total_price(upload_select))
        alert('pay success');
    else
        alert('pay error');
}
```
  
```php
//  结算系统代码

if (isset($_GET['price']) && isset($_GET['payid'])) {
    $upload_price = $_GET['price'];
    $payid = $_GET['payid'];

    eval('$pay_price =' + $upload_price);  //  使用eval()计算所有的商品价格与折扣

    build_pay_link($pay_price,$payid);  //  创建结算支付链接
}
 
```

  注意代码`eval('$pay_price =' + $upload_price);`.首先,发往服务器的GET请求是可以被任意构造的,所以$_GET['price']和$_GET['payid']的值可以被控制.再往下看,`eval()`函数的参数构造里其实是执行代码`$pay_price = $_GET['price']`,当参数`price`为100时,`eval()`函数会执行代码`$pay_price = 100`,把值100赋给`$pay_price`变量;如果我们构造参数为`1;phpinfo();`时,`eval()`函数会执行代码`$pay_price = 1;phpinfo();`,执行完对`$pay_price`的赋值后输出php运行环境的信息.<br/>

  eval()函数的另一滥用则是一句话木马的应用,示例代码:<br/>

```php
<?php eval($_GET['upload']);>
```

  `eval()`函数会把`$_GET['upload']`传递过来的数据当作PHP代码来执行,上传的代码可以结合PHP文件API读写任意目录和文件,也可以结合SQL接口读写只有这台服务器才可以访问的数据库,还能够使用system()执行命令和上传到服务器的恶意程序.<br/>
  
  *远程代码执行之文件上传与服务器解析
  
  大多数的建站系统都有文件上传功能,和文件上传相关的功能有头像上传,附件上传,数据导入等.引发文件上传漏洞的根本一点是上传的文件在保存的时候没有对文件设置正确的拓展名,而是保存成`.PHP``.ASP``.JSP`等可以被服务器解析执行的脚本文件,然后再去访问这些文件,里面保存的恶意代码就会执行,从而导致漏洞的产生.<br/>
  关于文件上传漏洞的产生,可以分为两种:代码设计失误与服务器漏洞利用.下面分别举例子:<br/>
  
  
  
  
  
  *远程与本地文件包含
  
  文件包含与`include`和`require`语句有关.在PHP里为了模块化设计程序,采用`include`和`require`从另外的文件导入PHP代码到当前的PHP文件执行.如果包含语句可以被用户控制,那么同样也可以做到远程代码执行,举个例子:<br/>
  
  很多PHP框架采用MVC结构设计,所以经常能够看到类似于这样的链接:<br/>
  
```
  http://xxx/index.php?action=packageRead&id=11327  action
  http://xxx/claimAction.do?cmd=claim_view&sn=61079  cmd
  http://xxx/forum.php?mod=viewthread&action=printable&tid=369231  action
```
  
  第一条链接使用`action`,第二条链接使用`cmd`,第三条链接使用`mod`和`action`作为页面的控制,比如控制页面应该是输出数据还是在编辑状态,是使用A模板还是B模板.对于`action`这样的页面控制,有部分开发者使用`switch`语句进行控制,另一部分的开发者使用`include`等包含语句导入代码,于是代码里面会出现以下这段代码<br/>
  
```PHP
if (isset($_GET['action'])) {
    $action = $_GET['action'];

    include $__DIR__ . '/' . $action . '.php';
}
```
  
  输入链接为:`http://xxx/test.php?action=edit`时,包含当前目录下的edit.php文件<br/>
  输入链接为:`http://xxx/test.php?action=../other`时,包含上一层目录下的other.php文件<br/>
  输入链接为:`http://xxx/test.php?action=../../../../../etc/passwd;`时,根据多次构造`../`测试目录的深度,读取到`/etc/passwd`,由于`;`的存在,导致`/etc/passwd;.php`被系统过滤成`/etc/passwd`,泄漏远程服务器上的帐户信息<br/>
  
  `include`还有一个特性:无论文件的拓展名什么,只要找到`<?php ... ?>`,PHP解析器就会执行其中的代码,于是可以构造带有恶意代码的图片并且上传到服务器.假设上传的图片保存在`/upload`目录,可以构造`http://xxx/test.php?action=/upload/img-123jsd12.png;`触发远程代码执行.<br/>
  
  毕竟有部分的WEB开发者从培训班或者大学毕业没有什么项目经验就直接写比较复杂后台代码,所以写出<br/>
  
```PHP
include $_GET['module']
```

  这样的代码不足为怪,前面说到的文件包含漏洞,因为在构造包含文件的路径时,已经被本地目录所限制,于是只能包含服务器上的文件,故称本地文件包含;这段代码能够直接包含可控的输入,于是可以在此传递URL链接,让`include`包含语句直接包含远程文件,达到远程代码执行的效果<br/>
  
  *远程命令执行
  
  system exec 
  
  
  
  
  *SQL注入
  
  有关SQL注入的知识可以写一本书那么厚,关于SQL注入,根本的一点是没有对传递到SQL语句的数据进行严格的过滤,这些数据把原来的SQL结构打乱,变成恶意的SQL语句,比如这样<br/>
  
```sql
SELECT user_id FROM user_database WHERE user_name='$user_name'
```

  正常的查询,SQL语句如下:<br/>
  
```sql
SELECT user_id FROM user_database WHERE user_name='LCatro'
```
  
  作者稍微偷了下懒,没有对`$user_name`输入的数据进行过滤,所以构造`$user_name`为`LCatro'`<br/>
  
```sql
SELECT user_id FROM user_database WHERE user_name='LCatro''
```
  
  注意,此时的SQL字符串闭合已经被打乱,访问这条SQL语句将会出现错误.再构造用户名为`LCatro' UNION SELECT user_password FROM user_database //`,SQL语句会组合成:<br/>

```sql
SELECT user_id FROM user_database WHERE user_name='LCatro' UNION SELECT user_password FROM user_database //'
```
  `SELECT user_id FROM user_database WHERE user_name='LCatro'`这部分SQL语句是正常的,但是黑客插入了`UNION SELECT user_password FROM user_database`语句,使得从数据库里的查询连带返回了用户的密码信息.最后还需要注意的一点是,因为原来SQL语句里面的`'`还存在,如果不使用`//`注释掉`'`,那么会导致SQL语句在执行的时候产生语法错误<br/>
  
  代码里面出现SQL注入,基本上数据库上的数据都能够访问,黑客把拉取数据库上的数据称为拖库,SQL注入漏洞除了能够拖数据之外,也可以做到远程命令执行,讨论这些细节超出本文的范畴,请读者自行搜索.<br/>
  
  对于防御SQL注入的方法很简单,使用`str_replace()`或者`addslashes()`函数过滤敏感字符即可<br/>
  
  
  *任意文件下载
  
  任意文件下载漏洞会让服务器上的任意文件泄漏给黑客,比如:帐户密码文件,数据库等重要文件.有些WEB项目的设计里,需要提供给用户下载文件的地方很有可能存在这样的漏洞<br/>
  
```
  https://xxx/members/dl.php?download=document.pdf
```

  在代码设计里,会从`download`中读取内容,然后再到本地目录里打开文件,再传输到用户.那么可以构造这样的链接<br/>

  `https://xxx/members/dl.php?download=dl.php`,读取当前目录下`dl.php`文件,直接把`dl.php`的源码泄漏出来<br/>
  `https://xxx/members/dl.php?download=/database/main.db`,读取服务器上的数据库文件<br/>
  `https://xxx/members/dl.php?download=../../etc/shadow`,读取服务器上的用户密码文件<br/>

  
  聪明的你肯定发现,上面所说的漏洞全部都是因为用户的输入而导致的.**那些敏感函数和敏感语句,千万不要让用户的输入流向这里,如果一定要依靠用户的输入来执行,千万注意要对输入的数据做好过滤**<br/>
  
  
  *针对用户的攻击
  
  