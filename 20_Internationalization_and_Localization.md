# Symfony2 Jobeet Day 19: 国际化和本地化

昨天我们通过AJAX方法完善了搜索引擎并使得其更有趣。今天我们需要聊聊Jobeet的国际化(i18n)和本地化(l10n)

> *来自维基百科：*
> 
> ***国际化**是指设计软件应用程序中的一个过程，以便它可以适应不同的语言和地区而不需要更改工程。*
>  
> ***本地化**是指适应特定地区或语言的软件通过添加特定于区域设置组件和翻译文本的过程。*

## 用户

没有国际化就意味着可能没有用户。当你的网站提供多种语言或在世界不同地区使用时，应当使用户选择哪个语言最适合他。

Symfony的i18n和l10n特性是基于用户文化的。用户文化是结合了他们各自的语言和国籍组成的。例如，对一个说法语的用户来说他的用户文化是`fr`，而如果他又恰巧来自法国，那么他的用户文化就是`fr_FR`。


> *译者注：一如既往的绕口(-__-)b*

Symfony中翻译是通过Translator服务来处理的，它使用用户的区域设置来查找并返回翻译后的信息。在使用之前，我们需要在配置文件中开启它：

```yaml
#app/config/config.yml

#...

framework:
    #esi:             ~
    translator:      { fallback: en }
    # ...
    default_locale:  "en"

#...
```

## URL中的Culture

由于Jobeet网站将提供英文和法文两种语言支持，但一个URL仅能表示一个资源，因此我们需要将culture嵌入URL中。为了做到这一点，我们打开routing.yml文件，并添加一个特殊的locale变量在除了api之外的所有路由。为了使路由更简单，我们在url的前面加上/{_locale}

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml

login:
    pattern: /login
    defaults: { _controller: EnsJobeetBundle:Default:login }

login_check:
    pattern: /login_check

logout:
  pattern: /logout

EnsJobeetBundle_category:
    pattern:  /{_locale}/category/{slug}/{page}
    defaults: { _controller: EnsJobeetBundle:Category:show, page: 1 }   
    requirements:
        _locale: en|fr

EnsJobeetBundle_job:
    resource: "@EnsJobeetBundle/Resources/config/routing/job.yml"
    prefix:   /{_locale}/job
    requirements: 
        _locale: en|fr

ens_jobeet_homepage:
    pattern:  /
    defaults: { _controller: EnsJobeetBundle:Job:index } 

EnsJobeetBundle_api:
    pattern: /api/{token}/jobs.{_format}
    defaults: {_controller: "EnsJobeetBundle:Api:list"}
    requirements:
        _format: xml|json|yaml

EnsJobeetBundle_ens_affiliate:
    resource: "@EnsJobeetBundle/Resources/config/routing/affiliate.yml"
    prefix:   /{_locale}/affiliate       
    requirements: 
        _locale: en|fr
```

我们的主页需要支持尽可能多的语言 (/en/, /fr/, …),因此默认的主页(/)应该是根据不同的用户文化来跳转到特定的主页。但如果用户没有culture怎么办呢？比如他第一次登录我们的Jobeet网站，那么将为他首选一个默认语言(如我们之前所定义的，将会是en)。so，让我们回到前面的文件，添加一个新的路由来做这个事儿，并修改你的主页路由：

```yaml
#src/Ens/Resources/config/routing.ymlYAML

#...
ens_jobeet_homepage:
    pattern:  /{_locale}/
    defaults: { _controller: EnsJobeetBundle:Job:index }
    requirements: 
        _locale: en|fr
#...

EnsJobeetBundle_nonlocalized:
    pattern:  /
    defaults: { _controller: "EnsJobeetBundle:Job:index" }
```

让我们在控制器中添加和修改这两个路由的行为：

```php
// src/Ens/JobeetBundle/Controller/JobController.php

// ...

    public function indexAction()
    {
        $request = $this->getRequest();

        if($request->get('_route') == 'EnsJobeetBundle_nonlocalized') {
            return $this->redirect($this->generateUrl('ens_jobeet_homepage'));
        }

        $em = $this->getDoctrine()->getManager();

        // ...
    }

