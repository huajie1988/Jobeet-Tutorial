# 功能测试

功能测试是一个优秀的点对点测试工具：点对点是指从浏览器发送请求到接收服务器的响应的过程。它测试所有东西：路由、 Model、 action和模板。它与你已经手动做的过程非常相似：每次当你添加或修改一个action，你就需要打开浏览器检查所有的一切是否如你预期那般正常工作，包括点击链接或检查页面上所呈现的元素等操作。换句话说，你现在所运行的就是未来实际应用的场景。

每次手动完成这个过程不仅繁琐而且易错的。当你每次改变你的代码之后不得不再次重复这个过程以确保不会因此而造成新的问题。It's crazy!幸运的是Symfony的单元测试工具提供了一种更简单的方式来帮助我们完成这件事。Symfony可以产生不同的场景进而一遍遍的模拟真实用户在浏览器中的操作。与单元测试一样，它能使你对自己的代码更有信心。

功能测试的工作流程非常具体︰

* 发出一个请求
* 测试响应结果
* 点击一个链接或者提交一个表单
* 测试响应结果
* 重复上述步骤

##第一个功能测试用例

功能测试是一个简单的PHP文件，它通常位于你Bundle下的Tests/Controller下。比如你想测试CategoryController，那么可以在其下新建一个CategoryControllerTest.php，并让它继承WebTestCase类：
```php
// src/Ens/JobeetBundle/Teste/Controller/CategoryControllerTest.php
namespace Ens\JobeetBundle\Tests\Controller;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
 
class CategoryControllerTest extends WebTestCase
{
  public function testShow()
  {
    $client = static::createClient();
 
    $crawler = $client->request('GET', '/category/index');
    $this->assertEquals('Ens\JobeetBundle\Controller\CategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
    $this->assertTrue(200 === $client->getResponse()->getStatusCode());
  }
}
```

## 运行测试

与单元测试一样，我们可以使用**phpunit**来运行测试：

> **phpunit -c app/ src/Ens/JobeetBundle/Tests/Controller/CategoryControllerTest**

这个测试必然的会失败，因为在现有的路由中/category/index并不存在：

>PHPUnit 3.6.10 by Sebastian Bergmann.
 
>Configuration read from /home/dragos/work/jobeet/app/phpunit.xml.dist
 
>F
 
>Time: 2 seconds, Memory: 14.25Mb
 
>There was 1 failure:
 
>1) Ens\JobeetBundle\Tests\Controller\CategoryControllerTest::testShow  
Failed asserting that false is true.

## 编写功能测试
功能测试就像你在浏览器中运行一样。我们现在需要把所有[第二天](03_The_Project.md)所描述的场景全部写出来进行测试。

首先让我们为测试主页修改 JobControllerTest.php 文件。并对这些功能逐一测试：

### 过期职位不显示
```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
namespace Ens\JobeetBundle\Tests\Controller;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
 
class JobControllerTest extends WebTestCase
{
  public function testIndex()
  {
    $client = static::createClient();
 
    $crawler = $client->request('GET', '/');
    $this->assertEquals('Ens\JobeetBundle\Controller\JobController::indexAction', $client->getRequest()->attributes->get('_controller'));
    $this->assertTrue($crawler->filter('.jobs td.position:contains("Expired")')->count() == 0);
  }
}
```

要验证首页是否没有过期的职位，我们可以使用CSS选择器 `.jobs td.position:contains("Expired")` 如果在响应的HTML中没有任何匹配就说明测试通过(你或许还记得，我们用fixtures填充数据时仅在过期的工作的position字段里加入了包含"**Expired**"关键字的信息)

###仅返回某分类下前N个职位

在测试文件的末尾添加以下代码。我们这次需要使用内核函数使它能正确读取我们在`app/config/config.yml`中定义的参数：

```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
// ...
 
$kernel = static::createKernel();
$kernel->boot();
$max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');
$this->assertTrue($crawler->filter('.category_programming tr')->count() <= $max_jobs_on_homepage );

```

在这次测试中我们需要添加相应的类到Job/index.html.twig中(这样我们才能在职位列表中选择每个分类并统计出数量)

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->
<!-- ... -->
 
    {% for category in categories %}</pre>
<div class="category_{{ category.slug }}"><!-- ... -->
```

###当职位过多时显示**更多**按钮

```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
// ...
 
