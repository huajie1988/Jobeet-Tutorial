# Symfony2 Jobeet Day 7: 玩转分类页

今天我们将创建分页类像我们第二天所描述的那样：“用户可以看到按类别和日期排序并且分页的所有工作的列表，每页有20个职位。”

## 分类路由
首先我们要为分类页添加一个好看的路由，打开路由文件，并在开头写上：

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml
 
EnsJobeetBundle_category:
    pattern:  /category/{slug}
    defaults: { _controller: EnsJobeetBundle:Category:show }
```

为了得到slug 我们要将getSlug()方法添加到category类中：

```php
// src/Ens/JobeetBundle/Entity/Category.php
// ...
 
use Ens\JobeetBundle\Utils\Jobeet;
 
class Category
{
  // ...
 
  public function getSlug()
  {
    return Jobeet::slugify($this->getName());
  }
 
  // ...
}
```

## 添加分类链接
打开index.html.twig并添加分类页的链接：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->
<!-- some HTML code -->
 
            <h1><a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug }) }}">{{ category.name }}</a></h1>
 
<!-- some HTML code -->
 
          </table>
 
          {% if category.morejobs %}
            <div class="more_jobs">
              and <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug }) }}">{{ category.morejobs }}</a>
              more...
            </div>
          {% endif %}
        </div>
      {% endfor %}
    </div>
{% endblock %}
```
在模板中我们需要使用category.morejobs，现在然我们来定义一下：

```php
// src/Ens/JobeetBunlde/Entity/Category.php
// ...
 
private $more_jobs;
 
// ...
 
public function setMoreJobs($jobs)
{
  $this->more_jobs = $jobs >=  0 ? $jobs : 0;
}
 
public function getMoreJobs()
{
  return $this->more_jobs;
}
 
// ...
```

more_jobs属性是指所有该分类的有效职位数量减去在首页列表中已显示的该分类的职位数量。具体看代码：

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function indexAction()
{
  $em = $this->getDoctrine()->getEntityManager();
 
  $categories = $em->getRepository('EnsJobeetBundle:Category')->getWithJobs();
 
  foreach($categories as $category)
  {
    $category->setActiveJobs($em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), $this->container->getParameter('max_jobs_on_homepage')));
    $category->setMoreJobs($em->getRepository('EnsJobeetBundle:Job')->countActiveJobs($category->getId()) - $this->container->getParameter('max_jobs_on_homepage'));
  }
 
  return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
    'categories' => $categories
  ));
}
// ...
```

接着在JobRepository里面添加countActiveJobs方法：

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php
// ...
 
public function countActiveJobs($category_id = null)
{
  $qb = $this->createQueryBuilder('j')
    ->select('count(j.id)')
    ->where('j.expires_at > :date')
    ->setParameter('date', date('Y-m-d H:i:s', time()));
 
  if($category_id)
  {
    $qb->andWhere('j.category = :category_id')
    ->setParameter('category_id', $category_id);
  }
 
  $query = $qb->getQuery();
 
  return $query->getSingleScalarResult();
}
 
// ...
```

