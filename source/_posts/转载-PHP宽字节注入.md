---
title: '[转载]PHP宽字节注入'
date: 2021-02-22 07:40:44
tags: ['php']
categories:
cover: https://upload-images.jianshu.io/upload_images/9113981-f59156b3b123c9d2.png?imageMogr2/auto-orient/strip|imageView2/2/w/716/format/webp
---
<meta name="referrer" content="no-referrer" />

# [转载]PHP宽字节注入

在PHPMySql语句中存在着宽字节注入漏洞，MySQL宽字节注入漏洞是SQL注入漏洞攻防技术相互促进的一个典型例子。
了解宽字节注入之前先了解一下PHP中经典SQL注入漏洞

### 1.经典SQL注入漏洞

例子1是没有任何SQL注入防护措施的PHP程序，它存在SQL注入漏洞。



```php
<?php
  $name=$_GET['name'];
  $conn=mysql_connect('localhost','root','root');
  if($conn==null){exit("connect error !<br>");}
  mysql_select_db("aaa",$conn);
  $sql="select * from a1 where name='".$name."'";
  $result=mysql_query($sql,$conn);
  while($val=mysql_fetch_row($result)){
      print_r($val);
      print("<br>");
  }
?>
```

> 对该PHP程序的SQL注入POC包括：
> （1） http://127.0.0.1/test/t1.php?name=a'or 'a'='a
> （2） http://127.0.0.1/test/t1.php?name=a'or 1=1 -- %20
> （3） http://127.0.0.1/test/t1.php?name=a'or 1=1 -- %23

其中，%20对应空格，%23对应#的URL编码，该POC在PHP5.4.45+Apache测试成功。

### 2 安全过滤

首先，宽字节注入与HTML页面编码是无关的，笔者曾经看到
`<meta charset=utf8>`
就放弃了尝试，这是一个误区，SQL注入不是XSS。虽然他们中编码的成因相似，不过发生的地点不同。

#### 字符、字符集与字符序

字符(character)是组成字符集(character set)的基本单位。对字符赋予一个数值(encoding)来确定这个字符在该字符集中的位置。

字符序(collation)指同一字符集内字符间的比较规则。

#### UTF8

由于ASCII表示的字符只有128个，因此网络世界的规范是使用UNICODE编码，但是用ASCII表示的字符使用UNICODE并不高效。因此出现了中间格式字符集，被称为通用转换格式，及UTF(Universal Transformation Format)。

#### 宽字节

GB2312、GBK、GB18030、BIG5、Shift_JIS等这些都是常说的宽字节，实际上只有两字节。宽字节带来的安全问题主要是吃ASCII字符(一字节)的现象。

如果对例子1中的PHP程序中的$name变量进行安全过滤，如使用转义函数`addslashes`，`mysql_real_escape_string`，`mysql_escape_string`，则对应的POC全部失效了。

还有一种情况是magic_quote_gpc，php中的magic_quotes_gpc是配置在php.ini中的，他的作用类似addslashes()，就是对输入的字符创中的字符进行转义处理。他可以对`_POST`、`$__GET`以及进行数据库操作的sql进行转义处理，防止sql注入。不过高版本的PHP将去除这个特性。

**转义函数影响的字符包括：**



```cpp
 （1）          ASCII（NULL）字符\x00，

 （2）          换行字符\n，addslashes不转义

 （3）          回车字符\r，addslashes不转义

 （4）          反斜杠字符\，

 （5）          单引号字符‘，

 （6）          双引号字符“，

 （7）          \x1a，addslashes不转义
```

对于例子1进行安全增强后，得到例子2，如下所示。

注意：三个转义函数的功能稍有区别，同时，转义只对字符型SQL注入防范有效，对于数值型SQL注入无效。



```ruby
<?php
$name=$_GET['name'];
//$name=addslashes($name);
//$name=mysql_escape_string($name);
$conn=mysql_connect('localhost','root','root');
$name=mysql_real_escape_string($name);
if($conn==null){exit("connect error !<br>");}
mysql_select_db("aaa",$conn);
$sql="select * from a1 where name='".$name."'";
$result=mysql_query($sql,$conn);
while($val=mysql_fetch_row($result)){
    print_r($val);
    print("<br>");
}
?>
```

### 3 宽字节注入漏洞原理

#### 前要 字符编码问题

通常来说，一个gbk编码汉字，占用2个字节。一个utf-8编码的汉字，占用3个字节。在php中，我们可以通过输出

> echo strlen("和");

