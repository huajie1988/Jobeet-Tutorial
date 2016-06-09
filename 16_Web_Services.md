# Symfony2 Jobeet Day 15: Web Services

我们为Jobeet添加了订阅之后，求职者已经能实时了解新的就业机会。

另一方面，当你发布了一份职位信息之后，可能希望更多的人能看到这个信息。如果你的工作在很多小网站上联合展示，你将会有更好的机会找到合适这份职位的人。这就是长尾理论的力量。合作伙伴将能够在他们的网站发布我们最新的职位信息，这有赖于我们今天开发的web services功能。

## 合作伙伴

正如我们在本教程第2天已经说的，合作伙伴可以获取当前有效的职位信息列表。

## 似乎需要点数据

让我们为合作伙伴创建一个新的fixture文件。

```php
// src/Ens/JobeetBundle/DataFixtures/ORM/LoadAffiliateData.php
namespace Ens\JobeetBundle\DataFixtures\ORM;
 
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Common\DataFixtures\AbstractFixture;
use Doctrine\Common\DataFixtures\OrderedFixtureInterface;
use Ens\JobeetBundle\Entity\Affiliate;
 
class LoadAffiliateData extends AbstractFixture implements OrderedFixtureInterface
{
    public function load(ObjectManager $em)
    {
        $affiliate = new Affiliate();
 
        $affiliate->setUrl('http://sensio-labs.com/');
        $affiliate->setEmail('address1@example.com');
        $affiliate->setToken('sensio-labs');
        $affiliate->setIsActive(true);
        $affiliate->addCategorie($em->merge($this->getReference('category-programming')));
 
        $em->persist($affiliate);
 
        $affiliate = new Affiliate();
 
        $affiliate->setUrl('/');
        $affiliate->setEmail('address2@example.org');
        $affiliate->setToken('symfony');
        $affiliate->setIsActive(false);
        $affiliate->addCategorie($em->merge($this->getReference('category-programming')), $em->merge($this->getReference('category-design')));
 
        $em->persist($affiliate);
        $em->flush();
 
        $this->addReference('affiliate', $affiliate);
    }
 
    public function getOrder()
    {
        return 3; // This represents the order in which fixtures will be loaded
    }
}

```

现在使用如下命令即可将你的数据保存到数据库中：

> **php app/console doctrine:fixtures:load**

在fixture文件中，Token使用硬编码以简化测试，但当一个实际用户申请帐户时，我们需要将其由系统生成。为此，让我们在Affiliate类中新建一个函数。首先需要将setTokenValue方法添加到lifecycleCallbacks中：

```yaml
#src//Ens/JobeetBundle/Resources/config/doctrine/Affiliate.orm.yml
    lifecycleCallbacks:
        prePersist: [ setCreatedAtValue, setTokenValue ]
```

现在让我们来创建setTokenValue函数，运行以下命令：

> **php app/console doctrine:generate:entities EnsJobeetBundle**

接着修改该方法：

```php
// src/Ens/JobeetBundle/Entity/Affiliate.php
public function setTokenValue()
    {
        if(!$this->getToken()) {
            $token = sha1($this->getEmail().rand(11111, 99999));
            $this->token = $token;
        }
 
        return $this;
    }
```

重新加载数据

> **php app/console doctrine:fixtures:load**

## 职位Web Service

当您创建新资源的时候，先定义路由总是一种好的习惯︰

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml
#...

EnsJobeetBundle_api:
    pattern: /api/{token}/jobs.{_format}
    defaults: {_controller: "EnsJobeetBundle:Api:list"}
    requirements:
        _format: xml|json|yaml

#...
```

通常修改路由文件之后要记得清除缓存：

> **php app/console cache:clear --env=dev**
> **php app/console cache:clear --env=prod**


下一步是创建 api action及其模板，它们将共享相同的操作。现在让我们创建一个新的控制器文件，称为 ApiController:

```php

// src/Ens/JobeetBundle/Controller/ApiController.php

namespace Ens\JobeetBundle\Controller;
 
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Ens\JobeetBundle\Entity\Affiliate;
use Ens\JobeetBundle\Entity\Job;
use Ens\JobeetBundle\Repository\AffiliateRepository;
 
