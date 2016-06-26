# Symfony2 Jobeet Day 6: 深入Model

## Doctrine Query对象
在第二天的需求中我们知道：在首页，用户能看到最新的有效职位信息，但现在所有信息都被显示了，无论其是否有效

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
class JobController extends Controller
{
  public function indexAction()
  {
    $em = $this->getDoctrine()->getEntityManager();
 
    $entities = $em->getRepository('EnsJobeetBundle:Job')->findAll();
 
    return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
      'entities' => $entities
    ));
 
  // ...
}
```

一个有效的职位应该在30天之内，然而
`$entities = $em->getRepository('EnsJobeetBundle:Job')->findAll();`方法将会返回所有的职位信息，这样不加条件的检索显然是不可行的。
现在让我们再来施点魔法，使得它能筛选出有效的职位。

```php
public function indexAction()
{
  $em = $this->getDoctrine()->getEntityManager();
 
  $query = $em->createQuery(
    'SELECT j FROM EnsJobeetBundle:Job j WHERE j.created_at > :date'
  )->setParameter('date', date('Y-m-d H:i:s', time() - 86400 * 30));
  $entities = $query->getResult();
 
  return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
    'entities' => $entities
  ));
}
```

## 调试Doctrine生成的SQL
有时能看到Doctrine生成的SQL对我们的帮助会很大，比如一个查询不能像我们期望的那样工作。
幸运的是在开发环境中我们有Symfony web 开发调试栏，你所要的所有信息都能在这里找到(http://jobeet.local/app_dev.php):
![jobeet_debug_sql](http://www.ens.ro/wp-content/uploads/2012/04/jobeet_debug_sql-1024x763.png)
 

## 对象序列化
此时你一定以为差不多完美了，然而我又要不幸的告诉你这还有一些瑕疵。正如第二天所说的，用户可以重新回来激活或者延长这份职位30天的有效性。。。

因为上面的代码只是依赖created_at即创建日期，这显然不能满足我们的需求

如果你还记的在第三天定义的数据库的话，应该知道有一个expires_at列。当前它还是空的，现在我们需要做的就是当一个职位被创建的时候，将它的值自动设置为当前日期之后的30天

当你需要做一些自动化设置在Doctrine对象序列化进入数据库之前，你可以添加一个新的action在lifecycle callback的map对象中，就像我们稍早前的created_at列一样。

```yaml
# src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml
# ...
 
  lifecycleCallbacks:
    prePersist: [ setCreatedAtValue, setExpiresAtValue ]
    preUpdate: [ setUpdatedAtValue ]
```

现在我们重新生成的实体类中将会添加一个新的方法

> **php app/console doctrine:generate:entities EnsJobeetBundle**

打开`src/Ens/JobeetBundle/Entity/Job.php`文件并编辑这个新的函数

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
 
public function setExpiresAtValue()
{
  if(!$this->getExpiresAt())
  {
    $now = $this->getCreatedAt() ? $this->getCreatedAt()->format('U') : time();
    $this->expires_at = new \DateTime(date('Y-m-d H:i:s', $now + 86400 * 30));
  }
}
```

现在让我们来修改一下action，将created_at改为expires_at 使得它能正确筛选职位


## 更多数据

当你刷新Jobeet的主页时并不会发现有任何的改变---因为没有数据---现在让我们来添加一些：

```php
// src/Ens/JobeetBundle/DataFixtures/ORM/LoadJobData.php
// ...
 
$job_expired = new Job();
$job_expired->setCategory($em->merge($this->getReference('category-programming')));
$job_expired->setType('full-time');
$job_expired->setCompany('Sensio Labs');
$job_expired->setLogo('sensio-labs.gif');
$job_expired->setUrl('http://www.sensiolabs.com/');
$job_expired->setPosition('Web Developer Expired');
$job_expired->setLocation('Paris, France');
$job_expired->setDescription('Lorem ipsum dolor sit amet, consectetur adipisicing elit.');
$job_expired->setHowToApply('Send your resume to lorem.ipsum [at] dolor.sit');
$job_expired->setIsPublic(true);
$job_expired->setIsActivated(true);
$job_expired->setToken('job_expired');
$job_expired->setEmail('job@example.com');
$job_expired->setCreatedAt(new \DateTime('2005-12-01'));
 
// ...
 
$em->persist($job_expired);
```

重新加载fixtures并刷新浏览器

> **php app/console doctrine:fixtures:load**

## 重构
尽管我们的代码已经能很好的完成任务了，但还是有一些不对（这次我没说然而<_<）,你能发现其中的问题吗？
Doctrine的查询代码并不属于action（Controller层），它应该属于Model层。在 MVC 模型中，Model定义了所有的业务逻辑，而Controller仅调用从Model查询所返回的数据。
因此让我们把代码移动到Model，并返回一个职位的collection
为此，我们需要在Job entity中添加一个repository类，并在其中添加相应的方法

打开`/src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml`并修改如下位置

