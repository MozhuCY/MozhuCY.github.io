title: javascript 0x00
categories:
- FE
---
实例代码:
```html
<!DOCTYPE html>
<html>
<body>

<h1>JavaScript练习</h1>

<p id="d">
JavaScript
</p>

<script>
function myn()
{
x=document.getElementById("d");  // 找到元素
x.innerHTML="Hello JavaScript!";    // 改变内容
}
</script>

<button type="button" onclick="myn()">chance</button>

</body>
</html>
```
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <p >大黄的py</p>
    <p id="h">1</p>
    <p >mozhu的py</p>
    <p id="m">19</p>
    <script>
        function f(){
            var content = document.getElementById("h").innerHTML;
            document.getElementById("h").innerHTML=Number(content) + 1;
            var content = document.getElementById("m").innerHTML;
            document.getElementById("m").innerHTML=Number(content) - 1;
        }
    </script>
    <button type = "button" onclick="f()">button</button>
</body>
</html>
```
