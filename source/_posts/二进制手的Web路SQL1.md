title: 二进制手的Web路（sql注入篇
date: 1970-1-1
categories:
- WEB
---
**sql注入0x01：宽字节注入**</br>&ensp;&ensp;从宽字节注入开始记录吧，在网站的搭建时，为了安全起见，在PHP代码中常常会设置过滤，也正是因为PHP 的addslashes() 函数，防止了一部分sql注入，此函数会将用户输入的 “  ' ”，转义为“ \\' ”,从而防止了单引号闭合所导致的sql注入漏洞的发生。</br></br>
&ensp;&ensp;但是，随其出现的问题，便是接下来要说的了。因为GBK编码的原因，假如用户输入的命令为" %df' "那么很不幸的是，%df会和\结合起来形成一个汉字（其实只要汉字编码结尾为%5c（即为“\”）的都可以）。从而使得服务器端的assslashes()函数失效，达到单引号闭合的目的。</br>
```php
<?php
mysql_connect("localhost","root","");
mysql_select_db('sae');
	if(isset($_GET['id'])){
		$id  = addslashes($_GET['id']);
		$sql = "select id,title from news where id = '".$id."'" ;
		echo "your sql:".$sql."<br>";
        $query = mysql_query($sql);
        $array = mysql_fetch_array($query);
        if($query){
            echo $array['title'] ;
        }
	}else{
        echo "I need id";
    }
?>
```
&ensp;&ensp;同样在CGCTF的习题中存在以上代码。这题还要和union查询一同才可以起作用，先说一下关于GBK sql注入的吧。在我构造以下下sql语句时，返回页面如下</br>
>id=2%27or%20id=2#</br>
your sql:select id,title from news where id = '2\'or id=2'</br>gbk_sql_injection

&ensp;&ensp;但是当构造了如下sql注入语句时，居然产生了如下所示的报错
>id=%df%27or%20id=2# </br>your sql:select id,title from news where id = 'ß\'or id=2'</br></br>
>>Warning: mysql_fetch_array() expects parameter 1 to be resource, boolean given in SQL-GBK/index.php on line 10

&ensp;&ensp;这就说明，注入点存在，这时就可以进行下一步操作了（union联合查找//貌似是这样叫...）不过还不是很明白，为什么打印到屏幕上的$id不是吃掉/的汉字呢，难道是网页编码问题  \_(:3」∠)_ 下次再探究吧，去学一波union联合查找先。

&ensp;&ensp;30分钟后:我又回来了，果然是网页编码的问题，我的chrome浏览器网页编码不是GBK。但是换成了360浏览器就可以清楚的看到 \ 被吃了,谜题解开了，继续学习union联合查询。