现在打开浏览器刷新之后就可以看到如下效果：
![day8_the_category_link1](http://www.ens.ro/wp-content/uploads/2012/04/SC4OSY1-1024x742.png) 

## 创建分类控制器
现在是时候创建category 控制器了。打开Controller文件夹并在其中新建一个CategoryController.php文件

```php
// src/Ens/JobeetBundle/Controller/CategoryController
 
namespace Ens\JobeetBundle\Controller;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Ens\JobeetBundle\Entity\Category;
 
/**
* Category controller.
*
*/
class CategoryController extends Controller
{
 
}
```

当然我们此时也可以使用doctrine:generate:crud命令来创建基本的操作，但其中90%的代码我们并不需要，所以我们只要手动创建一个控制器即可

## 升级数据库

我们需要在category表中添加slug列并将其添加到lifecycle中，使其能在插入时自动填充

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Category.orm.yml
 
Ens\JobeetBundle\Entity\Category:
  type: entity
  repositoryClass: Ens\JobeetBundle\Repository\CategoryRepository
  table: category
  id:
    id:
      type: integer
      generator: { strategy: AUTO }
  fields:
    name:
      type: string
      length: 255
      unique: true
    slug:
      type: string
      length: 255
      unique: true
  oneToMany:
    jobs:
      targetEntity: Job
      mappedBy: category
    category_affiliates:
      targetEntity: CategoryAffiliate
      mappedBy: category
  lifecycleCallbacks:
    prePersist: [ setSlugValue ]
    preUpdate: [ setSlugValue ]
```

移除我们稍早之前在Category entity (`src/Ens/JobeetBundle/Entity/Category.php`)中创建的getSlug方法，并执行如下命令更新Category entity类

> **php app/console doctrine:generate:entities EnsJobeetBundle**

现在我们再看Category.php就能看到被添加的方法

```php
// srr/Ens/JobeetBundle/Entity/Category.php
// ...
 
/**
* @var string $slug
*/
private $slug;
 
/**
* Set slug
*
* @param string $slug
*/
public function setSlug($slug)
{
    $this->slug = $slug;
}
/**
* Get slug
*
* @return string
*/
public function getSlug()
{
    return $this->slug;
}
/**
* @ORM\prePersist
*/
public function setSlugValue()
{
    $this->slug = Jobeet::slugify($this->getName());
}
```

我们现在不得不删除数据库并创建一个新的列在category表中，并加载数据：

> **php app/console doctrine:database:drop --force**  
> **php app/console doctrine:database:create**  
> **php app/console doctrine:schema:update --force**  
> **php app/console doctrine:fixtures:load**  

## 分类页
现在我们可将如下代码添加到CategoryController.php中的showAction()方法中

```php
// src/Ens/JobeetBundle/Controller/CategoryController.php
// ...
 
public function showAction($slug)
{
    $em = $this->getDoctrine()->getEntityManager();
 
    $category = $em->getRepository('EnsJobeetBundle:Category')->findOneBySlug($slug);
 
    if (!$category) {
        throw $this->createNotFoundException('Unable to find Category entity.');
    }
 
    $category->setActiveJobs($em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId()));
 
    return $this->render('EnsJobeetBundle:Category:show.html.twig', array(
        'category' => $category,
    ));
}
 
// ...
```

接下来创建show.html.twig模板：

```html
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% block title %}
  Jobs in the {{ category.name }} category
{% endblock %}
 
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/jobs.css') }}" type="text/css" media="all" />
{% endblock %}
 
{% block content %}
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
{% endblock %}
```
 
## 包含其他Twig模板
现在我们已经看到我们从job的index.html.twig模板中复制了一份<table>，这显然不是好的设计。当你需要重复使用模板的某些部分时，你可以创建一个新的Twig模板并将相同的部分写入，然后在需要用到的地方再包含进来。

创建一个list.html.twig文件：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/list.html.twig -->
 
<table class="jobs">
  {% for entity in jobs %}
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
```

此时你只要使用{% include %}标签即可替换<table>部分的HTML代码：

```html
<!-- in src/Ens/JobeetBundle/Resources/view/Job/index.html.twig -->
{% include 'EnsJobeetBundle:Job:list.html.twig' with {'jobs': category.activejobs} %}
 
<!-- in src/Ens/JobeetBundle/Resources/view/Category/show.html.twig -->
{% include 'EnsJobeetBundle:Job:list.html.twig' with {'jobs': category.activejobs} %}
```

是不是很鹅妹子嘤啊
 
## 列表分页

至今为止（至作者写本文时）Symfony2并没有提供任何好用的分页工具，为了解决这个问题我们使用一个古老但经典的方法来完成它。

首先，让我们在EnsJobeetBundle_category路由中添加一个page参数。这个参数的默认值为1，所以它也可以不存在

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml
 
EnsJobeetBundle_category:
    pattern: /category/{slug}/{page}
    defaults: { _controller: EnsJobeetBundle:Category:show, page: 1 }
 
#...
```

然后在 app/config/config.yml中定义每页的职位数量

```yaml
#app/config/config.yml
#...
 
parameters:
    max_jobs_on_homepage: 10
    max_jobs_on_category: 20
