# Symfony2 Jobeet Day 8: 单元测试

## 在Symfony2中进行测试

在Symfony2中有两种不同的方式进行自动化测试：单元测试和功能测试。单元测试验证每个方法或函数是否能正常工作。每个测试必须尽可能的保持独立而不与其他方面有依赖。另一方面，功能测试用来验证一个应用程序整体上是否正确。

本篇主要介绍单元测试，而下一章则着重讲述功能测试。

Symfony2 中 集成了一个独立的测试库---**PHPUnit**, 为你提供了一个良好的测试框架。
为了运行测试你必须安装 PHPUnit 3.5.11 或更高版本。

如果你没有安装PHPUnit，那么你可以通过以下方式来安装它们：

> **sudo apt-get install phpunit**  
> **sudo pear channel-discover pear.phpunit.de**  
> **sudo pear channel-discover pear.symfony-project.com**  
> **sudo pear channel-discover components.ez.no**  
> **sudo pear channel-discover pear.symfony.com**  
> **sudo pear update-channels**  
> **sudo pear upgrade-all**  
> **sudo pear install pear.symfony.com/Yaml**  
> **sudo pear install --alldeps phpunit/PHPUnit**  
> **sudo pear install --force --alldeps phpunit/PHPUnit**  

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

你可以去查看[PHPUnit的官方文档](http://www.phpunit.de/manual/current/en/writing-tests-for-phpunit.html#writing-tests-for-phpunit.assertions)，获取完整的断言列表。

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
>   
>Configuration read from /home/dragos/work/jobeet/app/phpunit.xml.dist  
>   
>F  
>   
>Time: 0 seconds, Memory: 3.50Mb  
>   
>There was 1 failure:  
>    
>1) Ens\JobeetBundle\Tests\Utils\JobeetTest::testSlugify  
>Failed asserting that two strings are equal.  
>--- Expected  
>+++ Actual  
>@@ @@  
>-'n-a'  
>+''  
>   
>/home/dragos/work/jobeet/src/Ens/JobeetBundle/Tests/Utils/JobeetTest.php:17  
>    
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
// src/Ens/JobeetBundle/Utils/Jobeet.php
// ...

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
// src/Ens/JobeetBundle/Utils/Jobeet.php
// ...

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

## 代码覆盖率

当我们编写测试案例时，很容易忘记部分代码，从而没写测试的话显然会很糟糕。为此如果你想要添加一个新的功能或想查看是否有遗漏时，代码覆盖率显得尤为重要。我们可以使用--coverage-html选项来检查代码覆盖率。

