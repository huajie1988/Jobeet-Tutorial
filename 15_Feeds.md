# Symfony2 Jobeet Day 14: 订阅

如果你正在寻找一份工作，可能会希望尽快得到最新发布的职位信息。然而每个小时去网站查看这些信息显然是不现实的。为此我们需要添加一个订阅功能，保证我们的用户总能在第一时间收到最新的职位信息。

## 格式化模板

模板是用来呈现格式化内容的通用方式。虽然在大多数情况下你仅仅使用模板来呈现HTML的内容，但实际上模板可以很容易地生成 JavaScript、 CSS、 XML 或其他任何格式。

例如，相同"资源"经常可以渲染成多个不同的格式，我们只需要在文件名称中加上需要渲染的格式名即可做不同的格式化渲染。例如要把文章首页渲染成XML，只要在文件名中包含xml即可 ︰

* XML 模板名称: AcmeArticleBundle:Article:index.xml.twig
* XML 模板文件名: index.xml.twig

当然实际上这仅仅只是一种命名约定，模板并不会实际将其转化为我们所要的格式。

> *译者注：没卯月你说那么多干啥=_=|||*

在许多情况下，你可能想要使用单一控制器来实现不同的"格式请求"，渲染的格式化内容也不同。出于这个原因，一种常见模式是进行以下操作 ︰

```php
public function indexAction()
{
    $format = $this->getRequest()->getRequestFormat();
 
    return $this->render('AcmeBlogBundle:Blog:index.'.$format.'.twig');
}
```

getRequestFormat在请求对象时默认为html，但也可以根据用户所请求的格式，返回任何其他格式。请求格式常被路由所管理当一个路由被设置为/contact时则请求的格式为html，如果设置为/contact.xml则请求的格式为xml

若要创建包含格式参数的链接，我们需要在参数中包含_format关键字：

```html
<a href="{{ path('article_show', {'id': 123, '_format': 'pdf'}) }}">
    PDF Version
</a>
```

## 订阅

### 最新职位订阅

支持不同的格式就和创建不同的模板一样简单。若我们要创建一个Atom格式的最新职位信息订阅，请创建 index.atom.twig 模板 ︰

```xml
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.atom.twig -->
<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Jobeet</title>
  <subtitle>Latest Jobs</subtitle>
  <link href="" rel="self"/>
  <link href=""/>
  <updated></updated>
  <author><name>Jobeet</name></author>
  <id>Unique Id</id>
 
  <entry>
    <title>Job title</title>
    <link href="" />
    <id>Unique id</id>
    <updated></updated>
    <summary>Job description</summary>
    <author><name>Company</name></author>
  </entry>
</feed>
```

在Jobeet的底部添加一个订阅链接：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->
<!-- ... -->
 
<li><a href="{{ path('ens_job', {'_format': 'atom'}) }}">Full feed</a></li>
 
<!-- ... -->
```

在布局文件中添加一个<link>标签，使得浏览器能自动发现我们的订阅链接：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->
<!-- ... -->
 
<link rel="alternate" type="application/atom+xml" title="Latest Jobs" href="{{ url('ens_job', {'_format': 'atom'}) }}" />
 
<!-- ... -->
```

修改JobController中的indexAction，使其能根据_format参数来选择相应的模板：

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
$format = $this->getRequest()->getRequestFormat();
 
return $this->render('EnsJobeetBundle:Job:index.'.$format.'.twig', array(
    'categories' => $categories
));
 
// ...
```

用以下代码替换atom模板的头部 ︰

```xml
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.atom.twig -->
<!-- ... -->
 
  <title>Jobeet</title>
  <subtitle>Latest Jobs</subtitle>
  <link href="{{ url('ens_job', {'_format': 'atom'}) }}" rel="self"/>
  <link href="{{ url('EnsJobeetBundle_homepage') }}"/>
  <updated>{{ lastUpdated }}</updated>
  <author><name>Jobeet</name></author>
  <id>{{ feedId }}</id>
 
<!-- ... -->
```

接着把lastUpdated 和 feedId 传递给模板：

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
return $this->render('EnsJobeetBundle:Job:index.'.$format.'.twig', array(
    'categories' => $categories,
    'lastUpdated' => $em->getRepository('EnsJobeetBundle:Job')->getLatestPost()->getCreatedAt()->format(DATE_ATOM),
    'feedId' => sha1($this->get('router')->generate('ens_job', array('_format'=> 'atom'), true)),
));
 
// ...
```

为了得到最新发布的职位信息时间，我们需要在JobRepository里添加getLatestPost()方法：

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php
// ...
 
public function getLatestPost()
{
    $query = $this->createQueryBuilder('j')
        ->where('j.expires_at > :date')
        ->setParameter('date', date('Y-m-d H:i:s', time()))
        ->andWhere('j.is_activated = :activated')
        ->setParameter('activated', 1)
        ->orderBy('j.expires_at', 'DESC')
        ->setMaxResults(1)
        ->getQuery();
 
    try {
        $job = $query->getSingleResult();
    } catch (\Doctrine\Orm\NoResultException $e) {
        $job = null;
    }
 
    return $job;
}
 