class ApiController extends Controller
{
    public function listAction(Request $request, $token)
    {
        $em = $this->getDoctrine()->getManager();
 
        $jobs = array();
 
        $rep = $em->getRepository('EnsJobeetBundle:Affiliate');
        $affiliate = $rep->getForToken($token);
 
        if(!$affiliate) { 
            throw $this->createNotFoundException('This affiliate account does not exist!');
        }
 
        $rep = $em->getRepository('EnsJobeetBundle:Job');
        $active_jobs = $rep->getActiveJobs(null, null, null, $affiliate->getId());
 
        foreach ($active_jobs as $job) {
            $jobs[$this->get('router')->generate('Ens_job_show', array('company' => $job->getCompanySlug(), 'location' => $job->getLocationSlug(), 'id' => $job->getId(), 'position' => $job->getPositionSlug()), true)] = $job->asArray($request->getHost());
        }
 
        $format = $request->getRequestFormat();
        $jsonData = json_encode($jobs);
 
        if ($format == "json") {
            $headers = array('Content-Type' => 'application/json'); 
            $response = new Response($jsonData, 200, $headers);
 
            return $response;
        }
 
        return $this->render('EnsJobeetBundle:Api:jobs.' . $format . '.twig', array('jobs' => $jobs));  
    }
}
```

为了通过Token获取合作伙伴的信息，我们将创建 getForToken() 方法。此方法还会验证合作伙伴的帐户是否被激活，所以不需要额外再检查了。到目前为止，我们还没有使用 AffiliateRepository，这是因为它还不存在。。。若要创建它，请修改 ORM 文件如下：

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Affiliate.orm.yml

EnsJobeetBundleEntityAffiliate:
    type: entity
    repositoryClass: EnsJobeetBundleRepositoryAffiliateRepository
    # ...
```

> *译者注：原文缺少生成Repository的命令，请记得执行 php app/console doctrine:generate:entities EnsJobeetBundle *

创建完成之后打开︰

```php

// src/Ens/JobeetBundle/Repository/AffiliateRepository.php

namespace Ens\JobeetBundle\Repository;
 
use Doctrine\ORM\EntityRepository;
 
/**
 * AffiliateRepository
 *
 * This class was generated by the Doctrine ORM. Add your own custom
 * repository methods below.
 */
class AffiliateRepository extends EntityRepository
{
    public function getForToken($token)
    {
        $qb = $this->createQueryBuilder('a')
            ->where('a.is_active = :active')
            ->setParameter('active', 1)
            ->andWhere('a.token = :token')
            ->setParameter('token', $token)
            ->setMaxResults(1)
        ;
 
        try{
            $affiliate = $qb->getQuery()->getSingleResult();
        } catch(\Doctrine\Orm\NoResultException $e){
            $affiliate = null;
        }
 
        return $affiliate;
    }
}
```

现在我们能通过Token来识别合作伙伴了，我们可以使用getActiveJobs()方法来提供给合作伙伴他们想要的数据。如果你现在打开 JobRepository 文件，您将看到 getActiveJobs() 方法没有给任何合作伙伴提供共享方式。为此我们需要将其进行重构：

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php

// ...
 
    public function getActiveJobs($category_id = null, $max = null, $offset = null, $affiliate_id = null)
    {
        $qb = $this->createQueryBuilder('j')
            ->where('j.expires_at > :date')
            ->setParameter('date', date('Y-m-d H:i:s', time()))
            ->andWhere('j.is_activated = :activated')
            ->setParameter('activated', 1)
            ->orderBy('j.expires_at', 'DESC');
 
        if($max) {
            $qb->setMaxResults($max);
        }
 
        if($offset) {
            $qb->setFirstResult($offset);
        }
 
        if($category_id) {
            $qb->andWhere('j.category = :category_id')
                ->setParameter('category_id', $category_id);
        }
        // j.category c, c.affiliate a
        if($affiliate_id) {
            $qb->leftJoin('j.category', 'c')
               ->leftJoin('c.affiliates', 'a')
               ->andWhere('a.id = :affiliate_id')
               ->setParameter('affiliate_id', $affiliate_id)
            ;
        }
 
        $query = $qb->getQuery();
 
        return $query->getResult();
    }
 
