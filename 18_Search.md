# Symfony2 Jobeet Day 17: 搜索

在第14天时我们添加了订阅功能使得Jobeet的用户能获取最新的岗位信息。今天我们将完善用户体验，为Jobeet网站添加最后的主要功能：搜索引擎。

## 所需技术

今天我们想为Jobeet添加一个搜索引擎，而Zend Framework提供了一个好用的库来完成这事儿---[Zend Lucene](http://framework.zend.com/manual/1.12/en/zend.search.lucene.html)。它是广为人知的Java Lucene项目的一部分。因此我们将使用它而不是自己为Jobeet重新创建一个搜索引擎，毕竟那太复杂了。

首先我们要明确，今天并不是Zend Lucene库的使用教程，而是教你如何将它集成到你的Jobeet网站中去。或者更普遍地，教你如何将第三方类库融入你的Symfony项目中。如果你想要关于这项技术的详细信息，请参阅 [Zend Lucene 文档](http://framework.zend.com/manual/1.12/en/zend.search.lucene.html)。

## 安装并配置zend框架

Zend Lucene类库是Zend框架的一部分。因此我们只需要将Zend Framework安装到vendor目录下即可。

为此首先我们下载[Zend Framework](http://framework.zend.com/downloads/latest#ZF1)并将其解压到vendor/Zend目录下。不过请记住，zend 2.*的版本是不包含Zend Lucene库的，所以在本教程中请不要下载它们。

> *以下内容已在1.12.3版本的Zend框架中通过测试*

![Day-17-zend](http://www.intelligentbee.com/wp-content/uploads/2013/08/Day-17-zend.jpg)

你能清除除下列文件之外的所有其他文件：

* Exception.php
* Loader/
* Loader.php
* Search/

接下来将以下代码添加到`autoload.php`文件，通过一个简单的方式来注册Zend自动加载器：

```php
	// app/autoload.php

    // ...

    set_include_path(__DIR__.'/../vendor'.PATH_SEPARATOR.get_include_path());
    require_once __DIR__.'/../vendor/Zend/Loader/Autoloader.php';
    Zend_Loader_Autoloader::getInstance();

    return $loader;
```

## 索引

我们的Jobeet的搜索引擎将返回所有匹配用户输入关键字的职位。在搜索之前我们需要先建立索引，对于本项目来说我们会将索引数据存放在`/web/data`目录下---当然在此之前你需要先建立它。

Zend Lucene 提供两种方法来检索索引，这取决于索引是否已经存在。让我们在Job entity中创建一个辅助方法，用来返回一个存在的索引或者新建一个：

```php
src/Ens/JobeetBundle/Entity/Job.phpPHP

// ...

class Job
{
    // ...

    static public function getLuceneIndex()
    {
        if (file_exists($index = self::getLuceneIndexFile())) {
            return Zend_Search_Lucene::open($index);
        }

        return Zend_Search_Lucene::create($index);
    }

    static public function getLuceneIndexFile()
    {
        return __DIR__.'/../../../../web/data/job.index';
    }
}
```

当每次创建或更新一个职位信息时，这个索引就会被更新。编辑ORM文件，保证每次序列化职位到数据库中时都会更新索引。

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml

#...
    #...
    lifecycleCallbacks:
        #...
        postPersist: [ upload, updateLuceneIndex ]
        postUpdate: [ upload, updateLuceneIndex ]
        #...
```

现在运行generate:entities命令，让Job类中生成updateLuceneIndex()方法

> **php app/console doctrine:generate:entities EnsJobeetBundle**

接下来编辑updateLuceneIndex()方法，使其能实际工作。

```php
// src/Ens/JobeetBundle/Entity/Job.php

class Job
{
    // ...

    public function updateLuceneIndex()
    {
        $index = self::getLuceneIndex();

        // remove existing entries
        foreach ($index->find('pk:'.$this->getId()) as $hit)
        {
          $index->delete($hit->id);
        }

        // don't index expired and non-activated jobs
        if ($this->isExpired() || !$this->getIsActivated())
        {
          return;
        }

        $doc = new \Zend_Search_Lucene_Document();

        // store job primary key to identify it in the search results
        $doc->addField(\Zend_Search_Lucene_Field::Keyword('pk', $this->getId()));

        // index job fields
        $doc->addField(\Zend_Search_Lucene_Field::UnStored('position', $this->getPosition(), 'utf-8'));
        $doc->addField(\Zend_Search_Lucene_Field::UnStored('company', $this->getCompany(), 'utf-8'));
        $doc->addField(\Zend_Search_Lucene_Field::UnStored('location', $this->getLocation(), 'utf-8'));
        $doc->addField(\Zend_Search_Lucene_Field::UnStored('description', $this->getDescription(), 'utf-8'));

        // add job to the index
        $index->addDocument($doc);
        $index->commit();
    }
}

```

由于Zend Lucene是不能够更新现有条目，所以我们需要先移除已存在的索引。

添加Job索引本身很简单：当搜索职位信息时，主键是被保存作为将来的参考，而其他列(position, company, location 和 description)虽然有索引，但并未保存。因为我们是使用real objects来展示结果的。

我们需要添加一个deleteLuceneIndex()方法当删除一个职位信息后移除相应的索引条目。我们的删除操作与更新时相同，通过在ORM文件中 postRemove节添加deleteLuceneIndex()方法来启动 ︰

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml

#...
    #...
    lifecycleCallbacks:
        #...
        postRemove: [ removeUpload, deleteLuceneIndex ]
```

再一次运行generate:entities命令，并打开相应的文件编辑deleteLuceneIndex()方法：

```php
//src/Ens/JobeetBundle/Entity/Job.php

class Job
{
    // ...

    public function deleteLuceneIndex()
    {
        $index = self::getLuceneIndex();

        foreach ($index->find('pk:'.$this->getId()) as $hit) {
            $index->delete($hit->id);
        }
    }
}
```

为了从命令行或者web访问等方式能正确地修改索引，你都必须更改相应的索引目录权限 ︰

> **chmod -R 777 web/data**

现在一切就绪，我们可以重新加载fixture文件并索引它了：

>**php app/console doctrine:fixtures:load**

## 搜索

当完成上述工作之后实现搜索简直是小菜一碟。首先依旧是创建路由：

```yaml
#src/Ens/JobeetBundle/Resources/config/routing/job.yml

#...

ens_job_search:
    pattern: /search
    defaults: { _controller: "EnsJobeetBundle:Job:search" }
```

接着添加相应的action：

```php
// src/Ens/JobeetBundle/Controller/JobController.php

namespace Ens\JobeetBundle\Controller;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Ens\JobeetBundle\Entity\Job;
use Ens\JobeetBundle\Form\JobType;

class JobController extends Controller
{
    // ...

    public function searchAction(Request $request)
    {
        $em = $this->getDoctrine()->getManager();
        $query = $this->getRequest()->get('query');

        if(!$query) {
            return $this->redirect($this->generateUrl('ens_job'));
        }

        $jobs = $em->getRepository('EnsJobeetBundle:Job')->getForLuceneQuery($query);

        return $this->render('EnsJobeetBundle:Job:search.html.twig', array('jobs' => $jobs));
    }
}

```

在searcAction()中，用户如果查询请求不存在或为空将会跳转到JobController的index操作。

模板也是相当的简单︰

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/search.html.twig -->

{% extends 'EnsJobeetBundle::layout.html.twig' %}

{% block stylesheets %}
    {{ parent() }}
    <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/jobs.css') }}" type="text/css" media="all" />
{% endblock %}

{% block content %}
    <div id="jobs">
        {% include 'EnsJobeetBundle:Job:list.html.twig' with {'jobs': jobs} %}
    </div>
{% endblock %}
```

而搜索本身被委派给getForLuceneQuery()方法执行︰

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php

namespace Ens\JobeetBundle\Repository;

use Doctrine\ORM\EntityRepository;
use Ens\JobeetBundle\Entity\Job;

class JobRepository extends EntityRepository
{
    // ...

    public function getForLuceneQuery($query)
    {
        $hits = Job::getLuceneIndex()->find($query);

        $pks = array();
        foreach ($hits as $hit)
        {
          $pks[] = $hit->pk;
        }

        if (empty($pks))
        {
          return array();
        }

        $q = $this->createQueryBuilder('j')
            ->where('j.id IN (:pks)')
            ->setParameter('pks', $pks)
            ->andWhere('j.is_activated = :active')
            ->setParameter('active', 1)
            ->setMaxResults(20)
            ->getQuery();

        return $q->getResult();
    }
}
```

我们从Lucene索引得到所有的结果后，我们筛选掉未激活活动状态的职位信息，并将其数量限制为20条。

最后更新一下布局就可以开始工作了：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->

<!-- ... -->
    <!-- ... -->
        <div class="search">
            <h2>Ask for a job</h2>
            <form action="{{ path('ens_job_search') }}" method="get">
                <input type="text" name="query" value="{{ app.request.get('query') }}" id="search_keywords" />
                <input type="submit" value="search" />
                <div class="help">
                    Enter some keywords (city, country, position, ...)
                </div>
            </form>
        </div>
    <!-- ... -->
<!-- ... -->
```

## 单元测试

我们需要创建什么样的测试案例来对这个搜索引擎来进行单元测试呢？显然我们不会对Zend Lucene库本身进行测试，而是要去测试其与Job类集成的部分。

在JobRepositoryTest.php文件的最后添加如下代码：

```php
// src/Ens/JobeetBundle/Tests/Repository/JobRepositoryTest.php

// ... 
use Ens\JobeetBundle\Entity\Job;

class JobRepositoryTest extends WebTestCase
{
    // ...

    public function testGetForLuceneQuery()
    {
        $em = static::$kernel->getContainer()
            ->get('doctrine')
            ->getManager();

        $job = new Job();
        $job->setType('part-time');
        $job->setCompany('Sensio');
        $job->setPosition('FOO6');
        $job->setLocation('Paris');
        $job->setDescription('WebDevelopment');
        $job->setHowToApply('Send resumee');
        $job->setEmail('jobeet@example.com');
        $job->setUrl('http://sensio-labs.com');
        $job->setIsActivated(false);

        $em->persist($job);
        $em->flush();

        $jobs = $em->getRepository('EnsJobeetBundle:Job')->getForLuceneQuery('FOO6');
        $this->assertEquals(count($jobs), 0);

        $job = new Job();
        $job->setType('part-time');
        $job->setCompany('Sensio');
        $job->setPosition('FOO7');
        $job->setLocation('Paris');
        $job->setDescription('WebDevelopment');
        $job->setHowToApply('Send resumee');
        $job->setEmail('jobeet@example.com');
        $job->setUrl('http://sensio-labs.com');
        $job->setIsActivated(true);

        $em->persist($job);
        $em->flush();

        $jobs = $em->getRepository('EnsJobeetBundle:Job')->getForLuceneQuery('position:FOO7');
        $this->assertEquals(count($jobs), 1);
        foreach ($jobs as $job_rep) {
            $this->assertEquals($job_rep->getId(), $job->getId());
        }

        $em->remove($job);
        $em->flush();

        $jobs = $em->getRepository('EnsJobeetBundle:Job')->getForLuceneQuery('position:FOO7');

        $this->assertEquals(count($jobs), 0);
    }
}

```

我们测试未激活或已删除的工作将不显示在你的搜索结果中;同时我们还会检查给定匹配条件相应职位在结果中显示。

## 计划任务

最终我们需要更新JobeetCleanup计划任务用来清理陈旧的索引(例如当职位信息到期之后)以此来进行搜索优化。

```php
// src/Ens/JobeetBundle/Command/JobeetCleanupCommand.php

// ...
use  Ens\JobeetBundle\Entity\Job;

class JobeetCleanupCommand extends ContainerAwareCommand
{
    // ...

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $days = $input->getArgument('days');

        $em = $this->getContainer()->get('doctrine')->getManager();

        // cleanup Lucene index
        $index = Job::getLuceneIndex();

        $q = $em->getRepository('EnsJobeetBundle:Job')->createQueryBuilder('j')
          ->where('j.expires_at < :date')
          ->setParameter('date',date('Y-m-d'))
          ->getQuery();

        $jobs = $q->getResult();
        foreach ($jobs as $job)
        {
          if ($hit = $index->find('pk:'.$job->getId()))
          {
            $index->delete($hit->id);
          }
        }

        $index->optimize();

        $output->writeln('Cleaned up and optimized the job index');

        // Remove stale jobs
        $nb = $em->getRepository('EnsJobeetBundle:Job')->cleanup($days);

        $output->writeln(sprintf('Removed %d stale jobs', $nb));
    }
}

```

该任务删除所有已过期职位信息的索引并进行优化，这有赖于Zend Lucene内置optimize()方法。

这一天，我们在不到一个小时的时间里实现了完整的搜索引擎。每次你当想要向你的项目中添加一个新功能时，请先检查它是否尚未被解决，不要重复造轮子。

明天我们将使用一些unobtrusive的JavaScripts来提升用户体验---当用户在搜索栏输入时实时更新返回的结果并展示。当然，这将会谈论如何在Symfony中使用AJAX。

>*译者注：unobtrusive是指一种web编程理念，即将js脚本与html完全分离，那么当没有js或用户禁止js时也能正常浏览使用网页，只是用户体验较差。更多信息可查看[Unobtrusive JavaScript - Wikipedia](https://en.wikipedia.org/wiki/Unobtrusive_JavaScript)和[Unobtrusive Ajax - Jesse Skinner](http://www.thefutureoftheweb.com/talks/2006-10-ajax-experience/slides/)*