来测试。当将页面编码保存为gbk时输出2，utf-8时输出3。 除了gbk以外，所有ANSI编码都是2个字节。ansi只是一个标准，在不同的电脑上它代表的编码可能不相同，比如简体中文系统中ANSI就代表是GBK。

#### 3.1概述

首先我们了解下宽字节注入，宽字节注入主要是源于程序员设置数据库编码与PHP编码设置为不同的两个编码那么就有可能产生宽字节注入

宽字符是指两个字节宽度的编码技术，如UNICODE、GBK、BIG5等。当MYSQL数据库数据在处理和存储过程中，涉及到的字符集相关信息包括：

> - （1） character_set_client:客户端发送过来的SQL语句编码，也就是PHP发送的SQL查询语句编码字符集。
> - （2） character_set_connection:MySQL服务器接收客户端SQL查询语句后，在实施真正查询之前SQL查询语句编码字符集。
> - （3） character_set_database:数据库缺省编码字符集。
> - （4） character_set_filesystem:文件系统编码字符集。
> - （5） character_set_results:SQL语句执行结果编码字符集。
> - （6） character_set_server:服务器缺省编码字符集。
> - （7） character_set_system:系统缺省编码字符集。
> - （8） character_sets_dir:字符集存放目录，一般不要修改。

宽字节对转义字符的影响发生在character_set_client=gbk的情况，也就是说，如果客户端发送的数据字符集是gbk，则可能会吃掉转义字符\，从而导致转义消毒失败。

例如说PHP的编码为 UTF-8 而 MySql的编码设置为了
SET NAMES 'gbk' 或是 SET character_set_client =gbk，这样配置会引发编码转换从而导致的注入漏洞。
这里要说明一小点的是：
SET NAMES 'x'语句与这三个语句等价：
mysql>SET character_set_client =x;
mysql>SET character_set_results =x;
mysql>SET character_set_connection =x;
也就是说你设置了 SET NAMES 'x' 时就等于同时执行了上面的3条语句

而我认为的宽字节注入就是PHP发送请求到MySql时使用了语句
SET NAMES 'gbk' 或是 SET character_set_client =gbk 进行了一次编码，但是又由于一些不经意的字符集转换导致了宽字节注入

**3.11**
在我们正常情况下使用 addslashes函数或是开启PHP的GPC（注：在php5.4已上已给删除，并且需要说明特别说明一点，GPC无法过滤$_SERVER提交的参数）时过滤 GET、POST、COOKIE、REQUSET 提交的参数时，黑客们使用的预定义字符会给转义成添加反斜杠的字符串如下面的例子
例子：



```bash
单引号（'） = （\ '）
双引号（"） = （\ "）
反斜杠（\） = （\ \）
```

