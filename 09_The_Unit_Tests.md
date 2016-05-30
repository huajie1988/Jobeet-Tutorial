# Symfony2 Jobeet Day 8: 单元测试

## 在Symfony2中进行测试

在Symfony2中有两种不同的方式进行自动化测试：单元测试和功能测试。单元测试验证每个方法或函数是否能正常工作。每个测试必须尽可能的保持独立而不与其他方面有依赖。另一方面，功能测试用来验证一个应用程序整体上是否正确。

本篇主要介绍单元测试，而下一章则着重讲述功能测试。

Symfony2 中 集成了一个独立的测试库---**PHPUnit**, 为你提供了一个良好的测试框架。
为了运行测试你必须安装 PHPUnit 3.5.11 或更高版本。

任何一个测试---无论是单元测试还是功能测试---都在相应Bundle的Tests目录下有一个PHP类。
如果你遵循这一规则，那么你可以用如下命令对该应用程序的所有测试用例进行验证：

> **$ phpunit -c app/**

-c选项告诉PHPUnit使用app/目录下的配置文件。如果你对**PHPUnit**的选项感到好奇，可以查看 `app/phpunit.xml.dist` 文件。

## 单元测试

一个单元测试通常是针对一个特定的 PHP 类来进行测试的。让我们开始为 Jobeet::slugify() 方法编写一个测试用例。

在`src/Ens/JobeetBundle/Tests/Utils`目录下创建一个JobeetTest.php文件。依照惯例，Tests/下的子目录应该与你Bundle下的目录结构相同。所以当我们要测试你Bundle下的Utils目录下的一个类时，我们需要把测试类放到Tests/Utils/目录下。

```php
// src/Ens/JobeetBundle/Tests/Utils/JobeetTest.php
 
namespace Ens\JobeetBundle\Tests\Utils;
use Ens\JobeetBundle\Utils\Jobeet;
 
class JobeetTest extends \PHPUnit_Framework_TestCase
{
  public function testSlugify()
  {
    $this->assertEquals('sensio', Jobeet::slugify('Sensio'));
    $this->assertEquals('sensio-labs', Jobeet::slugify('sensio labs'));
    $this->assertEquals('sensio-labs', Jobeet::slugify('sensio   labs'));
    $this->assertEquals('paris-france', Jobeet::slugify('paris,france'));
    $this->assertEquals('sensio', Jobeet::slugify('  sensio'));
    $this->assertEquals('sensio', Jobeet::slugify('sensio  '));
  }
}
```

你可以使用如下命令来只运行这个测试：

>**phpunit -c app/ src/Ens/JobeetBundle/Tests/Utils/JobeetTest**

如果一切正常，你将会得到以下结果：

> PHPUnit 3.6.10 by Sebastian Bergmann.

> Configuration read from /home/dragos/work/jobeet/app/phpunit.xml.dist.

> Time: 0 seconds, Memory: 3.50Mb

> OK (1 test, 6 assertions)

## 基于测试添加新的功能

如果将一个空字符串传递给slugify()处理，它是能正常工作的，并且依旧返回一个空字符串。但在URL里面有一个空的字符串并不是一个好主意。那么如果当我们想要使得空字符串返回一个"n-a"该如何做呢？

首先我们要写一个测试用例，然后不断的修改，直到它通过测试。你或许有些不习惯这种编程方式，但它确实能第一时间使你对代码的正确性有充分的信心。

```php
// src/Ens/JobeetBundle/Tests/Utils/JobeetTest.php

// ...
 
$this->assertEquals('n-a', Jobeet::slugify(''));
 
// ...

```

现在我们再运行一遍测试会发现测试失败：
>PHPUnit 3.6.10 by Sebastian Bergmann.
 
>Configuration read from /home/dragos/work/jobeet/app/phpunit.xml.dist
 
>F
 
>Time: 0 seconds, Memory: 3.50Mb
 
>There was 1 failure:
 
>1) Ens\JobeetBundle\Tests\Utils\JobeetTest::testSlugify
>Failed asserting that two strings are equal.
>--- Expected
>+++ Actual
>@@ @@
>-'n-a'
>+''
 
>/home/dragos/work/jobeet/src/Ens/JobeetBundle/Tests/Utils/JobeetTest.php:17
 
>FAILURES!
>Tests: 1, Assertions: 7, Failures: 1.

现在, 让我们修改 `Jobeet` 类并在开始处添加一个新的判断:

```php
// src/Ens/JobeetBundle/Utils/Jobeet.php
// ...
 
static public function slugify($text)
{
  if (empty($text))
  {
    return 'n-a';
  }
 
// ...
}
```
现在测试通过了，是不是考虑去green bar喝一杯庆祝一下:-P
>译者注：green bar原文有双关的意思。

## 基于Bug添加一个测试

随着时间的流逝，可能某天你的一个用户向你报告了一个Bug，一些职位连接跳转到404页面。经过一番调查之后你发现，由于某种原因，这些职位的公司名称、 职位或工作地点是空的。

Why?

但当你查询数据库的记录时发现这些列并非是空的，何以如此？于是发动技能---容我三思。消耗掉三根棒棒糖之后你终于找到原因了---当一个字符串仅包含非 ASCII 字符时，slugify() 方法会将它转换为一个空字符串---于是你飞快的打开Jobeet类并计划立马解决这个问题。然而，请就此打住，这并不是一个好主意，首先，还是让我们先来添加一个测试用例：
```php
$this->assertEquals('n-a', Jobeet::slugify(' - '));
```

检测显而易见，你并不能通过测试。

现在我们修改Jobeet类，将空字符串判断放到方法末：
```php
static public function slugify($text)
{
  // ...
 
  if (empty($text))
  {
     return 'n-a';
  }
 
  return $text;
}
```

现在你已经通过了测试，而且slugify()测试覆盖率已经达到了100%，但它依旧有一个Bug，我们后面会说。

你可能想不到所有的边缘值并为其编写测试用例，这没关系。但你需要意识到应该在你修正你的代码之前先写测试。这将使你的代码随着时间的推移越来越好。

## 更好的slugify方法

你可能知道Symfony是一群法国人开发的，所以让我们来添加一个包含法语口音的测试。

```php
$this->assertEquals('developpeur-web', Jobeet::slugify('Développeur Web'));
```
显而易见的它会失败。我们期望是用`e`替换掉`é`，但实际情况是被替换成了`-`。
这是一个棘手的问题，不过庆幸的是我们安装了**iconv**库，它能帮助我们完成这项工作。将slugify方法中的代码替换如下：
```php
static public function slugify($text)
{
  // replace non letter or digits by -
  $text = preg_replace('#[^\\pL\d]+#u', '-', $text);
 
  // trim
  $text = trim($text, '-');
 
  // transliterate
  if (function_exists('iconv'))
  {
    $text = iconv('utf-8', 'us-ascii//TRANSLIT', $text);
  }
 
  // lowercase
  $text = strtolower($text);
 
  // remove unwanted characters
  $text = preg_replace('#[^-\w]+#', '', $text);
 
  if (empty($text))
  {
    return 'n-a';
  }
 
  return $text;
}
```
记得将你的PHP文件保存成UTF-8编码格式，这是Symfony的默认编码，并且使用iconv时也要使用UTF-8。

当然也要记得在你的测试用例中添加仅当**iconv**库存在时的判断：

```php
if (function_exists('iconv'))
{
  $this->assertEquals('developpeur-web', Jobeet::slugify('Développeur Web'));
}
```