$this->assertTrue($crawler->filter('.category_design .more_jobs')->count() == 0);
$this->assertTrue($crawler->filter('.category_programming .more_jobs')->count() == 1);
// ...
```

在这个测试中我们检查是否存在`more jobs`链接在design分类中(.category_design .more_jobs 理应不存在)，并测试在programming分类中是否存在`more jobs`链接(.category_programming .more_jobs 理应是存在的)

###测试职位是否按日期排序

若要测试职位实际上是按日期排序的，那么我们需要检查主页列表上的第一份职位是否符合我们的预期。这可以通过检查该 URL 是否包含预期的主键来验证。这个主键能在运行时被改变，因此我们首先需要从数据库中获取一个Doctrine object。

```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
// ...
 
$kernel = static::createKernel();
$kernel->boot();
$em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
$query = $em->createQuery('SELECT j from EnsJobeetBundle:Job j LEFT JOIN j.category c WHERE c.slug = :slug AND j.expires_at > :date ORDER BY j.created_at DESC');
$query->setParameter('slug', 'programming');
$query->setParameter('date', date('Y-m-d H:i:s', time()));
$query->setMaxResults(1);
$job = $query->getSingleResult();
 
$this->assertTrue($crawler->filter('.category_programming tr')->first()->filter(sprintf('a[href*="/%d/"]', $job->getId()))->count() == 1);
// ...
```

尽管这个测试现在已经能工作了，但我们还是需要重构一下代码。获得第一条programming分类下职位的代码我们将会在多个地方使用。但这并不意味着我们需要将其移动到Model层，相反，我们会在测试类里添加 *getMostRecentProgrammingJob* 函数，并把代码移入其中：

```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
// ...
 
public function getMostRecentProgrammingJob()
{
  $kernel = static::createKernel();
  $kernel->boot();
  $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
  $query = $em->createQuery('SELECT j from EnsJobeetBundle:Job j LEFT JOIN j.category c WHERE c.slug = :slug AND j.expires_at > :date ORDER BY j.created_at DESC');
  $query->setParameter('slug', 'programming');
  $query->setParameter('date', date('Y-m-d H:i:s', time()));
  $query->setMaxResults(1);
  return $query->getSingleResult();
}
 
// ...
```
现在再将之前的代码替换如下：
```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
// ...
 
$this->assertTrue($crawler->filter('.category_programming tr')->first()->filter(sprintf('a[href*="/%d/"]', $this->getMostRecentProgrammingJob()->getId()))->count() == 1);
// ...
```

###测试首页的每个职位链接都可以点击

若要测试首页上的职位链接，我们需要模拟一个"Web Developer"文本上的点击操作。在页面上有很多链接，但我们需要明确的告知浏览器点击的是第一个。

然后测试每个请求参数，以确保其点击链接后路由已正确完成其工作。

```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
// ...
 
$job = $this->getMostRecentProgrammingJob();
$link = $crawler->selectLink('Web Developer')->first()->link();
$crawler = $client->click($link);
$this->assertEquals('Ens\JobeetBundle\Controller\JobController::showAction', $client->getRequest()->attributes->get('_controller'));
$this->assertEquals($job->getCompanySlug(), $client->getRequest()->attributes->get('company'));
$this->assertEquals($job->getLocationSlug(), $client->getRequest()->attributes->get('location'));
$this->assertEquals($job->getPositionSlug(), $client->getRequest()->attributes->get('position'));
$this->assertEquals($job->getId(), $client->getRequest()->attributes->get('id'));
// ...
```

## 实例学习

在本节中，你已经有了所有关于职位页和分类页的测试代码。请仔细阅读这些代码，因为你可能会从中学到一些新的技巧：

```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php
 
namespace Ens\JobeetBundle\Tests\Controller;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
 
class JobControllerTest extends WebTestCase
{
  public function getMostRecentProgrammingJob()
  {
    $kernel = static::createKernel();
    $kernel->boot();
    $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
    $query = $em->createQuery('SELECT j from EnsJobeetBundle:Job j LEFT JOIN j.category c WHERE c.slug = :slug AND j.expires_at > :date ORDER BY j.created_at DESC');
    $query->setParameter('slug', 'programming');
    $query->setParameter('date', date('Y-m-d H:i:s', time()));
    $query->setMaxResults(1);
    return $query->getSingleResult();
  }
 