// ...
```

具体的订阅信息代码如下：

```xml
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.atom.twig -->
<!-- ... -->
 
{% for category in categories %}
  {% for entity in category.activejobs %}
    <entry>
      <title>{{ entity.position }} ({{ entity.location }})</title>
      <link href="{{ url('ens_job_show', { 'id': entity.id, 'company': entity.companyslug, 'location': entity.locationslug, 'position': entity.positionslug }) }}" />
      <id>{{ entity.id }}</id>
      <updated>{{ entity.createdAt.format(constant('DATE_ATOM')) }}</updated>
      <summary type="xhtml">
        <div xmlns="http://www.w3.org/1999/xhtml">
          {% if entity.logo %}
            <div>
              <a href="{{ entity.url }}">
                <img src="http://{{ app.request.host }}/uploads/jobs/{{ entity.logo }}" alt="{{ entity.company }} logo" />
              </a>
            </div>
          {% endif %}
          <div>
            {{ entity.description|nl2br }}
          </div>
          <h4>How to apply?</h4>
          <p>{{ entity.howtoapply }}</p>
        </div>
      </summary>
      <author><name>{{ entity.company }}</name></author>
    </entry>
  {% endfor %}
{% endfor %}
 
<!-- ... -->

```

### 分类下的最新职位订阅

Jobeet 的目标之一是帮助人们找到更多有针对性的就业岗位。所以我们需要为每个分类提供订阅。

首先，让我们更新在模板中的分类订阅链接 ︰

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->
<div class="feed">
  <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, '_format': 'atom' }) }}">Feed</a>
</div>
 
<!-- src/Ens/JobeetBundle/Resources/views/Category/show.html.twig -->
<div class="feed">
  <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, '_format': 'atom' }) }}">Feed</a>
</div>
```

更新 CategoryController的showAction 来呈现相应的模板 ︰

```php
// src/Ens/JobeetBundle/Controller/CategoryController.php
// ...

$latestJob = $em->getRepository('EnsJobeetBundle:Job')->getLatestPost($category->getId());
 
if($latestJob) {
    $lastUpdated = $latestJob->getCreatedAt()->format(DATE_ATOM); 
} else {
    $lastUpdated = new \DateTime();
    $lastUpdated = $lastUpdated->format(DATE_ATOM);
}
 
$format = $this->getRequest()->getRequestFormat();
 
return $this->render('EnsJobeetBundle:Category:show.'.$format.'.twig', array(
    'category' => $category,
    'last_page' => $last_page,
    'previous_page' => $previous_page,
    'current_page' => $page,
    'next_page' => $next_page,
    'total_jobs' => $total_jobs,
    'feedId' => sha1($this->get('router')->generate('EnsJobeetBundle_category', array('slug' =>  $category->getSlug(), '_format' => 'atom'), true)),
    'lastUpdated'=>$lastUpdated,
));
 
// ...
```

最后，我们创建 show.atom.twig 模板 ︰

```xml
<!-- src/Ens/JobeetBundle/Resources/views/Category/show.atom.twig -->
<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Jobeet ({{ category.name }})</title>
  <subtitle>Latest Jobs</subtitle>
  <link href="{{ url('EnsJobeetBundle_category', { 'slug': category.slug, '_format': 'atom' }) }}" rel="self" />
  <link href="{{ url('EnsJobeetBundle_category', { 'slug': category.slug }) }}" />
  <updated>{{ category.activejobs[0].createdAt.format(constant('DATE_ATOM')) }}</updated>
  <author><name>Jobeet</name></author>
  <id>{{ feedId }}</id>
 
  {% for entity in category.activejobs %}
    <entry>
      <title>{{ entity.position }} ({{ entity.location }})</title>
      <link href="{{ url('ens_job_show', { 'id': entity.id, 'company': entity.companyslug, 'location': entity.locationslug, 'position': entity.positionslug }) }}" />
      <id>{{ entity.id }}</id>
      <updated>{{ entity.createdAt.format(constant('DATE_ATOM')) }}</updated>
      <summary type="xhtml">
        <div xmlns="http://www.w3.org/1999/xhtml">
          {% if entity.logo %}
            <div>
              <a href="{{ entity.url }}">
                <img src="http://{{ app.request.host }}/uploads/jobs/{{ entity.logo }}" alt="{{ entity.company }} logo" />
              </a>
            </div>
          {% endif %}
          <div>
            {{ entity.description|nl2br }}
          </div>
          <h4>How to apply?</h4>
          <p>{{ entity.howtoapply }}</p>
        </div>
      </summary>
      <author><name>{{ entity.company }}</name></author>
    </entry>
  {% endfor %}
</feed>
```