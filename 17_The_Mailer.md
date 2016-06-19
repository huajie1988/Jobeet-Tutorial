# Symfony2 Jobeet Day 16: 邮件

昨天在我们的Jobeet添加了一个只读的Web Services。但合作伙伴在创建账号之后必须等待管理员激活之后才能使用。因此为了使合作伙伴能得到他们的Token，我们需要发送邮件通知他们。这就是我们今天所要做的内容。

在Symfony中内置了一个好用的邮件解决方案：Swift Mailer。不止如此，这个类库还与Symfony充分结合，并在其默认功能上加上了一些很酷的功能。现在让我们开始发送一封简单的邮件通知他们账号已激活并附上他们的Token。但首先，我们需要配置一下环境︰

```yaml
#app/config/parameters.yml

#...
    #...
    mailer_transport:  gmail
    mailer_host:       ~
    mailer_user:       address@example.com
    mailer_password:   your_password
    #...
```

> 为了让代码能正常工作，你需要将address@example.com和your_password替换为你真实的邮件地址和密码。

~~将同样的内容写到`app/config/parameters_test.yml`文件中~~

接着清除缓存：
> **php app/console cache:clear --env=dev**  
> **php app/console cache:clear --env=prod**  

因为我们传输方式选择的是gmail，所以当你替换mailer_user的时候请输入gmail的邮箱。

现在让我们来想一下，当你点击撰写按钮创建一封邮件之后你需要做哪些步骤呢？首先写一个主题，接着指定收件人，最后书写正文。

创建一封邮件你需要