```yaml
# /src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml
 
Ens\JobeetBundle\Entity\Job:
  type: entity
  repositoryClass: Ens\JobeetBundle\Repository\JobRepository
  # ...
```

运行以下代码，使Doctrine帮我们创建一个repository类

> **php app/console doctrine:generate:entities EnsJobeetBundle**

接下来我们在这个repository类中添加一个新的getActiveJobs()方法。这个方法将查到所有有效的并且按expires_at排序的Job entities（同时如果你传递了$category_id参数的话，它将还会按类别进行筛选。）

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php
 
namespace Ens\JobeetBundle\Repository;
use Doctrine\ORM\EntityRepository;
 
class JobRepository extends EntityRepository
{
  public function getActiveJobs($category_id = null)
  {
    $qb = $this->createQueryBuilder('j')
      ->where('j.expires_at > :date')
      ->setParameter('date', date('Y-m-d H:i:s', time()))
      ->orderBy('j.expires_at', 'DESC');
 
    if($category_id)
    {
      $qb->andWhere('j.category = :category_id')
         ->setParameter('category_id', $category_id);
    }
 
    $query = $qb->getQuery();
 
    return $query->getResult();
  }
}
```

现在action能使用这个方法来查询数据了。

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function indexAction()
{
  $em = $this->getDoctrine()->getEntityManager();
 
  $entities = $em->getRepository('EnsJobeetBundle:Job')->getActiveJobs();
 
  return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
    'entities' => $entities
  ));
}
 
// ...
```

这次重构可以带来以下优点：
* 读取有效职位的逻辑代码都放到Model层，在它该在的地方
* 在控制器中的代码是更简洁、 更具可读性
* 使得getActiveJobs()方法能在更多的地方使用
* Model层代码现在可以进行单元测试

> *译者注：如果你之前使用Annotations方法生成Enity的话此时已经自动生成相应的repository，你可以直接在里面修改。*

## 主页分类

根据第二天的要求，我们需要将职位职位按类别排序。但直到现在我们却并没有考虑过职位分类这个功能。依据要求我们需要按不同的分类来展示不同的职位。首先我们要获取所有有效职位的所有分类

为Category entity创建一个repository类:

```yaml
#/src/Ens/JobeetBundle/Resources/config/doctrine/Category.orm.yml
 
Ens\JobeetBundle\Entity\Category:
  type: entity
  repositoryClass: Ens\JobeetBundle\Repository\CategoryRepository
  #...
```

然后运行：

> **php app/console doctrine:generate:entities EnsJobeetBundle**

打开CategoryRepository类并向其中添加getWithJobs()方法：

```php
// src/Ens/JobeetBundle/Repository/CategoryRepository.php
 
namespace Ens\JobeetBundle\Repository;
use Doctrine\ORM\EntityRepository;
 
class CategoryRepository extends EntityRepository
{
  public function getWithJobs()
  {
    $query = $this->getEntityManager()->createQuery(
      'SELECT c FROM EnsJobeetBundle:Category c LEFT JOIN c.jobs j WHERE j.expires_at > :date'
    )->setParameter('date', date('Y-m-d H:i:s', time()));
 
    return $query->getResult();
  }
}
```

并相应的修改index action

```php
public function indexAction()
{
  $em = $this->getDoctrine()->getEntityManager();
 
  $categories = $em->getRepository('EnsJobeetBundle:Category')->getWithJobs();
 
  foreach($categories as $category)
  {
    $category->setActiveJobs($em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId()));
  }
 
  return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
    'categories' => $categories
  ));
}
```

为了能使其正常工作，我们还需要为Category类添加新的属性操作：

```php
// src/Ens/JobeetBundle/Entity/Category.php
 
namespace Ens\JobeetBundle\Entity;
use Doctrine\ORM\Mapping as ORM;
 
class Category
{
  // ...
 
  private $active_jobs;
 
  // ...
 
  public function setActiveJobs($jobs)
  {
    $this->active_jobs = $jobs;
  }
 
  public function getActiveJobs()
  {
    return $this->active_jobs;
  }
}
```

在模板中我们需要遍历所有的类别并展示有效职位：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->
<!-- ... -->
 
{% block content %}
  <div id="jobs">
    {% for category in categories %}
      <div>
        <div class="category">
          <div class="feed">
            <a href="">Feed</a>
          </div>
          <h1>{{ category.name }}</h1>
        </div>
        <table class="jobs">
          {% for entity in category.activejobs %}
            <tr class="{{ cycle(['even', 'odd'], loop.index) }}">
              <td class="location">{{ entity.location }}</td>
              <td class="position">
                <a href="{{ path('ens_job_show', { 'id': entity.id, 'company': entity.companyslug, 'location': entity.locationslug, 'position': entity.positionslug }) }}">
                  {{ entity.position }}
                </a>
              </td>
              <td class="company">{{ entity.company }}</td>
            </tr>
          {% endfor %}
        </table>
      </div>
    {% endfor %}
  </div>
{% endblock %}
```

## 限制结果集

我们还有一个要求没有实现，就是将结果限制在10条以内。这很容易实现，只要在JobRepository::getActiveJobs()方法中添加一个$max参数即可。

```php
public function getActiveJobs($category_id = null, $max = null)
{
  $qb = $this->createQueryBuilder('j')
    ->where('j.expires_at > :date')
    ->setParameter('date', date('Y-m-d H:i:s', time()))
    ->orderBy('j.expires_at', 'DESC');
 
  if($max)
  {
    $qb->setMaxResults($max);
  }
 
  if($category_id)
  {
    $qb->andWhere('j.category = :category_id')
       ->setParameter('category_id', $category_id);
  }
 
  $query = $qb->getQuery();
 
  return $query->getResult();
}
```

修改调用getActiveJobs的地方，使其传入$max参数

```php
// src/Ens/JobeetBundle/Controller/JobController.php
 
