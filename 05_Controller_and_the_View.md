# Symfony2 Jobeet Day 4: 控制器与视图
今天，我们要继续完成昨天创建的基础job控制器。它已经拥有了Jobeet所需的大部分的代码：

* 所有职位的列表
* 创建一个新的职位
* 修改职位的页面
* 删除一个职位

虽然代码已经可以使用,但我们还需要在此基础上重构模板，使得它更匹配Jobeet模型。

## MVC架构

对于Web开发，组织你代码的方式有很多，但其中最为常见的解决方案就是[MVC设计模式](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)。简而言之, MVC设计模式是定义了一种最自然的组织代码的方式。
这种模式将代码分为**三个层次**：
**Model层**定义了业务逻辑（数据库属于这一层）。通过前一天的实战，你已经知道在Symfony2中存储所有和model相关的类和文件都在Entity/文件夹下。
**View层**是用于与用户进行交互的图形界面（模板引擎是该层的一部分）。在Symfony2中,视图层主要使用Twig模板。它们存储在Resources/views/文件夹下，我们会在稍后看到。
**Controller**是一段调用Model层代码，以此来获得一些数据,并传递给View层进而呈现给客户端。当我们在安装Symfony开始本教程时，我们可以看到所有的请求都是由前端控制器（app.php和app_dev.php）进行管理的。这些前端控制器将实际工作委派给具体的**action**。

## 页面布局

如果你有仔细观察设计,就会注意到的每个页面看起来都很相似。众所周知，重复的代码是一种不好的设计，无论是html还是php。所以我们需要找到一种方式来防止这些公用元素所造成的重复代码。