* 调用Swift_message的newInstance()方法(你可以通过[Swift Mailer](http://swiftmailer.org/docs/introduction.html)的官方文档了解更多这个对象信息)
* 使用setFrom()方法设置发件人地址(From:)
* 使用setSubject()方法设置邮件主题
* 使用setTo(), setCc() 或 setBcc()设置收件人
* 使用setBody()设置邮件内容

修改activate action如下：

```php
// src/Ens/JobeetBundle/Controller/AffiliateAdminController.php

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

            $message = \Swift_Message::newInstance()
                ->setSubject('Jobeet affiliate token')
                ->setFrom('address@example.com')
                ->setTo($affiliate->getEmail())
                ->setBody(
                    $this->renderView('EnsJobeetBundle:Affiliate:email.txt.twig', array('affiliate' => $affiliate->getToken())))
            ;

            $this->get('mailer')->send($message);
        } catch(\Exception $e) {
            $this->get('session')->setFlash('sonata_flash_error', $e->getMessage());
        }

        return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
    }

// ...
```

发送邮件就是如此简单，我们只需要将Swift_Message实例传递给mailer的send方法就可以了。

对于邮件正文，我们我们创建一个新的文件，并将其命名为email.txt.twig，它包含了所有我们想通知给合作伙伴的内容。

```html
<!-- src/Ens/JobeetBundle/Resources/views/Affiliate/email.txt.twig -->

Your affiliate account has been activated.
Your secret token is {{affiliate}}.
You can see the jobs list at the following addresses:
http://jobeet.local/app_dev.php/api/{{affiliate}}/jobs.xml
or http://jobeet.local/app_dev.php/api/{{affiliate}}/jobs.json
or http://jobeet.local/app_dev.php/api/{{affiliate}}/jobs.yaml
```

现在让我们将邮件功能添加到batchActionActivate中，这样即使我们选择了多个合作伙伴进行激活，也能使他们收到邮件。

```php
// src/Ens/JobeetBundle/Controller/AffiliateAdminController.php

// ... 

    public function batchActionActivate(ProxyQueryInterface $selectedModelQuery)
    {
        // ...

        try {
            foreach($selectedModels as $selectedModel) {
                $selectedModel->activate();
                $modelManager->update($selectedModel);

                $message = \Swift_Message::newInstance()
                    ->setSubject('Jobeet affiliate token')
                    ->setFrom('address@example.com')
                    ->setTo($selectedModel->getEmail())
                    ->setBody(
                        $this->renderView('EnsJobeetBundle:Affiliate:email.txt.twig', array('affiliate' => $selectedModel->getToken())))
                ;

                $this->get('mailer')->send($message);
            }
        } catch(\Exception $e) {
            $this->get('session')->setFlash('sonata_flash_error', $e->getMessage());

            return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
        }

        // ...
    }

// ...
```

## 测试

现在我们已经看到如何使用symfony mailer组件来发送邮件，也是时候写一些功能测试来确保我们所做的是正确的。

若要测试此项新功能，我们需要登录。而登录又需要用户名和密码，为此我们不得不新建一个fixture文件，并为之添加用户管理员

```php
// src/Ens/JobeetBundle/DataFixtures/ORM/LoadUserData.php

namespace Ens\JobeetBundle\DataFixtures\ORM;

use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Common\DataFixtures\AbstractFixture;
use Doctrine\Common\DataFixtures\FixtureInterface;
use Doctrine\Common\DataFixtures\OrderedFixtureInterface;
use Symfony\Component\DependencyInjection\ContainerAwareInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Ens\JobeetBundle\Entity\User;

class LoadUserData implements FixtureInterface, OrderedFixtureInterface, ContainerAwareInterface
{
    /**
     * @var ContainerInterface
     */
    private $container;

    /**
     * {@inheritDoc}
     */
    public function setContainer(ContainerInterface $container = null)
    {
        $this->container = $container;
    }

    /**
     * @param \Doctrine\Common\Persistence\ObjectManager $em
     */
    public function load(ObjectManager $em)
    {
        $user = new User();
        $user->setUsername('admin');
        $encoder = $this->container
            ->get('security.encoder_factory')
            ->getEncoder($user)
        ;

        $encodedPassword = $encoder->encodePassword('admin', $user->getSalt());
        $user->setPassword($encodedPassword);

        $em->persist($user);
        $em->flush();
    }

    public function getOrder()
    {
        return 4; // the order in which fixtures will be loaded
    }
}

```

在这个测试中，我们将使用profiler的swiftmailer collector来获取上一次消息发送的信息。现在，让我们添加一些测试，以检查是否正确地发送电子邮件︰

```php

// src/Ens/JobeetBundle/Tests/Controller/AffiliateAdminControllerTest.php

namespace Ens\JobeetBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Output\NullOutput;
use Symfony\Component\Console\Input\ArrayInput;
use Doctrine\Bundle\DoctrineBundle\Command\DropDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\CreateDatabaseDoctrineCommand;
use Doctrine\Bundle\DoctrineBundle\Command\Proxy\CreateSchemaDoctrineCommand;

class AffiliateAdminControllerTest extends WebTestCase
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

    public function testActivate()
    {
        $client = static::createClient();

        // Enable the profiler for the next request (it does nothing if the profiler is not available)
        $client->enableProfiler();
        $crawler = $client->request('GET', '/login');

        $form = $crawler->selectButton('login')->form(array(
            '_username'      => 'admin',
            '_password'      => 'admin'
        ));

        $crawler = $client->submit($form);
        $crawler = $client->followRedirect();

        $this->assertTrue(200 === $client->getResponse()->getStatusCode());

        $crawler = $client->request('GET', '/admin/ibw/jobeet/affiliate/list');

        $link = $crawler->filter('.btn.edit_link')->link();
        $client->click($link);

        $mailCollector = $client->getProfile()->getCollector('swiftmailer');

        // Check that an e-mail was sent
        $this->assertEquals(1, $mailCollector->getMessageCount());

        $collectedMessages = $mailCollector->getMessages();
        $message = $collectedMessages[0];

        // Asserting e-mail data
        $this->assertInstanceOf('Swift_Message', $message);
        $this->assertEquals('Jobeet affiliate token', $message->getSubject());
        $this->assertRegExp(
            '/Your secret token is symfony/',
            $message->getBody()
        );
    }
}

```

如果你此时又不听劝告急急忙忙的去运行这个测试，那么结果是显而易见的---你会失败。为了防止这种情况的发生，你需要修改`config_test.yml`文件使profiler项开启。

```yaml
#app/config/config_test.yml

#...

framework:
    test: ~
    session:
        storage_id: session.storage.mock_file
    profiler:
        enabled: true

#...

```

现在你可以清除缓存来运行这个测试了，开始享受成功的喜悦吧。

>**phpunit -c app src/Ibw/JobeetBundle/Tests/Controller/AffiliateAdminControllerTest**