> **phpunit --coverage-html=web/cov/ -c app/**

运行之后我们可以在浏览器中打开http://jobeet.local/cov/index.html页面来查看效果。

> *只有你安装了XDebug及其相关依赖并启用后这个检查才能生效。安装方式如下：*  

> **sudo apt-get install php5-xdebug**

在`cov/index.html`页面中你将看到如下图所示：

![day9_code_coverage_1](./image/day9_code_coverage_1.jpg)

请记住当你的单元测试全部通过的时候仅仅只意味着每一行都被执行了，但并非说明每一个边缘值都做过测试。

## Doctrine单元测试

对一个Doctrine Model类的单元测试稍微有些复杂，因为它需要连接一个数据库。虽然你已经有了一个开发数据库，但更好的习惯是为测试建立一个专用的数据库。

在开始本次教程之前我们应该已经介绍过可以改变应用程序的设置来切换不同的环境。

默认情况下，Symfony的所有测试都是在测试环境中运行的，所以我们可以配置一个不同的数据库，专门用于测试环境 ︰

打开你的`app/config`目录，拷贝并粘贴一个parameters.yml文件，同时重命名为parameters_test.yml。打开并修改parameters_test.yml文件，将所需连接的数据库信息修改为jobeet_test的信息(假设你的测试数据库为jobeet_test)最后打开config_test.yml文件导入测试配置：

```yaml
#app/config/config_test.yml
imports:
    - { resource: config_dev.yml }
    - { resource: parameters_test.yml }
#...
```
## 测试Job Entity

首先我们需要在Tests/Entity文件夹下创建一个JobTest.php文件。

每次当你运行测试时setUp函数都会对数据库进行相应的操作。首先它会删除你当前的数据库，然后重建并从fixtures中重新加载数据。

这将帮助你在每次运行测试之前测试环境的数据库都具有相同的初始数据。

> *译者注：其中setUp的具体功能及使用可参考这篇《[Setup Testing and Fixtures in Symfony2: The Easy Way](http://intelligentbee.com/blog/2016/03/11/setup-testing-and-fixtures-in-symfony2-the-easy-way/)》*

```php
// src/Ens/JobeetBundle/Tests/Entity/JobTest.php

namespace Ens\JobeetBundle\Entity;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Ens\JobeetBundle\Utils\Jobeet as Jobeet;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;

class JobTest extends WebTestCase
{
    private $em;
    private $application;

    public function setUp()
    {
        static::$kernel = static::createKernel();
        static::$kernel->boot();

        $this->application = new Application(static::$kernel);

        // drop the database
        $command = new DropDatabaseDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:database:drop',
            '--force' => true
        ));
        $command->run($input, new NullOutput());

        // we have to close the connection after dropping the database so we don't get "No database selected" error
        $connection = $this->application->getKernel()->getContainer()->get('doctrine')->getConnection();
        if ($connection->isConnected()) {
            $connection->close();
        }

        // create the database
        $command = new CreateDatabaseDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:database:create',
        ));
        $command->run($input, new NullOutput());

        // create schema
        $command = new CreateSchemaDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:schema:create',
        ));
        $command->run($input, new NullOutput());

        // get the Entity Manager
        $this->em = static::$kernel->getContainer()
            ->get('doctrine')
            ->getManager();

        // load fixtures
        $client = static::createClient();
        $loader = new \Symfony\Bridge\Doctrine\DataFixtures\ContainerAwareLoader($client->getContainer());
        $loader->loadFromDirectory(static::$kernel->locateResource('@EnsJobeetBundle/DataFixtures/ORM'));
        $purger = new \Doctrine\Common\DataFixtures\Purger\ORMPurger($this->em);
        $executor = new \Doctrine\Common\DataFixtures\Executor\ORMExecutor($this->em, $purger);
        $executor->execute($loader->getFixtures());
    }

    public function testGetCompanySlug()
    {
        $job = $this->em->createQuery('SELECT j FROM EnsJobeetBundle:Job j ')
            ->setMaxResults(1)
            ->getSingleResult();

        $this->assertEquals($job->getCompanySlug(), Jobeet::slugify($job->getCompany()));
    }

    public function testGetPositionSlug()
    {
        $job = $this->em->createQuery('SELECT j FROM EnsJobeetBundle:Job j ')
            ->setMaxResults(1)
            ->getSingleResult();

        $this->assertEquals($job->getPositionSlug(), Jobeet::slugify($job->getPosition()));
    }

    public function testGetLocationSlug()
    {
        $job = $this->em->createQuery('SELECT j FROM EnsJobeetBundle:Job j ')
            ->setMaxResults(1)
            ->getSingleResult();

        $this->assertEquals($job->getLocationSlug(), Jobeet::slugify($job->getLocation()));
    }

    public function testSetExpiresAtValue()
    {
        $job = new Job();
        $job->setExpiresAtValue();

        $this->assertEquals(time() + 86400 * 30, $job->getExpiresAt()->format('U'));
    }

    protected function tearDown()
    {
        parent::tearDown();
        $this->em->close();
    }
}

```

## 测试Repository

现在让我们为JobRepository 类写一些测试，来看看我们在前一天创建的函数是否能返回正确的值︰

```php
// src/Ens/JobeetBundle/Tests/Repository/JobRepositoryTest.php

namespace Ens\JobeetBundle\Tests\Repository;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;

class JobRepositoryTest extends WebTestCase
{
    private $em;
    private $application;

    public function setUp()
    {
        static::$kernel = static::createKernel();
        static::$kernel->boot();

        $this->application = new Application(static::$kernel);

        // drop the database
        $command = new DropDatabaseDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:database:drop',
            '--force' => true
        ));
        $command->run($input, new NullOutput());

        // we have to close the connection after dropping the database so we don't get "No database selected" error
        $connection = $this->application->getKernel()->getContainer()->get('doctrine')->getConnection();
        if ($connection->isConnected()) {
            $connection->close();
        }

        // create the database
        $command = new CreateDatabaseDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:database:create',
        ));
        $command->run($input, new NullOutput());

        // create schema
        $command = new CreateSchemaDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:schema:create',
        ));
        $command->run($input, new NullOutput());

        // get the Entity Manager
        $this->em = static::$kernel->getContainer()
            ->get('doctrine')
            ->getManager();

        // load fixtures
        $client = static::createClient();
        $loader = new \Symfony\Bridge\Doctrine\DataFixtures\ContainerAwareLoader($client->getContainer());
        $loader->loadFromDirectory(static::$kernel->locateResource('@EnsJobeetBundle/DataFixtures/ORM'));
        $purger = new \Doctrine\Common\DataFixtures\Purger\ORMPurger($this->em);
        $executor = new \Doctrine\Common\DataFixtures\Executor\ORMExecutor($this->em, $purger);
        $executor->execute($loader->getFixtures());
    }

    public function testCountActiveJobs()
    {
        $query = $this->em->createQuery('SELECT c FROM EnsJobeetBundle:Category c');
        $categories = $query->getResult();

        foreach($categories as $category) {
            $query = $this->em->createQuery('SELECT COUNT(j.id) FROM EnsJobeetBundle:Job j WHERE j.category = :category AND j.expires_at > :date');
            $query->setParameter('category', $category->getId());
            $query->setParameter('date', date('Y-m-d H:i:s', time()));
            $jobs_db = $query->getSingleScalarResult();

            $jobs_rep = $this->em->getRepository('EnsJobeetBundle:Job')->countActiveJobs($category->getId());
            // This test will verify if the value returned by the countActiveJobs() function
            // coincides with the number of active jobs for a given category from the database
            $this->assertEquals($jobs_rep, $jobs_db);
        }
    }

    public function testGetActiveJobs()
    {
        $query = $this->em->createQuery('SELECT c from EnsJobeetBundle:Category c');
        $categories = $query->getResult();

        foreach ($categories as $category) {
            $query = $this->em->createQuery('SELECT COUNT(j.id) from EnsJobeetBundle:Job j WHERE j.expires_at > :date AND j.category = :category');
            $query->setParameter('date', date('Y-m-d H:i:s', time()));
            $query->setParameter('category', $category->getId());
            $jobs_db = $query->getSingleScalarResult();

            $jobs_rep = $this->em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), null, null);
            // This test tells if the number of active jobs for a given category from
            // the database is the same as the value returned by the function
            $this->assertEquals($jobs_db, count($jobs_rep));
        }
    }

    public function testGetActiveJob()
    {
        $query = $this->em->createQuery('SELECT j FROM EnsJobeetBundle:Job j WHERE j.expires_at > :date');
        $query->setParameter('date', date('Y-m-d H:i:s', time()));
        $query->setMaxResults(1);
        $job_db = $query->getSingleResult();

        $job_rep = $this->em->getRepository('EnsJobeetBundle:Job')->getActiveJob($job_db->getId());
        // If the job is active, the getActiveJob() method should return a non-null value
        $this->assertNotNull($job_rep);

        $query = $this->em->createQuery('SELECT j FROM EnsJobeetBundle:Job j WHERE j.expires_at < :date');         $query->setParameter('date', date('Y-m-d H:i:s', time()));
        $query->setMaxResults(1);
        $job_expired = $query->getSingleResult();

        $job_rep = $this->em->getRepository('EnsJobeetBundle:Job')->getActiveJob($job_expired->getId());
        // If the job is expired, the getActiveJob() method should return a null value
        $this->assertNull($job_rep);
    }

    protected function tearDown()
    {
        parent::tearDown();
        $this->em->close();
    }
}
```

同样的别忘了CategoryRepository：

```php
// src/Ens/JobeetBundle/Tests/Repository/CategoryRepositoryTest.php

namespace Ens\JobeetBundle\Tests\Repository;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;

class CategoryRepositoryTest extends WebTestCase
{
    private $em;
    private $application;

    public function setUp()
    {
        static::$kernel = static::createKernel();
        static::$kernel->boot();

        $this->application = new Application(static::$kernel);

        // drop the database
        $command = new DropDatabaseDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:database:drop',
            '--force' => true
        ));
        $command->run($input, new NullOutput());

        // we have to close the connection after dropping the database so we don't get "No database selected" error
        $connection = $this->application->getKernel()->getContainer()->get('doctrine')->getConnection();
        if ($connection->isConnected()) {
            $connection->close();
        }

        // create the database
        $command = new CreateDatabaseDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:database:create',
        ));
        $command->run($input, new NullOutput());

        // create schema
        $command = new CreateSchemaDoctrineCommand();
        $this->application->add($command);
        $input = new ArrayInput(array(
            'command' => 'doctrine:schema:create',
        ));
        $command->run($input, new NullOutput());

        // get the Entity Manager
        $this->em = static::$kernel->getContainer()
            ->get('doctrine')
            ->getManager();

        // load fixtures
        $client = static::createClient();
        $loader = new \Symfony\Bridge\Doctrine\DataFixtures\ContainerAwareLoader($client->getContainer());
        $loader->loadFromDirectory(static::$kernel->locateResource('@EnsJobeetBundle/DataFixtures/ORM'));
        $purger = new \Doctrine\Common\DataFixtures\Purger\ORMPurger($this->em);
        $executor = new \Doctrine\Common\DataFixtures\Executor\ORMExecutor($this->em, $purger);
        $executor->execute($loader->getFixtures());
    }

    public function testGetWithJobs()
    {
        $query = $this->em->createQuery('SELECT c FROM EnsJobeetBundle:Category c LEFT JOIN c.jobs j WHERE j.expires_at > :date');
        $query->setParameter('date', date('Y-m-d H:i:s', time()));
        $categories_db = $query->getResult();

        $categories_rep = $this->em->getRepository('EnsJobeetBundle:Category')->getWithJobs();
        // This test verifies if the number of categories having active jobs, returned
        // by the getWithJobs() function equals the number of categories having active jobs from database
        $this->assertEquals(count($categories_rep), count($categories_db));
    }

    protected function tearDown()
    {
        parent::tearDown();
        $this->em->close();
    }
}
```

当我们写完测试后，用下面的命令，以便得到代码覆盖率：

> **phpunit --coverage-html=web/cov/ -c app src/Ens/JobeetBundle/Tests/Repository/**

现在你如果访问http://jobeet.local/cov/Repository.html会发现现在的代码覆盖率并非100%

![day9_testing_the_repository_classes_1](./image/day9_testing_the_repository_classes_1.jpg)

让我们来为JobRepository类添加一些测试使代码覆盖率能达到100%。目前在我们的数据库中两个分类没有激活的职位信息，而另一个只有一条已激活的职位信息。

但我们需要对这只有三行的数据进行$max和$offset参数的测试，为此我们用下面的代码替换testGetActiveJobs()函数中的foreach语句：

```php
// src/Ens/JobeetBundle/Tests/Repository/JobRepositoryTest.php

// ...
foreach ($categories as $category) {
    // ...

    // If there are at least 3 active jobs in the selected category, we will
    // test the getActiveJobs() method using the limit and offset parameters too
    // to get 100% code coverage
    if($jobs_db > 2 ) {
        $jobs_rep = $this->em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), 2);
        // This test tells if the number of returned active jobs is the one $max parameter requires
        $this->assertEquals(2, count($jobs_rep));

        $jobs_rep = $this->em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), 2, 1);
        // We set the limit to 2 results, starting from the second job and test if the result is as expected
        $this->assertEquals(2, count($jobs_rep));
    }
}
// ...
```

再次运行代码覆盖率的检测命令：

> **phpunit --coverage-html=web/cov/ -c app src/Ibw/JobeetBundle/Tests/Repository/**

现在我们检查代码覆盖率，会发现已经到达了100%了。

![day9_testing_the_repository_classes_2.jpg](./image/day9_testing_the_repository_classes_2.jpg)

这就是今天的全部内容了，明天我们将讲功能测试。