$category->setActiveJobs($em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), 10));
```

## 自定义配置

在JobController的indexAction方法中我们是使用硬编码来限制结果集的，这显然不算是一个好的方法，更好的方式是将其放入配置文件中。在Symfony中，你应用程序的所有自定义参数都放在`app/config/config.yml`文件中的parameters键下（如果不存在则新建）：

```yaml
#app/config/config.yml
#...
 
parameters:
    max_jobs_on_homepage: 10
```

在控制器中修改如下:

```php
// src/Ens/JobeetBundle/Controller/JobController
// ...
 
public function indexAction()
{
  $em = $this->getDoctrine()->getEntityManager();
 
  $categories = $em->getRepository('EnsJobeetBundle:Category')->getWithJobs();
 
  foreach($categories as $category)
  {
    $category->setActiveJobs($em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), $this->container->getParameter('max_jobs_on_homepage')));
  }
 
  return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
    'categories' => $categories
  ));
}
```

## 我还要数据

当你又一次兴冲冲的在浏览器中等待奇迹发生的时候，无情的现实又把你pia到地面。因为我们现在的数据还是依旧太少。当然你可以不断的使用ctrl-c&ctrl-v在fixture中添加数据，但这并不是什么明智的做法，重复即意味着糟糕，即使在fixture中也是如此。

```php
// src/Ens/JobeetBundle/DataFixtures/ORM/LoadJobData.php
// ...
 
public function load(ObjectManager $em)
{
  // ...
 
  for($i = 100; $i <= 130; $i++)
  {
    $job = new Job();
    $job->setCategory($em->merge($this->getReference('category-programming')));
    $job->setType('full-time');
    $job->setCompany('Company '.$i);
    $job->setPosition('Web Developer');
    $job->setLocation('Paris, France');
    $job->setDescription('Lorem ipsum dolor sit amet, consectetur adipisicing elit.');
    $job->setHowToApply('Send your resume to lorem.ipsum [at] dolor.sit');
    $job->setIsPublic(true);
    $job->setIsActivated(true);
    $job->setToken('job_'.$i);
    $job->setEmail('job@example.com');
 
    $em->persist($job);
  }
 
  $em->flush();
}
 
// ...
```

一切就绪后，别忘了doctrine:fixtures:load 。
好了，现在请你打开浏览器，点击刷新或者F5吧。
![day6_model_dinamic_fixtures](http://www.ens.ro/wp-content/uploads/2012/04/Screenshot-at-2012-04-07-101816-1024x612.png) 

安全的详情页
当一份职位招聘过期之后，即使你知道它的URL，也理应不能访问。尝试访问一个已经过期的职位页。比如：
/app_dev.php/job/sensio-labs/paris-france/ID/web-developer-expired
将其中的ID替换为已经过期的数据的ID（获取方法  SELECT id, token FROM jobeet_job WHERE expires_at < NOW()）
你会发现如今依旧是可以访问的，因此我们需要给用户一个404页面来代替当前页面。
我们在JobRepository中添加一个方法

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php
// ...
 
public function getActiveJob($id)
{
  $query = $this->createQueryBuilder('j')
    ->where('j.id = :id')
    ->setParameter('id', $id)
    ->andWhere('j.expires_at > :date')
    ->setParameter('date', date('Y-m-d H:i:s', time()))
    ->setMaxResults(1)
    ->getQuery();
 
  try {
    $job = $query->getSingleResult();
  } catch (\Doctrine\Orm\NoResultException $e) {
    $job = null;
  }
 
  return $job;
}
```

> *注：getSingleResult()方法有可能抛出两个异常：
当没有结果返回时抛出Doctrine\ORM\NoResultException异常
当返回的记录不止一条时抛出Doctrine\ORM\NonUniqueResultException异常
因此你需要使用try-catch来确保结果的唯一性。*

现在修改JobController的showAction来使用新的repository方法

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
$entity = $em->getRepository('EnsJobeetBundle:Job')->getActiveJob($id);
 
// ..
```

此刻你如果重新刷新该页面应该能看到一个404页
![day6_model_secure_the_job_page](http://www.ens.ro/wp-content/uploads/2012/04/Screenshot-at-2012-04-07-122445-1024x625.png)

OK，今天的课程就到此结束了，see you tomorrow