// ...

```

如你所见，我们之前使用了一个新的函数asArray()，现在让我们定义它：

```php
public function asArray($host)
{
    return array(
        'category'     => $this->getCategory()->getName(),
        'type'         => $this->getType(),
        'company'      => $this->getCompany(),
        'logo'         => $this->getLogo() ? 'http://' . $host . '/uploads/jobs/' . $this->getLogo() : null,
        'url'          => $this->getUrl(),
        'position'     => $this->getPosition(),
        'location'     => $this->getLocation(),
        'description'  => $this->getDescription(),
        'how_to_apply' => $this->getHowToApply(),
        'expires_at'   => $this->getCreatedAt()->format('Y-m-d H:i:s'),
    );
}
```

## XML 格式

返回XML格式很简单，只需要创建一个模板即可：

```xml
<!-- src/Ens/JobeetBundle/Resources/views/Api/Jobs.xml.twig -->
<?xml version="1.0" encoding="utf-8"?>
<jobs>
{% for url, job in jobs %}
    <job url="{{ url }}">
{% for key,value in job %}
        <{{ key }}>{{ value }}</{{ key }}>
{% endfor %}
    </job>
{% endfor %}
</jobs>
```

## JSON格式

支持JSON也是一样的

```json
// src/Ens/JobeetBundle/Resources/views/Api/jobs.json.twig

{% for url, job in jobs %}
{% i = 0, count(jobs), ++i %}
[
    "url":"{{ url }}",
{% for key, value in job %} {% j = 0, count(key), ++j %}
    "{{ key }}":"{% if j == count(key)%} {{ json_encode(value) }}, {% else %} {{ json_encode(value) }}
                 {% endif %}"
{% endfor %}]
{% endfor %}
```

## YAML格式

```yaml
# src/Ens/JobeetBundle/Resources/views/Api/jobs.yaml.twig

{% for url,job in jobs %}
    Url: {{ url }}
{% for key, value in job %}
        {{ key }}: {{ value }}
{% endfor %}
{% endfor %}
```

如果你尝试使用一个无效的token来调用此 web service，则会返回一个404响应。如今你可以使用以下链接来测试一下效果如何。http://jobeet.local/app_dev.php/api/sensio-labs/jobs.xml 或 http://jobeet.local/app_dev.php/api/symfony/jobs.xml。
你也可以修改URL的后缀来返回你所喜欢的格式。

## 测试你的Web Services

```php
// src/Ens/JobeetBundle/Tests/Controller/ApiControllerTest.php

namespace Ens\JobeetBundle\Tests\Controller;
 
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;
use Symfony\Component\DomCrawler\Crawler;
use Symfony\Component\HttpFoundation\HttpExceptionInterface;
 
class ApiControllerTest extends WebTestCase
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
 
    public function testList()
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/api/sensio-labs/jobs.xml');
 
        $this->assertEquals('Ens\JobeetBundle\Controller\ApiController::listAction', $client->getRequest()->attributes->get('_controller'));
        $this->assertTrue($crawler->filter('description')->count() == 32);
 
        $crawler = $client->request('GET', '/api/sensio-labs87/jobs.xml');
 
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
 
        $crawler = $client->request('GET', '/api/symfony/jobs.xml');
 
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
 
        $crawler = $client->request('GET', '/api/sensio-labs/jobs.json');
 
        $this->assertEquals('Ens\JobeetBundle\Controller\ApiController::listAction', $client->getRequest()->attributes->get('_controller'));
        $this->assertRegExp('/"category":"Programming"/', $client->getResponse()->getContent());
 
        $crawler = $client->request('GET', '/api/sensio-labs87/jobs.json');
 
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
 
        $crawler = $client->request('GET', '/api/sensio-labs/jobs.yaml');
        $this->assertRegExp('/category: Programming/', $client->getResponse()->getContent());
 
        $this->assertEquals('Ens\JobeetBundle\Controller\ApiController::listAction', $client->getRequest()->attributes->get('_controller'));
 
        $crawler = $client->request('GET', '/api/sensio-labs87/jobs.yaml');
 
        $this->assertTrue(404 === $client->getResponse()->getStatusCode());
    }
}
```

在ApiControllerTest文件里面，我们对请求格式的正确接收及其是否返回正确的请求页面都进行了测试。


## 合作伙伴申请表单

现在我们的web service已经可以正常使用了，接下来让我们来为申请成为合作伙伴制作一个表单。为此，你需要写 HTML 表单，为每个字段的有效性创建验证规则，处理的表单提交的值并将它们存储到数据库中，当有错误时还要显示错误消息、重新填充表单字段等情况。

现在让我们创建一个新的控制器文件，并将其命名为AffiliateController：

```php
// src/Ens/JobeetBundle/Controller/AffiliateController.php