**3.12**
假如这个网站有宽字节注入那么我们提交：[http://127.0.0.1/unicodeSqlTest?id=%df%27](https://link.jianshu.com/?t=http://127.0.0.1/unicodeSqlTest?id=%df%27)
这时,假如我们现在使用的是addslashes来过滤,那么就会发生如下的转换过程
例子：



```php
%df%27===(addslashes)===>%df%5c%27===(数据库GBK)===>運'
```

这里可能有一些人没看懂，我可以粗略的解释一下。

前端输入**`%df%27`**时首先经过上面**`addslashes`**函数转义变成了**`%df%5c%27`**（%5c是反斜杠\），之后在数据库查询前因为设置了GBK编码，即是在汉字编码范围内两个字节都会给重新编码为一个汉字。然后MySQL服务器就会对查询语句进行GBK编码即是**`%df%5c`**转换成了汉字"運"，而单引号就逃逸了出来，从而造成了注入漏洞。

例如：

> http://www.xxx.com/login.php?user=%df’ or 1=1 limit 1,1%23&pass=

其对应的sql就是：

> select * fromcms_user where username = ‘運’ or 1=1 limit 1,1#’ and password=”

**注明：**

GBK编码，它的编码范围是`0×8140~0xFEFE`(不包括xx7F)，在遇到`%df`(ascii(223)) >ascii(128)时自动拼接`%5c`，因此吃掉‘\’，而%27、%20小于ascii(128)的字符就保留了。

补充：
GB2312是被GBK兼容的，它的高位范围是`0xA1~0xF7`，低位范围是`0xA1~0xFE`(`0x5C`不在该范围内)，因此不能使用编码吃掉`%5c`。
其它的宽字符集也是一样的分析过程，要吃掉`%5c`，只需要低位中包含正常的`0x5c`就行了。

**3.13**例子



```php
<?php
header("Content-Type:text/html;charset=gbk"); //为了显示，将页面默认为gbk
$name=$_GET['name'];
$name=addslashes($name);
$conn = mysqli_connect('127.0.0.1','root','');  
mysqli_select_db($conn,'mysql');
mysqli_query($conn,"SET NAMES 'gbk'");
if($conn==null){exit("connect error !<br>");}
$sql="select * from user where user='".$name."'";
$result=mysqli_query($conn,$sql);
echo $sql;
while($val=mysqli_fetch_row($result)){
    var_dump($val);
    print("<br>");
}
?>
```

这是一个带有宽字节注入的漏洞
这个PHP程序的SQL注入POC为：

```
http://127.0.0.1/test/t3.php?name=%df' or 1=1 %20%23`
对应的sql
`select * from a1 where name='運' or 1=1 #
```

![img](https://upload-images.jianshu.io/upload_images/9113981-56fe904b3450cef4.png?imageMogr2/auto-orient/strip|imageView2/2/w/810/format/webp)

图片.png

其原理是`mysql_query("SETNAMES 'gbk'",$conn)`语句将编码字符集修改为gbk，此时，`%df\'`对应的编码就是`%df%5c’`，即汉字“運’”，这样单引号之前的转义符号“\”就被吃调了，从而转义消毒失败。
像编码对应汉字的也有其他可以构造的，比如
`0xD50×5C` 对应了汉字“诚”，URL编码用百分号加字符的16进制编码表示字符，于是 `%d5%5c` 经URL解码后为“诚”。
**举一个注入详细的过程：**
例如：访问 `http://www.2cto.com /test.php?username=test%d5′%20or%201=1%23&pwd=test`

**经过[浏览器](https://www.2cto.com/os/liulanqi/)编码**，username参数值为(单引号的编码0×27)



```bash
username=test%d5%27%20or%201=1%23
```

**经过php的url解码**

`username=test 0xd5 0×27 0×20 or 0×20 1=1 0×23` (为了便于[阅读](http://book.2cto.com/)，在字符串与16进制编码之间加了空格)

**经过[PHP](https://www.2cto.com/kf/web/php/)的GPC自动转义变成(单引号0×27被转义成\’对应的编码0×5c0×27)：**



```bash
username=test 0xd5 0×5c 0×27 0×20 or 0×20 1=1 0×23
```

因为在[数据库](https://www.2cto.com/database/)初始化连接的时候SET NAMES ‘gbk’，`0xd50×5c`解码后为诚，`0×27`解码为’，`0×20`为空格，`0×23`为[mysql](https://www.2cto.com/database/MySQL/)的注释符#

**上面的SQL语句最终为：**

```
SELECT * FROM user WHERE username=’test诚’ or 1=1#’ and password=’test’;
```

注释符#后面的字符串已经无效，等价于



```bash
SELECT * FROM user WHERE username=’test诚’ or 1=1;
```

条件变成永真，成功注入。
**补充：**
0xD50×5C不是唯一可以绕过单引号转义的字符，0×81-0xFE开头+0×5C的字符应该都可以；
根据utf8的编码范围，无此问题；
**3.14**



```php
<?php  
header("Content-Type:text/html;charset=gbk"); //为了显示，将页面默认为gbk
error_reporting(0);  
$conn = mysql_connect('127.0.0.1','root','');  
mysql_select_db('mysql',$conn);  
mysql_query("set names gbk");  //不安全的编码设置方式  
$res = mysql_query("show variables like 'character%';"); //显示当前数据库设置的各项字符集  
while($row = mysql_fetch_array($res)){  
var_dump($row);  
}  
$user = addslashes($_GET['sql']); //mysql_real_escape_string() magic_quote_gpc=On addslashes() mysql_escape_string()功能类似  
$sql = "SELECT host,user FROM user WHERE user='{$user}'";  
echo $sql.'</br>';  
if($res = mysql_query($sql)){  
while($row = mysql_fetch_array($res)){  
var_dump($row);  
}  
}  
else{  
echo "Error".mysql_error()."<br/>";  
}  
?> 
```



![img](https://upload-images.jianshu.io/upload_images/9113981-cc5ee455778f5075.png?imageMogr2/auto-orient/strip|imageView2/2/w/760/format/webp)

图片.png


**解析：**





```rust
$_GET[‘sql’] 经过 addslashes编码之后带入了‘\’
1、%df%5C%27 or 1=1%23
2、带入mysql处理时使用了gbk字符集
%df%5c -> 運 成功的吃掉了%5c
%27 -> ‘ 单引号成功闭合
```

### 4. 宽字节注入漏洞深入补充

从宽字节注入漏洞原理可以看出，宽字节注入的关键点有两个：

（1） 设置宽字节字符集；

（2） 设置的宽字符集可能吃掉转义符号“\”（对应的编码为0x5c，即低位中包含正常的0x5c就行了）。

理论上，符合第二条的字符集都可能导致宽字节注入漏洞，这里以gbk字符集为典型，介绍宽字符注入漏洞典型案例。

宽字节注入漏洞的另一个关键是设置了character_set_client为宽字节字符集，这里有很多中设置的方式，主要包括隐式设置和显式设置。

隐含方式设置是指上面讲过的charcter_set_client缺省字符集就是宽字节字符集。

显式设置是指在PHP程序中调用相应的设置函数来实现字符集的设置或直接对字符串进行编码转换，设置的函数包括：



```bash
(1) mysql_query，如mysql_query("SET NAMES 'gbk'", $conn)、mysql_query("setcharacter_set_client = gbk", $conn)。

(2) mysql_set_charset，如mysql_set_charset("gbk",$conn)。

(3) mb_convert_encoding，如mb_convert_encoding($sql,"utf8","gbk")，将SQL语句从gbk格式转换为utf8格式时，0x5c被吃掉了。

(4) iconv，如iconv('GBK', 'UTF-8',$sql)，原理同上。
```

**4.1**
由上文可得宽字节注入是由于转编码而形成的，那具有转编码功能的函数也成了漏洞的成因。

**转码函数**

> mb_convert_encoding()
> iconv()

以下用iconv()来演示，修改上面的代码：



```php
<?php  
header("Content-Type:text/html;charset=gbk"); //为了显示，将页面默认为gbk 
error_reporting(0);  
$conn = mysql_connect('127.0.0.1','root','');  
mysql_select_db('mysql',$conn);  
mysql_set_charset("utf8"); //推荐的安全编码  
$user = mysql_real_escape_string(($_GET['sql'])); //推荐的过滤函数  
$user = iconv('GBK', 'UTF-8',$user);  
$sql = "SELECT host,userFROM user WHERE user='{$user}'";  
echo $sql.'</br>';  
$res = mysql_query($sql);  
while($row = mysql_fetch_array($res)){  
var_dump($row);  
}  
?> 
```

![img](https://upload-images.jianshu.io/upload_images/9113981-f59156b3b123c9d2.png?imageMogr2/auto-orient/strip|imageView2/2/w/716/format/webp)

图片.png

同样可以执行成功，编码解析的过程依然如上。

## 总结一下漏洞成因：

**代码3.14**

1、使用了不安全的字符集设置函数与过滤函数。

2、漏洞发生在PHP请求mysql时使用character_set_client值进行一次转码。

**代码4.1**
1、使用了推荐的设置函数与过滤函数。

2、解析错误发生在iconv()函数转码时，GBK转向UTF8吃掉了“\”

3、PHP请求mysql时转码安全。

另外：

当改变编码方向时`$user = iconv(‘UTF-8′, ’gbk’,$user);`

通过访问`http://localhost/xl.php?sql=root%e9%8c%a6`可以带入一个\，进而注释掉单引号。

这种情况下需要两个参数来配合注入。

例如：



```cpp
http://localhost/xl.php?sql=root%e9%8c%a6¶=%20or%201=1%23
```

#### 总结：

**1.**宽字节注入跟HTML页面编码无关。

**2** Mysql编码与过滤函数推荐使用`mysql_real_escape_string()`，`mysql_set_charset()`。

上文中代码使用了`mysql_query(“set names gbk”)`来设置编码，其实在mysql中是推荐`mysql_set_charset(“gbk”);`函数来进行编码设置的，这两个函数大致的功能相似，唯一不同之处是后者会修改mysql对象中的mysql->charset属性为设置的字符集。
同时配套的过滤函数为`mysql_real_escape_string()`。上面代码中列出了几个过滤的函数，他们之间的区别就是`mysql_real_escape_string()`会根据mysql对象中的mysql->charset属性来对待传入的字符串，因此可以根据当前字符集来进行过滤。

**3.**转编码函数同样会引起宽字节注入，即使使用了安全的设置函数。

参考：
http://netsecurity.51cto.com/art/201404/435074.htm
https://blog.csdn.net/helloweb2014/article/details/60757497
https://www.2cto.com/article/201209/153283.html