// ...
```

如果用户访问Jobeet平台没有指定他的首选语言的话(`http://jobeet.local/app_dev.php`)，他将被重定向到默认主页(`http://jobeet.local/app_dev.php/en/`)。

## Culture 测试

~~午时已到，乖乖站好！~~不过在我们添加测试之前还需要对原先的测试案例进行修正。由于我们更改了所有的URL，所以现在需要在所有功能测试的URL之前添加/en参数。

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php

namespace Ens\JobeetBundle\Tests\Controller;
 
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;
use Symfony\Component\DomCrawler\Crawler;
 
class JobControllerTest extends WebTestCase
{
	//...
 
    public function testIndex()
    {
        // get the custom parameters from app config.yml
        $kernel = static::createKernel();
        $kernel->boot();
        $max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');
 
        $client = static::createClient();
 
        $crawler = $client->request('GET', '/fr/');
        $this->assertEquals('Ens\JobeetBundle\Controller\JobController::indexAction', $client->getRequest()->attributes->get('_controller'));
 
        // If the selected culture is italian, the page requested will not be found
        $crawler = $client->request('GET', '/it/');
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());

        $crawler = $client->request('GET', '/en/');
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
        $crawler = $client->request('GET', '/en/job/foo-inc/milano-italy/0/painter');
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
 
