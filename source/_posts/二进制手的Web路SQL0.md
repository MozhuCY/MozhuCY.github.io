title: 二进制手的Web路（sql注入篇
date: 1970-1-1
categories:
- WEB
---
## **想入门一下Web，非常的那种。**

**那这几天系统的学一下SQL注入吧**

&ensp;&ensp;那么什么是SQL注入呢。SQL是数据库的一种，一般的SQL查询语句大概就是SELECT * FROM * WHERE * ,类似这种语句一般都会在PHP代码中构造好，预留出一个空，在取得客户端的用户输入的数值，将其填入，这时，如果不进行过滤的防护措施，不法用户所输入的恶意代码也可能被执行，进而达到控制数据库的效果，先举一个例子吧。
```html
<?php
if($_POST[user] && $_POST[pass]) {
    mysql_connect(SAE_MYSQL_HOST_M . ':' . SAE_MYSQL_PORT,SAE_MYSQL_USER,SAE_MYSQL_PASS);
  mysql_select_db(SAE_MYSQL_DB);
  $user = trim($_POST[user]);
  $pass = md5(trim($_POST[pass]));
  $sql="select user from ctf where (user='".$user."') and (pw='".$pass."')";
    echo '</br>'.$sql;
  $query = mysql_fetch_array(mysql_query($sql));
  if($query[user]=="admin") {
      echo "<p>Logged in! flag:******************** </p>";
  }
  if($query[user] != "admin") {
    echo("<p>You are not admin!</p>");
  }
}
echo $query[user];
?>
```
&ensp;&ensp;这是CGCTF上的一道题，只要以管理员身份登陆，就可以拿到flag。但是我们并不知道管理员账号admin的密码，这时我们就可以构造sql注入语句绕过对密码的判断。

_select user from ctf where (user='".$user."') and (pw='".$pass."')_

这句话便是关键了,构造sql语句为"admin')#"
```PHP
select user from ctf where (user='admin')#') and (pw='".$pass."')
```
&ensp;&ensp;从而达到我们的目的。（代码效果如上单引号闭合，直接“#”直接注释掉多余内容。</br>
&ensp;&ensp;大概了解了SQL注入的原理了，下面来细数一下SQL注入的几个类型

+ Boolean-based blind SQL injection（布尔型注入）
+ Error-based SQL injection（报错型注入）
+ UNION query SQL injection（可联合查询注入）
+ Stacked queries SQL injection（可多语句查询注入）
+ Time-based blind SQL injection（基于时间延迟注入）</br>

接下来的几篇博文，会分别整理出这5种SQL注入的方式，欢迎关注。

