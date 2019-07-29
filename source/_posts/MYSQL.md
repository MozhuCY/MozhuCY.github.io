title: 网站开发入门-MYSQL数据库的基础使用
date: 1970-1-1
categories :
- WEB
---

# MYSQL #

直接进入主题
MYSQL数据库的结构大概就是 库-->表单-->行列等等,那么管理使用数据库,就需要一门结构化查询语言(Structured Query Language)简称SQL.
从0开始,若想使用数据库,首先要建立一个库,于是有了
>CREATE DATABASE test
 DROP DATABASE test

然后是新建表单.表(Table)由行集合构成,一行是列的序列(集合)，每列与行对应一个数据项.
>CREATE TABLE 表名称
(
    列名称1 数据类型,
    列名称2 数据类型,
    列名称3 数据类型,
    ....
)

sql里的数据类型,和一般语言(PHP C python)表示有些不同.详见下图.
![](/image/sql.jpg)
在PHP中,调用sql数据库的方法大概如下:
```php
<?php
$con = mysql_connect("localhost:3306","root","root");
mysql_select_db('wcy');
if (!$con)
  {
  die('Could not connect: ' . mysql_error());
  }
$a = @$_GET['a'];

$sql = "select flag from ctf1 where id='$a'";
echo 'your sql:'.$sql.'<br>';
$b = @mysql_fetch_array(mysql_query($sql));
echo $b[0];
?>
```
```html
<link rel>
```