        // an expired job page forwards the user to a 404
        $crawler = $client->request('GET', sprintf('/en/job/sensio-labs/paris-france/%d/web-developer', $this->getExpiredJob()->getId()));
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
    }

    public function testJobForm()
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/en/job/new');

        $this->assertEquals('Ens\JobeetBundle\Controller\JobController::newAction', $client->getRequest()->attributes->get('_controller'));

        $form = $crawler->selectButton('Preview your job')->form(array(
            'job[company]'      => 'Sensio Labs',
            'job[url]'          => 'http://www.sensio.com',
            'job[file]'         => __DIR__.'/../../../../../web/bundles/ensjobeet/images/sensio-labs.gif',
            'job[how_to_apply]' => 'Send me an email',
            'job[description]'  => 'You will work with symfony to develop websites for our customers',
            'job[location]'     => 'Atlanta, USA',
            'job[email]'        => 'for.a.job@example.com',
            'job[position]'     => 'Developer',
            'job[is_public]'    => false,
        ));

        $client->submit($form);
        $this->assertEquals('Ens\JobeetBundle\Controller\JobController::createAction', $client->getRequest()->attributes->get('_controller'));

        $client->followRedirect();
        $this->assertEquals('Ens\JobeetBundle\Controller\JobController::previewAction', $client->getRequest()->attributes->get('_controller'));

        $kernel = static::createKernel();
        $kernel->boot();
        $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');

        $query = $em->createQuery('SELECT count(j.id) from EnsJobeetBundle:Job j WHERE j.location = :location AND j.is_activated IS NULL AND j.is_public = 0');
        $query->setParameter('location', 'Atlanta, USA');
        $this->assertTrue(0 < $query->getSingleScalarResult());

        $crawler = $client->request('GET', '/en/job/new');
        $form = $crawler->selectButton('Preview your job')->form(array(
            'job[company]'      => 'Sensio Labs',
            'job[position]'     => 'Developer',
            'job[location]'     => 'Atlanta, USA',
            'job[email]'        => 'not.an.email',
        ));
        $crawler = $client->submit($form);

        // check if we have 3 errors
        $this->assertTrue($crawler->filter('.error_list')->count() == 3);
        // check if we have error on job_description field
        $this->assertTrue($crawler->filter('#job_description')->siblings()->first()->filter('.error_list')->count() == 1);
        // check if we have error on job_how_to_apply field
        $this->assertTrue($crawler->filter('#job_how_to_apply')->siblings()->first()->filter('.error_list')->count() == 1);
        // check if we have error on job_email field
        $this->assertTrue($crawler->filter('#job_email')->siblings()->first()->filter('.error_list')->count() == 1);
    }

    public function createJob($values = array(), $publish = false)
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/en/job/new');
        $form = $crawler->selectButton('Preview your job')->form(array_merge(array(
            'job[company]'      => 'Sensio Labs',
            'job[url]'          => 'http://www.sensio.com/',
            'job[position]'     => 'Developer',
            'job[location]'     => 'Atlanta, USA',
            'job[description]'  => 'You will work with symfony to develop websites for our customers.',
            'job[how_to_apply]' => 'Send me an email',
            'job[email]'        => 'for.a.job@example.com',
            'job[is_public]'    => false,
        ), $values));

        $client->submit($form);
        $client->followRedirect();

        if($publish) {
            $crawler = $client->getCrawler();
            $form = $crawler->selectButton('Publish')->form();
            $client->submit($form);
            $client->followRedirect();
        }

        return $client;
    }

    // ...

    public function testEditJob()
    {
        $client = $this->createJob(array('job[position]' => 'FOO3'), true);
        $crawler = $client->getCrawler();
        $crawler = $client->request('GET', sprintf('/en/job/%s/edit', $this->getJobByPosition('FOO3')->getToken()));
        $this->assertTrue( 404 === $client->getResponse()->getStatusCode());
    }

    public function testExtendJob()
    {
        // A job validity cannot be extended before the job expires soon
        $client = $this->createJob(array('job[position]' => 'FOO4'), true);
        $crawler = $client->getCrawler();
        $this->assertTrue($crawler->filter('input[type=submit]:contains("Extend")')->count() == 0);

        // A job validity can be extended hen the job expires soon
        // Create a new FOO5 job
        $client = $this->createJob(array('job[position]' => 'FOO5'), true);
        // Get the job and change the expire date to today
        $kernel = static::createKernel();
        $kernel->boot();
        $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
        $job = $em->getRepository('EnsJobeetBundle:Job')->findOneByPosition('FOO5');
        $job->setExpiresAt(new \DateTime());
        $em->flush();

        // Go to preview page and extend the job
        $crawler = $client->request('GET', sprintf('/en/job/%s/%s/%s/%s', $job->getCompanySlug(), $job->getLocationSlug(), $job->getToken(), $job->getPositionSlug()));
        $crawler = $client->getCrawler();

        $form = $crawler->selectButton('Extend')->form();
        $client->submit($form);
        $client->followRedirect();
        $this->assertEquals('Ens\JobeetBundle\Controller\JobController::previewAction', $client->getRequest()->attributes->get('_controller'));

        // Reload the job from database
        $job = $this->getJobByPosition('FOO5');

        // Check the expiration date
        $this->assertTrue($job->getExpiresAt()->format('y/m/d') == date('y/m/d', time() + 86400 * 30));
    }

    public function testSearch()
    {
        $client = static::createClient();

        $crawler = $client->request('GET', '/en/job/search');
        $this->assertEquals('Ens\JobeetBundle\Controller\JobController::searchAction', $client->getRequest()->attributes->get('_controller'));

        $crawler = $client->request('GET', '/en/job/search?query=sens*', array(), array(), array(
            'X-Requested-With' => 'XMLHttpRequest',
        ));
        $this->assertTrue($crawler->filter('tr')->count()== 2);
    }
}

```

```php
// src/Ens/JobeetBundle/Tests/Controller/AffiliateControllerTest.php

// ...     

    public function testAffiliateForm()
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/en/affiliate/new');

        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::newAction', $client->getRequest()->attributes->get('_controller'));

        $form = $crawler->selectButton('Submit')->form(array(
            'affiliate[url]'   => 'http://sensio-labs.com/',
            'affiliate[email]' => 'fabien.potencier@example.com'
        ));

        $client->submit($form);
        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::createAction', $client->getRequest()->attributes->get('_controller'));

        $kernel = static::createKernel();
        $kernel->boot();
        $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');

        $crawler = $client->request('GET', '/en/affiliate/new');
        $form = $crawler->selectButton('Submit')->form(array(
            'affiliate[email]'        => 'not.an.email',
        ));
        $crawler = $client->submit($form);

        // check if we have 1 errors
        $this->assertTrue($crawler->filter('.error_list')->count() == 1);
        // check if we have error on affiliate_email field
        $this->assertTrue($crawler->filter('#affiliate_email')->siblings()->first()->filter('.error_list')->count() == 1);
    }

    public function testCreate()
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/en/affiliate/new');
        $form = $crawler->selectButton('Submit')->form(array(
            'affiliate[url]'   => 'http://sensio-labs.com/',
            'affiliate[email]' => 'address@example.com'
        ));

        $client->submit($form);
        $client->followRedirect();

        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::waitAction', $client->getRequest()->attributes->get('_controller'));

        return $client;
    }

    public function testWait()
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/en/affiliate/wait');

        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::waitAction', $client->getRequest()->attributes->get('_controller'));
    }

    // ...