namespace Ens\JobeetBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Ens\JobeetBundle\Entity\Affiliate;
use Ens\JobeetBundle\Form\AffiliateType;
use Symfony\Component\HttpFoundation\Request;
use Ens\JobeetBundle\Entity\Category;

class AffiliateController extends Controller
{
    // Your code goes here
}
```

然后在布局文件中添加申请链接：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->

<!-- ... -->
    <li class="last"><a href="{{ path('ens_affiliate_new') }}">Become an affiliate</a></li>
<!-- ... -->
```

接下来创建一个action来匹配这个路由：

```php
// src/Ens/JobeetBundle/Controller/AffiliateController.php

namespace Ens\JobeetBundle\Controller;
 
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Ens\JobeetBundle\Entity\Affiliate;
use Ens\JobeetBundle\Form\AffiliateType;
use Symfony\Component\HttpFoundation\Request;
use Ens\JobeetBundle\Entity\Category;
 
class AffiliateController extends Controller
{
    public function newAction()
    {
        $entity = new Affiliate();
        $form = $this->createForm(new AffiliateType(), $entity);
 
        return $this->render('Ens\JobeetBundle:Affiliate:affiliate_new.html.twig', array(
            'entity' => $entity,
            'form'   => $form->createView(),
        ));
    }
}
```

我们有了路由的名称，也有了处理路由的action，但还没有具体的路由映射，所以现在让我们来创建它：

```yaml
#src/Ens/JobeetBundle/Resources/config/routing/affiliate.yml
ens_affiliate_new:
    pattern:  /new
    defaults: { _controller: "EnsJobeetBundle:Affiliate:new" }

```

当然也别忘了将你的路由文件添加到主路由中去：

```yaml
src/Ens/JobeetBundle/Resources/config/routing.yml

# ...

EnsJobeetBundle_ens_affiliate:
    resource: "@EnsJobeetBundle/Resources/config/routing/affiliate.yml"
    prefix:   /affiliate
```

接下来我们需要开始创建表单文件。尽管Affiliate表的字段很多，但我们没必要将其一一呈现出来，因为其中某些字段对最终用户来说应该是不可编辑的。现在创建你的表单：

```php
// src/Ens/JobeetBundle/Form/AffiliateType.php

namespace Ens\JobeetBundle\Form;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolverInterface;
use Ens\JobeetBundle\Entity\Affiliate;
use Ens\JobeetBundle\Entity\Category;

class AffiliateType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('url')
            ->add('email')
            ->add('categories', null, array('expanded'=>true))
        ;
    }

    public function setDefaultOptions(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'data_class' => 'Ens\JobeetBundle\Entity\Affiliate',
        ));
    }

    public function getName()
    {
        return 'affiliate';
    }
}
```

现在我们需要为提交的数据添加数据有效性的检测，为此我们需要添加如下代码：

```yaml
# src/Ens/JobeetBundle/Resources/config/validation.yml

# ...

Ens\JobeetBundle\Entity\Affiliate:
    constraints:
        - Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity: email
    properties:
        url:
            - Url: ~
        email:
            - NotBlank: ~
            - Email: ~
```

在验证规则中我们使用了一个新的验证器---UniqueEntity。它检验一个字段或一组字段的值在entity中是否唯一。这很常用。比如防止新的用户使用一个在系统中已经存在的电子邮件地址进行注册。