  public function getExpiredJob()
  {
    $kernel = static::createKernel();
    $kernel->boot();
    $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
    $query = $em->createQuery('SELECT j from EnsJobeetBundle:Job j WHERE j.expires_at < :date');
    $query->setParameter('date', date('Y-m-d H:i:s', time()));
    $query->setMaxResults(1);
    return $query->getSingleResult();
  }
 
  public function testIndex()
  {
    // get the custom parameters from app config.yml
    $kernel = static::createKernel();
    $kernel->boot();
    $max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');
    $max_jobs_on_category = $kernel->getContainer()->getParameter('max_jobs_on_category');
 
    $client = static::createClient();
 
    $crawler = $client->request('GET', '/');
    $this->assertEquals('Ens\JobeetBundle\Controller\JobController::indexAction', $client->getRequest()->attributes->get('_controller'));
 
    // expired jobs are not listed
    $this->assertTrue($crawler->filter('.jobs td.position:contains("Expired")')->count() == 0);
 
    // only $max_jobs_on_homepage jobs are listed for a category
    $this->assertTrue($crawler->filter('.category_programming tr')->count()<= $max_jobs_on_homepage); 
    $this->assertTrue($crawler->filter('.category_design .more_jobs')->count() == 0);
    $this->assertTrue($crawler->filter('.category_programming .more_jobs')->count() == 1);
 
    // jobs are sorted by date
    $this->assertTrue($crawler->filter('.category_programming tr')->first()->filter(sprintf('a[href*="/%d/"]', $this->getMostRecentProgrammingJob()->getId()))->count() == 1);
 
    // each job on the homepage is clickable and give detailed information
    $job = $this->getMostRecentProgrammingJob();
    $link = $crawler->selectLink('Web Developer')->first()->link();
    $crawler = $client->click($link);
    $this->assertEquals('Ens\JobeetBundle\Controller\JobController::showAction', $client->getRequest()->attributes->get('_controller'));
    $this->assertEquals($job->getCompanySlug(), $client->getRequest()->attributes->get('company'));
    $this->assertEquals($job->getLocationSlug(), $client->getRequest()->attributes->get('location'));
    $this->assertEquals($job->getPositionSlug(), $client->getRequest()->attributes->get('position'));
    $this->assertEquals($job->getId(), $client->getRequest()->attributes->get('id'));
 
    // a non-existent job forwards the user to a 404
    $crawler = $client->request('GET', '/job/foo-inc/milano-italy/0/painter');
    $this->assertTrue(404 === $client->getResponse()->getStatusCode());
 
    // an expired job page forwards the user to a 404
    $crawler = $client->request('GET', sprintf('/job/sensio-labs/paris-france/%d/web-developer', $this->getExpiredJob()->getId()));
    $this->assertTrue(404 === $client->getResponse()->getStatusCode());
  }
}
``` 



```php
// src/Ens/JobeetBundle/Test/Controller/CategoryControllerTest.php
 
namespace Ens\JobeetBundle\Tests\Controller;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
 