解决这个问题的一个方法是定义一个页眉和页脚,并将它们包含在每个模板里面。另一个更好的方法是使用设计模式来解决这个问题:[装饰器模式](http://en.wikipedia.org/wiki/Decorator_pattern)。它可以将渲染好的模板置于一个全局模板中输出，该全局模板称之为**布局**

与Symfony 1.x不同，Symfony2中并没有一个默认模板，因此我们将先创建一个，并用它来装饰我们的应用程序页面。

创建一个新的`layout.html.twig`文件在`src/Ens/JobeetBundle/Resources/views/`目录下

代码：
```html
<!DOCTYPE html>
<html>
  <head>
    <title>
      {% block title %}
        Jobeet - Your best job board
      {% endblock %}
    </title>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    {% block stylesheets %}
      <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/main.css') }}" type="text/css" media="all" />
    {% endblock %}
    {% block javascripts %}
    {% endblock %}
    <link rel="shortcut icon" href="{{ asset('bundles/ensjobeet/images/favicon.ico') }}" />
  </head>
  <body>
    <div id="container">
      <div id="header">
        <div class="content">
          <h1><a href="{{ path('ens_job') }}">
            <img src="{{ asset('bundles/ensjobeet/images/logo.jpg') }}" alt="Jobeet Job Board" />
          </a></h1>
 
          <div id="sub_header">
            <div class="post">
              <h2>Ask for people</h2>
              <div>
                <a href="{{ path('ens_job') }}">Post a Job</a>
              </div>
            </div>
 
            <div class="search">
              <h2>Ask for a job</h2>
              <form action="" method="get">
                <input type="text" name="keywords" id="search_keywords" />
                <input type="submit" value="search" />
                <div class="help">
                  Enter some keywords (city, country, position, ...)
                </div>
              </form>
            </div>
          </div>
        </div>
      </div>
 
      <div id="content">
        {% if app.session.hasFlash('notice') %}
          <div class="flash_notice">
            {{ app.session.flash('notice') }}
          </div>
        {% endif %}
 
        {% if app.session.hasFlash('error') %}
          <div class="flash_error">
            {{ app.session.flash('error') }}
          </div>
        {% endif %}
 
        <div class="content">
            {% block content %}
            {% endblock %}
        </div>
      </div>
 
      <div id="footer">
        <div class="content">
          <span class="symfony">
            <img src="{{ asset('bundles/ensjobeet/images/jobeet-mini.png') }}" />
            powered by <a href="http://www.symfony.com/">
              <img src="{{ asset('bundles/ensjobeet/images/symfony.gif') }}" alt="symfony framework" />
            </a>
          </span>
          <ul>
            <li><a href="">About Jobeet</a></li>
            <li class="feed"><a href="">Full feed</a></li>
            <li><a href="">Jobeet API</a></li>
            <li class="last"><a href="">Affiliates</a></li>
          </ul>
        </div>
      </div>
    </div>
  </body>
</html>
```

## Twig Blocks
Twig是Symfony2默认的模板引擎。你能定义blocks像我们上面所展示的那样，一个Twig block可以有一个默认的内容（例如上面的title block）也能使用子模板替换或扩展（稍后将会看到）。

现在，利用我们创建的新布局编辑所有的job模板（在`src/En/JobeetBundle/Resources/views/Job/`目录下的edit, index, new 和show模板），并重写content block

```html
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% block content %}
  <!-- original template code goes here -->
{% endblock %}
```

## 样式表、图片与javascript
由于本教程不是关于网页设计的，所以在此我们已经准备好本项目所需的资源
下载[图片文件](http://www.ens.ro/downloads/jobeet-images.zip)并解压到`src/Ens/JobeetBundle/Resources/public/images/`目录下
下载[样式表文件](http://www.ens.ro/downloads/jobeet-css.zip)并解压到`src/Ens/JobeetBundle/Resources/public/css/`目录下

然后运行：

> **php app/console assets:install web**

以此通知Symfony这些资源是公共的。

> *译者注：此处如果是Linux下实际上是在web/bundles/下生成一个项目名的软连接，而Windows下则是直接将Resources下的所有文件直接复制过来，所以Windows下如果在Resources下更新文件之后必须记得再次执行上述命令进行同步*

我们有4个css文件admin.css, job.css, jobs.css 和main.css。其中main.css需要在所有的页面中都引用，其他css文件我们只需要在特定文件中引用即可。

在模板中添加一个新的css文件需要重写stylesheets block，但加入新的css文件之前调用父模板方法

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/jobs.css') }}" type="text/css" media="all" />
{% endblock %}
<!-- the rest of the code -->

<!-- src/Ens/JobeetBundle/Resources/views/Job/show.html.twig -->
{% extends 'EnsJobeetBundle::layout.html.twig' %}
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/job.css') }}" type="text/css" media="all" />
{% endblock %}
<!-- the rest of the code -->
```

## 首页Action

每个action对应于类中的一个方法，对于主页来说就是JobController中的indexAction()方法。它从数据库中检索所有的职位：

```php
public function indexAction()
{
    $em = $this->getDoctrine()->getManager();
 
    $entities = $em->getRepository('EnsJobeetBundle:Job')->findAll();
 
    return $this->render('EnsJobeetBundle:Job:index.html.twig', array(
        'entities' => $entities
    ));
}
```

> *译者注：原文中使用的getEntityManager方法已被废弃，现改为getManager方法效果一样。但因为原文使用地方较多，故代码已直接修改，以后文章出现亦同*

现在让我们仔细观察代码，首先在indexAction()方法中得到**Doctrine entity manager**对象，它会负责把数据持久化到数据库或者把数据从数据库中取出，然后repository对象生成一个查询并去数据库检索所有的职位信息，同时返回一个Doctrine ArrayCollection类型的Job对象并传递给模板(视图)

## Job主页模板
index.html.twig模板生成的所有职位的HTML表格。下面是当前模板的代码

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/jobs.css') }}" type="text/css" media="all" />
{% endblock %}
 
{% block content %}
    <h1>Job list</h1>
 
    <table class="records_list">
        <thead>
            <tr>
                <th>Id</th>
                <th>Type</th>
                <th>Company</th>
                <th>Logo</th>
                <th>Url</th>
                <th>Position</th>
                <th>Location</th>
                <th>Description</th>
                <th>How_to_apply</th>
                <th>Token</th>
                <th>Is_public</th>
                <th>Is_activated</th>
                <th>Email</th>
                <th>Expires_at</th>
                <th>Created_at</th>
                <th>Updated_at</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
        {% for entity in entities %}
            <tr>
                <td><a href="{{ path('ens_job_show', { 'id': entity.id }) }}">{{ entity.id }}</a></td>
                <td>{{ entity.type }}</td>
                <td>{{ entity.company }}</td>
                <td>{{ entity.logo }}</td>
                <td>{{ entity.url }}</td>
                <td>{{ entity.position }}</td>
                <td>{{ entity.location }}</td>
                <td>{{ entity.description }}</td>
                <td>{{ entity.howtoapply }}</td>
                <td>{{ entity.token }}</td>
                <td>{{ entity.ispublic }}</td>
                <td>{{ entity.isactivated }}</td>
                <td>{{ entity.email }}</td>
                <td>{% if entity.expiresat %}{{ entity.expiresat|date('Y-m-d H:i:s') }}{% endif%}</td>
                <td>{% if entity.createdat %}{{ entity.createdat|date('Y-m-d H:i:s') }}{% endif%}</td>
                <td>{% if entity.updatedat %}{{ entity.updatedat|date('Y-m-d H:i:s') }}{% endif%}</td>
                <td>
                    <ul>
                        <li>
                            <a href="{{ path('ens_job_show', { 'id': entity.id }) }}">show</a>
                        </li>
                        <li>
                            <a href="{{ path('ens_job_edit', { 'id': entity.id }) }}">edit</a>
                        </li>
                    </ul>
                </td>
            </tr>
        {% endfor %}
        </tbody>
    </table>
 
    <ul>
        <li>
            <a href="{{ path('ens_job_new') }}">
                Create a new entry
            </a>
        </li>
    </ul>
{% endblock %}
```


现在让我们清理一下，只显示有用的列，用如下代码替换掉twig block content里的内容

```html
{% block content %}
    <div id="jobs">
      <table class="jobs">
        {% for entity in entities %}
          <tr class="{{ cycle(['even', 'odd'], loop.index) }}">
            <td class="location">{{ entity.location }}</td>
            <td class="position">
              <a href="{{ path('ens_job_show', { 'id': entity.id }) }}">
                {{ entity.position }}
              </a>
            </td>
            <td class="company">{{ entity.company }}</td>
          </tr>
        {% endfor %}
      </table>
    </div>
{% endblock %}
```
![首页](http://www.ens.ro/wp-content/uploads/2012/03/Screenshot-at-2012-03-30-232147-1024x671.png) 

## 职位详情页模板
现在开始定制职位详情页模板，打开show.html.twig文件并将下方内容进行替换：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/show.html.twig -->
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% block title %}
    {{ entity.company }} is looking for a {{ entity.position }}
{% endblock %}
 
{% block stylesheets %}
    {{ parent() }}
    <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/job.css') }}" type="text/css" media="all" />
{% endblock %}
 
{% block content %}
    <div id="job">
      <h1>{{ entity.company }}</h1>
      <h2>{{ entity.location }}</h2>
      <h3>
        {{ entity.position }}
        <small> - {{ entity.type }}</small>
      </h3>
 
      {% if entity.logo %}
        <div class="logo">
          <a href="{{ entity.url }}">
            <img src="/uploads/jobs/{{ entity.logo }}"
              alt="{{ entity.company }} logo" />
          </a>
        </div>
      {% endif %}
 
      <div class="description">
        {{ entity.description|nl2br }}
      </div>
 
      <h4>How to apply?</h4>
 
      <p class="how_to_apply">{{ entity.howtoapply }}</p>
 
      <div class="meta">
        <small>posted on {{ entity.createdat|date('m/d/Y') }}</small>
      </div>
 
      <div style="padding: 20px 0">
        <a href="{{ path('ens_job_edit', { 'id': entity.id }) }}">
          Edit
        </a>
      </div>
    </div>
{% endblock %}
```

![详情页](http://www.ens.ro/wp-content/uploads/2012/03/Screenshot-at-2012-03-30-232158-1024x671.png) 


## 职位详情页Action
在JobController中创建showAction()方法

```php
public function showAction($id)
{
    $em = $this->getDoctrine()->getManager();
    $entity = $em->getRepository('EnsJobeetBundle:Job')->find($id);
 
    if (!$entity) {
        throw $this->createNotFoundException('Unable to find Job entity.');
    }
    $deleteForm = $this->createDeleteForm($id);
 
    return $this->render('EnsJobeetBundle:Job:show.html.twig', array(
        'entity'      => $entity,
        'delete_form' => $deleteForm->createView(),
    ));
}
```

在action中这次我们使用EnsJobeetBundle:Job repository的find()方法，该方法接收一个能唯一标识Job表的参数，一般是主键值，此处为职位id
如果寻找的职位详情不存在于数据库中时，将会调用
$this->createNotFoundException()方法抛出一个404页面
不过值得注意的是，在生产环境和开发环境显示给用户的404页面是的不同：
生产环境：
![生产环境](http://www.ens.ro/wp-content/uploads/2012/03/Screenshot-at-2012-03-30-231728-1024x671.png)

开发环境：
![开发环境](http://www.ens.ro/wp-content/uploads/2012/03/Screenshot-at-2012-03-30-231718-1024x671.png) 

译者注：关于生产环境中如何使用自定义错误页面，可参考[官方cookbook](http://symfony.com/doc/2.3/cookbook/controller/error_pages.html)