添加验证之后别忘了清除缓存。

最后让我们来创建表单视图：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Affiliate/affiliate_new.html.twig -->

{% extends 'EnsJobeetBundle::layout.html.twig' %}

{% set form_themes = _self %}

{% block form_errors %}
{% spaceless %}
    {% if errors|length > 0 %}
        <ul class="error_list">
            {% for error in errors %}
                <li>{{ error.messageTemplate|trans(error.messageParameters, 'validators') }}</li>
            {% endfor %}
        </ul>
    {% endif %}
{% endspaceless %}
{% endblock form_errors %}

{% block stylesheets %}
    {{ parent() }}
    <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/job.css') }}" type="text/css" media="all" />
{% endblock %}

{% block content %}
    <h1>Become an affiliate</h1>
        <form action="{{ path('ens_affiliate_create') }}" method="post" {{ form_enctype(form) }}>
            <table id="job_form">
                <tfoot>
                    <tr>
                        <td colspan="2">
                            <input type="submit" value="Submit" />
                        </td>
                    </tr>
                </tfoot>
                <tbody>
                    <tr>
                        <th>{{ form_label(form.url) }}</th>
                        <td>
                            {{ form_errors(form.url) }}
                            {{ form_widget(form.url) }}
                        </td>
                    </tr>
                    <tr>
                        <th>{{ form_label(form.email) }}</th>
                        <td>
                            {{ form_errors(form.email) }}
                            {{ form_widget(form.email) }}
                        </td>
                    </tr>
                    <tr>
                        <th>{{ form_label(form.categories) }}</th>
                        <td>
                            {{ form_errors(form.categories) }}
                            {{ form_widget(form.categories) }}
                        </td>
                    </tr>
                </tbody>
            </table>
        {{ form_end(form) }}
{% endblock %}

```

当用户提交这个表单之后，如果数据验证通过，则信息将会被保存到数据库中。为此我们需要在AffiliateController中添加一个create action：

```php
// src/Ens/JobeetBundle/Controller/AffiliateController.php

class AffiliateController extends Controller 
{
    // ...    

    public function createAction(Request $request)
    {
        $affiliate = new Affiliate();
        $form = $this->createForm(new AffiliateType(), $affiliate);
        $form->bind($request);
        $em = $this->getDoctrine()->getManager();

        if ($form->isValid()) {

            $formData = $request->get('affiliate');
            $affiliate->setUrl($formData['url']);
            $affiliate->setEmail($formData['email']);
            $affiliate->setIsActive(false);

            $em->persist($affiliate);
            $em->flush();

            return $this->redirect($this->generateUrl('ens_affiliate_wait'));
        }

        return $this->render('Ens\JobeetBundle:Affiliate:affiliate_new.html.twig', array(
            'entity' => $affiliate,
            'form'   => $form->createView(),
        ));
    }
}
```

为了能使创建操作正确执行，我们还需要添加路由映射：

```yaml
#src/Ens/JobeetBundle/Resources/config/routing/affiliate.yml
#...
 
ens_affiliate_create:
    pattern: /create
    defaults: { _controller: "EnsJobeetBundle:Affiliate:create" }
    requirements: { _method: post }
```

当合作伙伴注册完成之后将会跳到一个等待页面，现在让我们来实现它：

```php
src/Ens/JobeetBundle/Controller/AffiliateController.php

class AffiliateController extends Controller
{
    // ...

    public function waitAction()
    {
        return $this->render('EnsJobeetBundle:Affiliate:wait.html.twig');
    }
}
```

```html
src/Ens/JobeetBundle/Resources/views/Affiliate/wait.html.twig

{% extends "EnsJobeetBundle::layout.html.twig" %}

{% block content %}
    <div class="content">
        <h1>Your affiliate account has been created</h1>
        <div style="padding: 20px">
            Thank you!
            You will receive an email with your affiliate token
            as soon as your account will be activated.
        </div>
    </div>
{% endblock %}
```

```yaml
#src/Ens/JobeetBundle/Resources/config/routing/affiliate.yml

#...