class CategoryControllerTest extends WebTestCase
{
  public function testShow()
  {
    // get the custom parameters from app config.yml
    $kernel = static::createKernel();
    $kernel->boot();
    $max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');
    $max_jobs_on_category = $kernel->getContainer()->getParameter('max_jobs_on_category');
 
    $client = static::createClient();
 
    // categories on homepage are clickable
    $crawler = $client->request('GET', '/');
    $link = $crawler->selectLink('Programming')->link();
    $crawler = $client->click($link);
    $this->assertEquals('Ens\JobeetBundle\Controller\CategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
    $this->assertEquals('programming', $client->getRequest()->attributes->get('slug'));
 
    // categories with more than $max_jobs_on_homepage jobs also have a "more" link
    $crawler = $client->request('GET', '/');
    $link = $crawler->selectLink('22')->link();
    $crawler = $client->click($link);
    $this->assertEquals('Ens\JobeetBundle\Controller\CategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
    $this->assertEquals('programming', $client->getRequest()->attributes->get('slug'));
 
    // only $max_jobs_on_category jobs are listed
    $this->assertTrue($crawler->filter('.jobs tr')->count() assertRegExp('/32 jobs/', $crawler->filter('.pagination_desc')->text());
    $this->assertRegExp('/page 1\/2/', $crawler->filter('.pagination_desc')->text());
 
    $link = $crawler->selectLink('2')->link();
    $crawler = $client->click($link);
    $this->assertEquals(2, $client->getRequest()->attributes->get('page'));
    $this->assertRegExp('/page 2\/2/', $crawler->filter('.pagination_desc')->text());
  }
}
```

## 另一个实例

本部分为补充部分，参考[实例学习](http://intelligentbee.com/blog/2013/08/15/symfony2-jobeet-day-9-the-functional-tests/)部分内容，提供了另一种更完善的测试方式，并且也更新一点，可以综合起来看。

其中setUp的功能及使用可参考这篇《[Setup Testing and Fixtures in Symfony2: The Easy Way](http://intelligentbee.com/blog/2016/03/11/setup-testing-and-fixtures-in-symfony2-the-easy-way/)》


```php
// src/Ens/JobeetBundle/Test/Controller/JobControllerTest.php

namespace IbwJobeetBundleTestsController;
 
use SymfonyBundleFrameworkBundleTestWebTestCase;
use SymfonyBundleFrameworkBundleConsoleApplication;
use SymfonyComponentConsoleOutputNullOutput;
use SymfonyComponentConsoleInputArrayInput;
use DoctrineBundleDoctrineBundleCommandDropDatabaseDoctrineCommand;
use DoctrineBundleDoctrineBundleCommandCreateDatabaseDoctrineCommand;
use DoctrineBundleDoctrineBundleCommandProxyCreateSchemaDoctrineCommand;
 
class JobControllerTest extends WebTestCase
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
        $loader = new SymfonyBridgeDoctrineDataFixturesContainerAwareLoader($client->getContainer());
        $loader->loadFromDirectory(static::$kernel->locateResource('@IbwJobeetBundle/DataFixtures/ORM'));
        $purger = new DoctrineCommonDataFixturesPurgerORMPurger($this->em);
        $executor = new DoctrineCommonDataFixturesExecutorORMExecutor($this->em, $purger);
        $executor->execute($loader->getFixtures());
    }
 
    public function getMostRecentProgrammingJob()
    {
        $kernel = static::createKernel();
        $kernel->boot();
        $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
        $query = $em->createQuery('SELECT j from IbwJobeetBundle:Job j LEFT JOIN j.category c WHERE c.slug = :slug AND j.expires_at > :date ORDER BY j.created_at DESC');
        $query->setParameter('slug', 'programming');
        $query->setParameter('date', date('Y-m-d H:i:s', time()));
        $query->setMaxResults(1);
 
        return $query->getSingleResult();
    }
 
    public function getExpiredJob()
    {
        $kernel = static::createKernel();
        $kernel->boot();
        $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
        $query = $em->createQuery('SELECT j from IbwJobeetBundle:Job j WHERE j.expires_at < :date');             
        $query->setParameter('date', date('Y-m-d H:i:s', time()));
        $query->setMaxResults(1);
 
        return $query->getSingleResult();
    }
 
    public function testIndex()
    {
        // get the custom parameters from app config.yml
        $kernel = static::createKernel();
        $kernel->boot();
        $max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');
 
        $client = static::createClient();
 
        $crawler = $client->request('GET', '/');
        $this->assertEquals('IbwJobeetBundleControllerJobController::indexAction', $client->getRequest()->attributes->get('_controller'));
 
        // expired jobs are not listed
        $this->assertTrue($crawler->filter('.jobs td.position:contains("Expired")')->count() == 0);
 
        // only $max_jobs_on_homepage jobs are listed for a category
        $this->assertTrue($crawler->filter('.category_programming tr')->count()<= $max_jobs_on_homepage); 
        $this->assertTrue($crawler->filter('.category_design .more_jobs')->count() == 0);
        $this->assertTrue($crawler->filter('.category_programming .more_jobs')->count() == 1);
 
        // jobs are sorted by date
        $this->assertTrue($crawler->filter('.category_programming tr')->first()->filter(sprintf('a[href*="/%d/"]', $this->getMostRecentProgrammingJob()->getId()))->count() == 1);
 
        // each job on the homepage is clickable and give detailed information
        $job = $this->getMostRecentProgrammingJob();
        $link = $crawler->selectLink('Web Developer')->first()->link();
        $crawler = $client->click($link);
        $this->assertEquals('IbwJobeetBundleControllerJobController::showAction', $client->getRequest()->attributes->get('_controller'));
        $this->assertEquals($job->getCompanySlug(), $client->getRequest()->attributes->get('company'));
        $this->assertEquals($job->getLocationSlug(), $client->getRequest()->attributes->get('location'));
        $this->assertEquals($job->getPositionSlug(), $client->getRequest()->attributes->get('position'));
        $this->assertEquals($job->getId(), $client->getRequest()->attributes->get('id'));
 
        // a non-existent job forwards the user to a 404
        $crawler = $client->request('GET', '/job/foo-inc/milano-italy/0/painter');
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
 
        // an expired job page forwards the user to a 404
        $crawler = $client->request('GET', sprintf('/job/sensio-labs/paris-france/%d/web-developer', $this->getExpiredJob()->getId()));
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
    }
}

```



```php
// src/Ens/JobeetBundle/Test/Controller/CategoryControllerTest.php

namespace IbwJobeetBundleTestsController;
 
use SymfonyBundleFrameworkBundleTestWebTestCase;
use SymfonyBundleFrameworkBundleConsoleApplication;
use SymfonyComponentConsoleOutputNullOutput;
use SymfonyComponentConsoleInputArrayInput;
use DoctrineBundleDoctrineBundleCommandDropDatabaseDoctrineCommand;
use DoctrineBundleDoctrineBundleCommandCreateDatabaseDoctrineCommand;
use DoctrineBundleDoctrineBundleCommandProxyCreateSchemaDoctrineCommand;
 
class CategoryControllerTest extends WebTestCase
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
        $loader = new SymfonyBridgeDoctrineDataFixturesContainerAwareLoader($client->getContainer());
        $loader->loadFromDirectory(static::$kernel->locateResource('@IbwJobeetBundle/DataFixtures/ORM'));
        $purger = new DoctrineCommonDataFixturesPurgerORMPurger($this->em);
        $executor = new DoctrineCommonDataFixturesExecutorORMExecutor($this->em, $purger);
        $executor->execute($loader->getFixtures());
    }
 
    public function testShow()
    {
        $kernel = static::createKernel();
        $kernel->boot();
 
        // get the custom parameters from app/config.yml
        $max_jobs_on_category = $kernel->getContainer()->getParameter('max_jobs_on_category');
        $max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');
 
        $client = static::createClient();
 
        $categories = $this->em->getRepository('IbwJobeetBundle:Category')->getWithJobs();
 
        // categories on homepage are clickable
        foreach($categories as $category) {
            $crawler = $client->request('GET', '/');
 
            $link = $crawler->selectLink($category->getName())->link();
            $crawler = $client->click($link);
 
            $this->assertEquals('IbwJobeetBundleControllerCategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
            $this->assertEquals($category->getSlug(), $client->getRequest()->attributes->get('slug'));
 
            $jobs_no = $this->em->getRepository('IbwJobeetBundle:Job')->countActiveJobs($category->getId()); 
 
            // categories with more than $max_jobs_on_homepage jobs also have a "more" link                 
            if($jobs_no > $max_jobs_on_homepage) {
                $crawler = $client->request('GET', '/');
                $link = $crawler->filter(".category_" . $category->getSlug() . " .more_jobs a")->link();
                $crawler = $client->click($link);
 
                $this->assertEquals('IbwJobeetBundleControllerCategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
                $this->assertEquals($category->getSlug(), $client->getRequest()->attributes->get('slug'));
            }
 
            $pages = ceil($jobs_no/$max_jobs_on_category);
 
            // only $max_jobs_on_category jobs are listed 
            $this->assertTrue($crawler->filter('.jobs tr')->count() <= $max_jobs_on_category);
            $this->assertRegExp("/" . $jobs_no . " jobs/", $crawler->filter('.pagination_desc')->text());
 
            if($pages > 1) {
                $this->assertRegExp("/page 1/" . $pages . "/", $crawler->filter('.pagination_desc')->text());
 
                for ($i = 2; $i <= $pages; $i++) {
                    $link = $crawler->selectLink($i)->link();
                    $crawler = $client->click($link);
 
                    $this->assertEquals('IbwJobeetBundleControllerCategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
                    $this->assertEquals($i, $client->getRequest()->attributes->get('page'));
                    $this->assertTrue($crawler->filter('.jobs tr')->count() <= $max_jobs_on_category);
                    if($jobs_no >1) {
                        $this->assertRegExp("/" . $jobs_no . " jobs/", $crawler->filter('.pagination_desc')->text());
                    }
                    $this->assertRegExp("/page " . $i . "/" . $pages . "/", $crawler->filter('.pagination_desc')->text());
                }
            }     
        }
    }
}
```