```

```php
// src/Ens/JobeetBundle/Test/Controller/CategoryControllerTest.php

// ...

    public function testShow()
    {
        $kernel = static::createKernel();
        $kernel->boot();

        // get the custom parameters from app/config.yml
        $max_jobs_on_category = $kernel->getContainer()->getParameter('max_jobs_on_category');
        $max_jobs_on_homepage = $kernel->getContainer()->getParameter('max_jobs_on_homepage');

        $client = static::createClient();

        $categories = $this->em->getRepository('EnsJobeetBundle:Category')->getWithJobs();

        // categories on homepage are clickable
        foreach($categories as $category) {
            $crawler = $client->request('GET', '/en/');

            $link = $crawler->selectLink($category->getName())->link();
            $crawler = $client->click($link);

            $this->assertEquals('Ens\JobeetBundle\Controller\CategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
            $this->assertEquals($category->getSlug(), $client->getRequest()->attributes->get('slug'));

            $jobs_no = $this->em->getRepository('EnsJobeetBundle:Job')->countActiveJobs($category->getId()); 

            // categories with more than $max_jobs_on_homepage jobs also have a "more" link                 
            if($jobs_no > $max_jobs_on_homepage) {
                $crawler = $client->request('GET', '/en/');
                $link = $crawler->filter(".category_" . $category->getSlug() . " .more_jobs a")->link();
                $crawler = $client->click($link);

                $this->assertEquals('Ens\JobeetBundle\Controller\CategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
                $this->assertEquals($category->getSlug(), $client->getRequest()->attributes->get('slug'));
            }

            $pages = ceil($jobs_no/$max_jobs_on_category);

            // only $max_jobs_on_category jobs are listed 
            $this->assertTrue($crawler->filter('.jobs tr')->count() <= $max_jobs_on_category);
            $this->assertRegExp("/" . $jobs_no . " jobs/", $crawler->filter('.pagination_desc')->text());

            if($pages > 1) {
                $this->assertRegExp("/page 1\/" . $pages . "/", $crawler->filter('.pagination_desc')->text());

                for ($i = 2; $i <= $pages; $i++) {
                    $link = $crawler->selectLink($i)->link();
                    $crawler = $client->click($link);

                    $this->assertEquals('Ens\JobeetBundle\Controller\CategoryController::showAction', $client->getRequest()->attributes->get('_controller'));
                    $this->assertEquals($i, $client->getRequest()->attributes->get('page'));
                    $this->assertTrue($crawler->filter('.jobs tr')->count() <= $max_jobs_on_category);
                    if($jobs_no > 1) {
                        $this->assertRegExp("/" . $jobs_no . " jobs/", $crawler->filter('.pagination_desc')->text());
                    }
                    $this->assertRegExp("/page " . $i . "\/" . $pages . "/", $crawler->filter('.pagination_desc')->text());
                }
            }     
        }
    }

    // ...

```

## 切换语言

为了用户能更改他的语言，我们需要在`layout`中添加一个表单：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->

<!-- ... -->
<div id="footer">
    <div class="content">
        <!-- ... -->
        <form action="{{ path('EnsJobeetBundle_changeLanguage') }}" method="get">
            <label>Language</label>
                <select name="language">
                    <option value="en" {% if app.request.get('_locale') == 'en' %}selected="selected"{% endif %}>English</option>
                    <option value="fr" {% if app.request.get('_locale') == 'fr' %}selected="selected"{% endif %}>French</option>
                </select>
            <input type="submit" value="Ok"> 
        </form>
    </div>
</div>
<!-- ... -->
```

添加一个新的路由为我们更改语言时匹配action：

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml

#...

EnsJobeetBundle_changeLanguage:
    pattern: /change_language
    defaults: { _controller: "EnsJobeetBundle:Default:changeLanguage" }
```

添加新的action：

```php
src/Ens/JobetBundle/Controller/DefaultController.php

namespace Ens\JobeetBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\Security\Core\SecurityContext;

class DefaultController extends Controller
{
    // ...

    public function changeLanguageAction()
    {
        $language = $this->getRequest()->get('language');
        return $this->redirect($this->generateUrl('ens_jobeet_homepage', array('_locale' => $language)));
    }
}
```

完成之后记得清除缓存哦~

## 模板

一个国际化的网站意味着用户界面可以被翻译成多种语言。对我们来说就是英语和法语两种。为了翻译模板，我们将使用Twig的{% trans %}标签。当Symfony渲染模板时，每次{% trans %}标签被找到，Symfony会查找当前用户的区域性设置进行翻译并返回。如果找到了翻译，就使用它，否则将原样返回。

所有翻译文件都存放在`src/Ens/JobeetBundle/Resources/translations/`目录下。我们将使用XLIFF格式来保存它们，这是一种标准而又灵活的格式。

首先让我们在模板中添加{% trans %}标签：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->

<!DOCTYPE html>
<html>
    <head>
        <title>
            {% block title %}
                {% trans %}Jobeet - Your best job board{% endtrans %}
            {% endblock %}
        </title>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        {% block stylesheets %}
            <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/main.css') }}" type="text/css" media="all" />
            <link rel="alternate" type="application/atom+xml" title="Latest Jobs" href="{{ url('ens_job', {'_format': 'atom'}) }}" />
        {% endblock %}
        {% block javascripts %}
            <script type="text/javascript" src="{{ asset('bundles/ensjobeet/js/jquery-2.0.3.min.js') }}"></script>
            <script type="text/javascript" src="{{ asset('bundles/ensjobeet/js/search.js') }}"></script>
        {% endblock %}
        <link rel="shortcut icon" href="{{ asset('bundles/ensjobeet/images/favicon.ico') }}" />
    </head>
    <body>
        <div id="container">
            <div id="header">
                <div class="content">
                    <h1><a href="{{ path('ens_jobeet_homepage') }}">
                        <img alt="Jobeet Job Board" src="{{ asset('bundles/ensjobeet/images/logo.jpg') }}" />
                    </a></h1>

                    <div id="sub_header">
                        <div class="post">
                            <h2>{% trans %}Ask for people{% endtrans %}</h2>
                            <div>
                                <a href="{{ path('ens_job_new') }}">{% trans %}Post a Job{% endtrans %}</a>
                            </div>
                        </div>

                        <div class="search">
                            <h2>{% trans %}Ask for a job{% endtrans %}</h2>
                            <form action="{{ path('ens_job_search') }}" method="get">
                                <input type="text" name="query" value="{{ app.request.get('query') }}" id="search_keywords" />
                                <input type="submit" value="search" />
                                <img id="loader" src="{{ asset('bundles/ensjobeet/images/loader.gif') }}" style="vertical-align: middle; display: none" />
                                <div class="help">
                                    {% trans %}Enter some keywords (city, country, position, ...){% endtrans %}
                                </div>
                            </form>
                        </div>
                    </div>
                </div>
            </div>
           <div id="job_history">
                {% trans %}Recent viewed jobs:{% endtrans %}
                <ul>
                    {% for job in app.session.get('job_history') %}
                        <li>
                            <a href="{{ path('ens_job_show', { 'id': job.id, 'company': job.companyslug, 'location': job.locationslug, 'position': job.positionslug }) }}">{{ job.position }} - {{ job.company }}</a>
                        </li>
                    {% endfor %}
                </ul>
            </div>
           <div id="content">
               {% for flashMessage in app.session.flashbag.get('notice') %}
                   <div class="flash_notice">
                       {{ flashMessage }}
                   </div>
               {% endfor %}

               {% for flashMessage in app.session.flashbag.get('error') %}
                   <div class="flash_error">
                       {{ flashMessage }}
                   </div>
               {% endfor %}

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
                       <li><a href="">{% trans %}About Jobeet{% endtrans %}</a></li>
                       <li class="feed"><a href="{{ path('ens_job', {'_format': 'atom'}) }}">{% trans %}Full feed{% endtrans %}</a></li>
                       <li><a href="">{% trans %}Jobeet API{% endtrans %}</a></li>
                       <li class="last"><a href="{{ path('ens_affiliate_new') }}">{% trans %}Become an affiliate{% endtrans %}</a></li>
                   </ul>
                   <form action="{{ path('EnsJobeetBundle_changeLanguage') }}" method="get">
                       <label>{% trans %}Language{% endtrans %}</label>
                       <select name="language">
                           <option value="en" {% if app.request.get('_locale') == 'en' %}selected="selected"{% endif %}>English</option>
                                <option value="fr" {% if app.request.get('_locale') == 'fr' %}selected="selected"{% endif %}>French</option>
                       </select>
                       <input type="submit" value="Ok"> 
                   </form>
               </div>
           </div>
       </div>
   </body>
</html>

```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/show.html.twig -->

{% extends 'EnsJobeetBundle::layout.html.twig' %}

{% block title %}
    {% trans with {'%company%': entity.company, '%position%': entity.position} %}%company% is looking for a %position%{% endtrans %}
{% endblock %}

{% block stylesheets %}
    {{ parent() }}
    <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/job.css') }}" type="text/css" media="all" />
{% endblock %}

{% block content %}
    {% if app.request.get('token') %}
        {% include 'EnsJobeetBundle:Job:admin.html.twig' with {'job': entity} %}
    {% endif %}
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

        <h4>{% trans %}How to apply?{% endtrans %}</h4>

        <p class="how_to_apply">{{ entity.howtoapply }}</p>

        <div class="meta">
            <small>{% trans with {'%date%': entity.createdat|date('m/d/Y')} %}posted on %date%{% endtrans %}</small>
        </div>
    </div>
{% endblock %}

```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/new.html.twig -->

<!-- ... -->
{% block content %}
    <h1>{% trans %}Job creation{% endtrans %}</h1>
    <!-- ... -->
        <br /> {% trans %}Whether the job can also be published on affiliate websites or not.{% endtrans %}
    <!-- ... -->
<!-- ... -->
```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/index.html.twig -->

<!-- ... -->
    {% if category.morejobs %}
        <div class="more_jobs">
            {% trans with {'%count%': '<a href="' ~ path('EnsJobeetBundle_category', { 'slug': category.slug }) ~ '">' ~  category.morejobs ~ '</a>'} %}and %count% more...{% endtrans %}
        </div>
    {% endif %}
<!-- ... -->
```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/edit.html.twig -->

<!-- ... -->
{% block content %}
    <h1>{% trans %}Job edit{% endtrans %}</h1>
    <!-- ... -->
        <br /> {% trans %}Whether the job can also be published on affiliate websites or not.{% endtrans %}
    <!-- ... -->
<!-- ... -->
```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/admin.html.twig -->

<div id="job_actions">
    <h3>Admin</h3>
    <ul>
        {% if not job.isActivated %}
            <ul>
                <li><a href="{{ path('ens_job_edit', { 'token': job.token }) }}">{% trans %}Edit{% endtrans %}</a></li>
                <li>
                    <form action="{{ path('ens_job_publish', { 'token': job.token }) }}" method="post">
                        {{ form_widget(publish_form) }}
                            <button type="submit">{% trans %}Publish{% endtrans %}</button>
                    </form>
                </li>
            </ul>
        {% endif %}
        <li>
            <form action="{{ path('ens_job_delete', { 'token': job.token }) }}" method="post">
                {{ form_widget(delete_form) }}
                    <button type="submit" onclick="if(!confirm('{% trans %}Are you sure?{% endtrans %}')) { return false; }">{% trans %}Delete{% endtrans %}</button>
            </form>
        </li>
        {% if job.isActivated %}
            <li {% if job.expiresSoon %} class="expires_soon" {% endif %}>
                {% if job.isExpired %}
                    {% trans %}Expired{% endtrans %}
                {% else %}
                    {% trans with {'%count%':'<strong>' ~ job.getDaysBeforeExpires ~ '</strong>' } %}Expires in %count% days{% endtrans %}
                {% endif %}

                {% if job.expiresSoon %}
                    <form action="{{ path('ens_job_extend', { 'token': job.token }) }}" method="post">
                        {{ form_widget(extend_form) }}
                            <button type="submit" value="Extend">{% trans %}Extend{% endtrans %}</button> {% trans %}for another 30 days{% endtrans %}
                    </form>
                {% endif %}
            </li>
        {% else %}
            <li>
                [{% trans with {'%url%': '<a href="' ~ url('ens_job_preview', { 'token': job.token, 'company': job.companyslug, 'location': job.locationslug, 'position': job.positionslug }) ~ '">URL</a>'} %}Bookmark this %url% to manage this job in the future{% endtrans %}.]
            </li>
        {% endif %}
    </ul>
</div>

```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Category/show.html.twig -->

{% extends 'EnsJobeetBundle::layout.html.twig' %}

{% block title %}
    {% trans with {'%category%': category.name} %}Jobs in the %category% category{% endtrans %}
{% endblock %}

{% block stylesheets %}
    {{ parent() }}
    <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/jobs.css') }}" type="text/css" media="all" />
{% endblock %}

{% block content %}
    <div class="category">
        <div class="feed">
            <a href="{{ path('EnsJobeetBundle_category', { 'slug': category.slug, '_format': 'atom' }) }}">Feed</a>
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
        {% transchoice total_jobs with {'%count%': '<strong>' ~ total_jobs ~ '</strong>'} %}
            {0} No job in this category|{1} One job in this category|]1,Inf] %count% jobs in this category
        {% endtranschoice %}
        {% if last_page > 1 %}
            - page <strong>{{ current_page }}/{{ last_page }}</strong>
        {% endif %}
    </div>        
{% endblock %}
```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Affiliate/wait.html.twig -->

{% extends "EnsJobeetBundle::layout.html.twig" %}

{% block content %}
    <div class="content">
        <h1>{% trans %}Your affiliate account has been created{% endtrans %}</h1>
        <div style="padding: 20px">
            {% trans %}Thank you!
            You will receive an email with your affiliate token
            as soon as your account will be activated.{% endtrans %}
        </div>
    </div>
{% endblock %}
```

```html
<!-- src/Ens/JobeetBundle/Resources/views/Affiliate/affiliate_new.html.twig -->

<!-- ... -->
    <h1>{% trans %}Become an affiliate{% endtrans %}</h1>
<!-- ... -->
```

每次翻译都交由一个trans-unit标签管理，它具有一个唯一的id属性。现在你可以编辑这个文件并添加法语支持。

```xml
<!-- src/Ens/JobeetBundle/Resources/translations/message.fr.xlf -->

<?xml version="1.0"?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
    <file source-language="en" datatype="plaintext" original="file.ext">
        <body>
            <trans-unit id="1">
                <source>Jobeet - Your best job board</source>
                <target>Jobeet - Les meilleurs offres d'emplois</target>
            </trans-unit>
            <trans-unit id="2">
                <source>Enter some keywords (city, country, position, ...)</source>
                <target>Entre des mots cle (ville, pays, position, ...)</target>
            </trans-unit>
            <trans-unit id="3">
                <source>Recent viewed jobs:</source>
                <target>Dernier emplois vus:</target>
            </trans-unit>
            <trans-unit id="4">
                <source>About Jobeet</source>
                <target>Apropos de Jobeet</target>
            </trans-unit>
            <trans-unit id="5">
                <source>Become an affiliate</source>
                <target>Devenir un affilie</target>
            </trans-unit>
            <trans-unit id="6">
                <source>and %count% more...</source>
                <target>et %count%
                        autres...</target>
            </trans-unit>
            <trans-unit id="7">
                <source>Language</source>
                <target>Langue</target>
            </trans-unit>
            <trans-unit id="8">
                <source>Publish</source>
                <target>Publier</target>
            </trans-unit>
            <trans-unit id="9">
                <source>Edit</source>
                <target>Editer</target>
            </trans-unit>
            <trans-unit id="10">
                <source>Are you sure?</source>
                <target>Etes-vous sur?</target>
            </trans-unit>
            <trans-unit id="11">
                <source>Delete</source>
                <target>Supprimer</target>
            </trans-unit>
            <trans-unit id="12">
                <source>Extend</source>
                <target>Prolonger</target>
            </trans-unit>
            <trans-unit id="13">
                <source>for another 30 days</source>
                <target>pour 30 jours supplementaires</target>
            </trans-unit>
            <trans-unit id="14">
                <source>Bookmark this %url% to manage this job in the future</source>
                <target>Marquer cette %url% pour gerer ce travail a l'avenir</target>
            </trans-unit>
            <trans-unit id="15">
                <source>Whether the job can also be published on affiliate websites or not.</source>
                <target>Si le travail peut egalement etre publie sur les sites affilies ou non.</target>
            </trans-unit>
            <trans-unit id="16">
                <source>%company% is looking for a %position%</source>
                <target>%company% est a la recherche d'un %position%</target>
            </trans-unit>
            <trans-unit id="17">
                <source>How to apply?</source>
                <target>comment appliquer?</target>
            </trans-unit>
            <trans-unit id="18">
                <source>posted on %date%</source>
                <target>poste en %date%</target>
            </trans-unit>
            <trans-unit id="19">
                <source>{0} No job in this category|{1} One job in this category|]1,Inf] %count% jobs in this category</source>
                <target>{0}Aucune annonce dans cette categorie|{1}Une annonce dans cette categorie|]1,+Inf] %count% annonces dans cette categorie</target>
            </trans-unit>
            <trans-unit id="20">
                <source>Jobs in the %category% category</source>
                <target>Travails dans le %category% categorie</target>
            </trans-unit>
            <trans-unit id="21">
                <source>Your affiliate account has been created</source>
                <target>Votre compte d'affiliation a ete cree</target>
            </trans-unit>
            <trans-unit id="22">
                <source>Thank you!
            You will receive an email with your affiliate token
            as soon as your account will be activated.</source>
                <target>On te remercie! Vous recevrez un email avec votre jeton d'affiliation des que votre compte sera active.</target>
            </trans-unit>
            <trans-unit id="23">
                <source>Expires in %count% days</source>
                <target>Expire en %count% jours</target>
            </trans-unit>
            <trans-unit id="24">
                <source>Ask for people</source>
                <target>Recherche des gens</target>
            </trans-unit>
            <trans-unit id="25">
                <source>Ask for a job</source>
                <target>Recherche d'un emploi</target>
            </trans-unit>
            <trans-unit id="26">
                <source>Jobeet API</source>
                <target>API Jobeet</target>
            </trans-unit>
            <trans-unit id="27">
                <source>Job creation</source>
                <target>Creation d'emploi</target>
            </trans-unit>
            <trans-unit id="28">
                <source>Job edit</source>
                <target>Edit l'emploi</target>
            </trans-unit>
            <trans-unit id="29">
                <source>Expired</source>
                <target>Expiré</target>
            </trans-unit>
            <trans-unit id="30">
                <source>Full feed</source>
                <target>Fil RSS</target>
            </trans-unit>
            <trans-unit id="31">
                <source>Post a Job</source>
                <target>Poste un emploi </target>
            </trans-unit>
        </body>
    </file>
</xliff>

```

对了，每当你添加完新的翻译，一定要记得清除缓存，否则不会生效。

至此我们的基于Symfony2的Jobeet实战课程终于结束了。也不知道你学会了没有，不过不要紧，不会就多看几遍嘛<_<当然如果有什么问题也可以直接在我的个github上留言，如果我看到了而且会的话，那么会回答你的XD