ens_affiliate_wait:
    pattern: /wait
    defaults: { _controller: "EnsJobeetBundle:Affiliate:wait" }
```

由于我们添加了路由，所以要记得清除缓存。

现在你可以试着在主页上点击合作伙伴的链接，它就能跳到申请页面了。

## 测试

为新功能添加功能测试：

```php
namespace Ens\JobeetBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;
use Symfony\Component\DomCrawler\Crawler;

class AffiliateControllerTest extends WebTestCase
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

    public function testAffiliateForm()
    {
        $client = static::createClient();
        $crawler = $client->request('GET', '/affiliate/new');

        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::newAction', $client->getRequest()->attributes->get('_controller'));

        $form = $crawler->selectButton('Submit')->form(array(
            'affiliate[url]' => 'http://sensio-labs.com/',
            'affiliate[email]' => 'jobeet@example.com'
        ));

        $client->submit($form);
        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::createAction', $client->getRequest()->attributes->get('_controller'));

        $kernel = static::createKernel();
        $kernel->boot();
        $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');

        $query = $em->createQuery('SELECT count(a.email) FROM EnsJobeetBundle:Affiliate a WHERE a.email = :email');
        $query->setParameter('email', 'jobeet@example.com');
        $this->assertEquals(1, $query->getSingleScalarResult());

        $crawler = $client->request('GET', '/affiliate/new');
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
        $crawler = $client->request('GET', '/affiliate/new');
        $form = $crawler->selectButton('Submit')->form(array(
            'affiliate[url]' => 'http://sensio-labs.com/',
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
        $crawler = $client->request('GET', '/affiliate/wait');

        $this->assertEquals('Ens\JobeetBundle\Controller\AffiliateController::waitAction', $client->getRequest()->attributes->get('_controller'));
    }
}

```

## 合作伙伴管理后台

我们需要使用SonataAdminBundle来为合作伙伴做一个管理后台。正如我们之前所说，当一个合作伙伴提交申请之后，他需要等待我们的管理员对其进行激活。所以当一个管理员访问合作伙伴页面时，他将只能看到未激活的账号，这样有助于他的工作更有效率。

首先我们需要在services.yml文件中定义一个新的affiliate服务

```yaml
#src/Ens/JobeetBundle/Resources/config/services.yml

#...
    ens.jobeet.admin.affiliate:
        class: Ens\JobeetBundle\Admin\AffiliateAdmin
        tags:
            - { name: sonata.admin, manager_type: orm, group: jobeet, label: Affiliates }
        arguments:
            - ~
            - Ens\JobeetBundle\Entity\Affiliate
            - 'EnsJobeetBundle:AffiliateAdmin'
```

接着创建admin文件：

```php
// src/Ens/JobeetBundle/Admin/AffiliateAdmin.php

namespace Ens\JobeetBundle\Admin;

use Sonata\AdminBundle\Admin\Admin;
use Sonata\AdminBundle\Datagrid\ListMapper;
use Sonata\AdminBundle\Datagrid\DatagridMapper;
use Sonata\AdminBundle\Validator\ErrorElement;
use Sonata\AdminBundle\Form\FormMapper;
use Sonata\AdminBundle\Show\ShowMapper;
use Ens\JobeetBundle\Entity\Affiliate;

class AffiliateAdmin extends Admin
{
    protected $datagridValues = array(
        '_sort_order' => 'ASC',
        '_sort_by' => 'is_active'
    );

    protected function configureFormFields(FormMapper $formMapper)
    {
        $formMapper
            ->add('email')
            ->add('url')
        ;
    }

    protected function configureDatagridFilters(DatagridMapper $datagridMapper)
    {
        $datagridMapper
            ->add('email')
            ->add('is_active');
    }

    protected function configureListFields(ListMapper $listMapper)
    {
        $listMapper
            ->add('is_active')
            ->addIdentifier('email')
            ->add('url')
            ->add('created_at')
            ->add('token')
        ;
    }
}
```

为了帮助管理员更好的工作，我们将只显示未被激活的账号。以下代码能帮我们筛选出is_active为false的数据：

```php
// src/Ens/JobeetBundle/Admin/AffiliateAdmin.php
// ...
    protected $datagridValues = array(
        '_sort_order' => 'ASC',
        '_sort_by' => 'is_active',
        'is_active' => array('value' => 2) // The value 2 represents that the displayed affiliate accounts are not activated yet
    );
 
// ...
```

接下来创建AffiliateAdmin控制器：

```php
// src/Ens/JobeetBundle/Controller/AffiliateAdminController.php

namespace Ens\JobeetBundle\Controller;

use Sonata\AdminBundle\Controller\CRUDController as Controller;
use Sonata\DoctrineORMAdminBundle\Datagrid\ProxyQuery as ProxyQueryInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;

class AffiliateAdminController extends Controller
{
    // Your code goes here
}
```

让我们创建激活和停用这两个批处理操作︰

```php
// src/Ens/JobeetBundle/Controller/AffiliateAdminController.php

namespace Ens\JobeetBundle\Controller;

use Sonata\AdminBundle\Controller\CRUDController as Controller;
use Sonata\DoctrineORMAdminBundle\Datagrid\ProxyQuery as ProxyQueryInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;

class AffiliateAdminController extends Controller
{
    public function batchActionActivate(ProxyQueryInterface $selectedModelQuery)
    {
        if($this->admin->isGranted('EDIT') === false || $this->admin->isGranted('DELETE') === false) {
            throw new AccessDeniedException();
        }

        $request = $this->get('request');
        $modelManager = $this->admin->getModelManager();

        $selectedModels = $selectedModelQuery->execute();

        try {
            foreach($selectedModels as $selectedModel) {
                $selectedModel->activate();
                $modelManager->update($selectedModel);
            }
        } catch(\Exception $e) {
            $this->get('session')->getFlashBag()->add('sonata_flash_error', $e->getMessage());

            return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
        }

        $this->get('session')->getFlashBag()->add('sonata_flash_success',  sprintf('The selected accounts have been activated'));

        return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
    }

    public function batchActionDeactivate(ProxyQueryInterface $selectedModelQuery)
    {
        if($this->admin->isGranted('EDIT') === false || $this->admin->isGranted('DELETE') === false) {
            throw new AccessDeniedException();
        }

        $request = $this->get('request');
        $modelManager = $this->admin->getModelManager();

        $selectedModels = $selectedModelQuery->execute();

        try {
            foreach($selectedModels as $selectedModel) {
                $selectedModel->deactivate();
                $modelManager->update($selectedModel);
            }
        } catch(\Exception $e) {
            $this->get('session')->getFlashBag()->add('sonata_flash_error', $e->getMessage());

            return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
        }

        $this->get('session')->getFlashBag()->add('sonata_flash_success',  sprintf('The selected accounts have been deactivated'));

        return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
    }
}

```

为了让这两个新的批处理操作能正常工作，我们需要在Admin类中的getBatchActions方法中添加它们︰

```php
// src/Ens/JobeetBundle/Admin/AffiliateAdmin.php

class AffiliateAdmin extends Admin
{
    // ... 

    public function getBatchActions()
    {
        $actions = parent::getBatchActions();

        if($this->hasRoute('edit') && $this->isGranted('EDIT') && $this->hasRoute('delete') && $this->isGranted('DELETE')) {
            $actions['activate'] = array(
                'label'            => 'Activate',
                'ask_confirmation' => true
            );

            $actions['deactivate'] = array(
                'label'            => 'Deactivate',
                'ask_confirmation' => true
            );
        }

        return $actions;
    }
}

```

当然还没完，之前我们用了activate和deactivate方法，现在让我们在entity中加入它们：

```php
// src/Ens/JobeetBundle/Entity/Affiliate.php
// ...
 
    public function activate()
    {
        if(!$this->getIsActive()) {
            $this->setIsActive(true);
        }
 
        return $this->is_active;
    }
 
    public function deactivate()
    {
        if($this->getIsActive()) {
            $this->setIsActive(false);
        }
 
        return $this->is_active;
    }
```

马达马达，我们还有activate和deactivate两个独立的action要做呢。如前所述，我们先创建路由。不过这次有一些不同，我们需要将Admin类继承的configureRoutes方法进行重写：

```php
// src/Ens/JobeetBundle/Admin/AffiliateAdmin.php

use Sonata\AdminBundle\Route\RouteCollection;

class AffiliateAdmin extends Admin
{
    // ...

    protected function configureRoutes(RouteCollection $collection) {
        parent::configureRoutes($collection);

        $collection->add('activate',
            $this->getRouterIdParameter().'/activate')
        ;

        $collection->add('deactivate',
            $this->getRouterIdParameter().'/deactivate')
        ;
    }
}

```

基础工作做完了也是时候实现我们的action了：

```php
// src/Ens/JobeetBundle/Controller/AffiliateAdminController.php

class AffiliateAdminController extends Controller
{
    // ...

    public function activateAction($id)
    {
        if($this->admin->isGranted('EDIT') === false) {
            throw new AccessDeniedException();
        }

        $em = $this->getDoctrine()->getManager();
        $affiliate = $em->getRepository('EnsJobeetBundle:Affiliate')->findOneById($id);

        try {
            $affiliate->setIsActive(true);
            $em->flush();
        } catch(\Exception $e) {
            $this->get('session')->getFlashBag()->add('sonata_flash_error', $e->getMessage());

            return new RedirectResponse($this->admin->generateUrl('list', $this->admin->getFilterParameters()));
        }

        return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));

    }

    public function deactivateAction($id)
    {
        if($this->admin->isGranted('EDIT') === false) {
            throw new AccessDeniedException();
        }

        $em = $this->getDoctrine()->getManager();
        $affiliate = $em->getRepository('EnsJobeetBundle:Affiliate')->findOneById($id);

        try {
            $affiliate->setIsActive(false);
            $em->flush();
        } catch(\Exception $e) {
            $this->get('session')->getFlashBag()->add('sonata_flash_error', $e->getMessage());

            return new RedirectResponse($this->admin->generateUrl('list', $this->admin->getFilterParameters()));
        }

        return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
    }
}
```

接下来创建两个新的模板用来添加新的按钮：

```html
<!-- src/Ens/JobeetBundle/Resources/views/AffiliateAdmin/list__action_activate.html.twig -->