```

修改JobRepository中的getActiveJobs方法，使其包含一个$offset参数传递给doctrine检索职位时使用

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php
// ...
 
public function getActiveJobs($category_id = null, $max = null, $offset = null)
{
  $qb = $this->createQueryBuilder('j')
    ->where('j.expires_at > :date')
    ->setParameter('date', date('Y-m-d H:i:s', time()))
    ->orderBy('j.expires_at', 'DESC');
 
  if($max)
  {
    $qb->setMaxResults($max);
  }
 
  if($offset)
  {
    $qb->setFirstResult($offset);
  }
 
  if($category_id)
  {
    $qb->andWhere('j.category = :category_id')
       ->setParameter('category_id', $category_id);
  }
 
  $query = $qb->getQuery();
 
  return $query->getResult();
}
 
// ...
```

修改CategoryController 的showAction：

```php
// src/Ens/JobeetBundle/Controller/CategoryController.php
// ...
 
public function showAction($slug, $page)
{
  $em = $this->getDoctrine()->getEntityManager();
 
  $category = $em->getRepository('EnsJobeetBundle:Category')->findOneBySlug($slug);
 
  if (!$category) {
    throw $this->createNotFoundException('Unable to find Category entity.');
  }
 
  $total_jobs = $em->getRepository('EnsJobeetBundle:Job')->countActiveJobs($category->getId());
  $jobs_per_page = $this->container->getParameter('max_jobs_on_category');
  $last_page = ceil($total_jobs / $jobs_per_page);
  $previous_page = $page > 1 ? $page - 1 : 1;
  $next_page = $page < $last_page ? $page + 1 : $last_page;
 
  $category->setActiveJobs($em->getRepository('EnsJobeetBundle:Job')->getActiveJobs($category->getId(), $jobs_per_page, ($page - 1) * $jobs_per_page));
 
  return $this->render('EnsJobeetBundle:Category:show.html.twig', array(
    'category' => $category,
    'last_page' => $last_page,
    'previous_page' => $previous_page,
    'current_page' => $page,
    'next_page' => $next_page,
    'total_jobs' => $total_jobs
  ));
}
```

最后我们来修改模板：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Category/show.html.twig -->
 
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% block title %}
  Jobs in the {{ category.name }} category
{% endblock %}
 
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/jobs.css') }}" type="text/css" media="all" />
{% endblock %}
 
{% block content %}
  <div class="category">
    <div class="feed">
      <a href="">Feed</a>
    </div>
    <h1>{{ category.name }}</h1>
  </div>
 
  {% include 'EnsJobeetBundle:Job:list.html.twig' with {'jobs': category.activejobs} %}
 
  {% if last_page > 1 %}
    <div class="pagination">
      <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, 'page': 1 }) }}">
        <img src="{{ asset('bundles/ensjobeet/images/first.png') }}" alt="First page" title="First page" />
      </a>
 
      <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, 'page': previous_page }) }}">
        <img src="{{ asset('bundles/ensjobeet/images/previous.png') }}" alt="Previous page" title="Previous page" />
      </a>
 
      {% for page in 1..last_page %}
        {% if page == current_page %}
          {{ page }}
        {% else %}
          <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, 'page': page }) }}">{{ page }}</a>
        {% endif %}
      {% endfor %}
 
      <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, 'page': next_page }) }}">
        <img src="{{ asset('bundles/ensjobeet/images/next.png') }}" alt="Next page" title="Next page" />
      </a>
 
      <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, 'page': last_page }) }}">
        <img src="{{ asset('bundles/ensjobeet/images/last.png') }}" alt="Last page" title="Last page" />
      </a>
    </div>
  {% endif %}
 
  <div class="pagination_desc">
    <strong>{{ total_jobs }}</strong> jobs in this category
 
    {% if last_page > 1 %}
      - page <strong>{{ current_page }}/{{ last_page }}</strong>
    {% endif %}
  </div>
{% endblock %}
```

OK，让我们来看看效果吧。
![day8_list_pagination_1](http://www.ens.ro/wp-content/uploads/2012/04/SX5P5ZT-1024x719.png) 

> *译者注：你也可以考虑使用第三方分页类，比如knp-paginator这种，具体方法可在https://packagist.org/中查找使用。*