{% if admin.isGranted('EDIT', object) and admin.hasRoute('activate') %}
    <a href="{{ admin.generateObjectUrl('activate', object) }}" class="btn edit_link" title="{{ 'action_activate'|trans({}, 'SonataAdminBundle') }}">
        <i class="icon-edit"></i>
        {{ 'activate'|trans({}, 'SonataAdminBundle') }}
    </a>
{% endif %}
```

```html
<!-- src/Ens/JobeetBundle/Resources/views/AffiliateAdmin/list__action_deactivate.html.twig -->

{% if admin.isGranted('EDIT', object) and admin.hasRoute('deactivate') %}
    <a href="{{ admin.generateObjectUrl('deactivate', object) }}" class="btn edit_link" title="{{ 'action_deactivate'|trans({}, 'SonataAdminBundle') }}">
        <i class="icon-edit"></i>
        {{ 'deactivate'|trans({}, 'SonataAdminBundle') }}
    </a>
{% endif %}

最后为了能使这些按钮和action显示并工作，我们需要在configureListFields方法中添加它们：

```php
// src/Ens/JobeetBundle/Admin/AffiliateAdmin.php

class AffiliateAdmin extends Admin
{
    // ...    

    protected function configureListFields(ListMapper $listMapper)
    {
        $listMapper
            ->add('is_active')
            ->addIdentifier('email')
            ->add('url')
            ->add('created_at')
            ->add('token')
            ->add('_action', 'actions', array( 'actions' => array('activate' => array('template' => 'EnsJobeetBundle:AffiliateAdmin:list__action_activate.html.twig'),
                'deactivate' => array('template' => 'EnsJobeetBundle:AffiliateAdmin:list__action_deactivate.html.twig'))))
        ;
    }
    /// ...
}
```

现在让我们来清除缓存看看效果吧！

这就是今天的所有内容了。在明天的课程中，我们将会学习如何在合作伙伴被激活之后自动发送邮件给他们。See you again！