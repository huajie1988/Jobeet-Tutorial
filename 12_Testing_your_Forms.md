# Symfony2 Jobeet Day 11: 测试你的表单

在昨天的课程中，我们使用Symfony2创建了第一个表单。现在人们已经能够在Jobeet上发布职位了，然而鉴于昨天时间所限，我们未能对其进行测试。因此我们今天来继续完成这件事。

## 提交表单

让我们打开 JobControllerTest 文件为职位的创建和验证过程添加功能测试。在文件的末尾，添加以下代码，以获得职位创建页面 ︰

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function testJobForm()
{
  $client = static::createClient();
 
  $crawler = $client->request('GET', '/job/new');
  $this->assertEquals('Ens\JobeetBundle\Controller\JobController::newAction', $client->getRequest()->attributes->get('_controller'));
}
```

我们使用selectButton()方法来提交表单，该方法能匹配`button `和类型为submit的`input`标签。只要你有一个可以使用的按钮节点，该方法就可以调用form()方法生成一个表单实例：

```php
$form = $crawler->selectButton('submit')->form();
```

当我们调用form()方法时可以传递一个数组用来替换表单中的默认值：

```php
$form = $crawler->selectButton('submit')->form(array(
  'name' => 'Fabien',
  'my_form[subject]' => 'Symfony Rocks!'
));
```

但要传递字段的值，我们首先需要知道他们的名字。如果你查看网页源代码或者使用火狐的Web开发工具栏 "*Forms > Display Form Details*"功能，你将会看到compony字段的名字可能与你预想的有些不同，它是ens_jobeetbundle_jobtype[company]。为了让这些看起来更简洁，让我们将格式更改为job[%s]的形式。 因此我们需要将 JobType 类的末尾的getName()方法替换为如下代码 ︰

```php
// src/Ens/JobeetBundle/Form/JobType.php
// ...
 
public function getName()
{
  return 'job';
}
```

之后再打开浏览器你就会发现compony的名称变成了job[company]。现在让我们来实际操♂练♂一♂番

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function testJobForm()
{
  $client = static::createClient();
 
  $crawler = $client->request('GET', '/job/new');
  $this->assertEquals('Ens\JobeetBundle\Controller\JobController::newAction', $client->getRequest()->attributes->get('_controller'));
 
  $form = $crawler->selectButton('Preview your job')->form(array(
    'job[company]'      => 'Sensio Labs',
    'job[url]'          => 'http://www.sensio.com/',
    'job[file]'         => __DIR__.'/../../../../../web/bundles/ensjobeet/images/sensio-labs.gif',
    'job[position]'     => 'Developer',
    'job[location]'     => 'Atlanta, USA',
    'job[description]'  => 'You will work with symfony to develop websites for our customers.',
    'job[how_to_apply]' => 'Send me an email',
    'job[email]'        => 'for.a.job@example.com',
    'job[is_public]'    => false,
  ));
 
  $client->submit($form);
  $this->assertEquals('Ens\JobeetBundle\Controller\JobController::createAction', $client->getRequest()->attributes->get('_controller'));
}
```

浏览器也会模拟文件上传---如果你给定要上传文件的绝对路径

之后提交表单我们可以检查create action是否成功执行。

## 测试表单跳转

如果表单验证被通过，那么职位将会被创建，并且用户会自动跳转到预览页面：

```php
$client->followRedirect();
$this->assertEquals('Ens\JobeetBundle\Controller\JobController::previewAction', $client->getRequest()->attributes->get('_controller'));
```

## 测试数据库记录

最终我们需要测试职位已经在数据库中被成功创建，并且检查is_activated字段是否为false---因为用户并没有将它发布出来。

```php
$kernel = static::createKernel();
$kernel->boot();
$em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
$query = $em->createQuery('SELECT count(j.id) from EnsJobeetBundle:Job j WHERE j.location = :location AND j.is_activated IS NULL AND j.is_public = 0');
$query->setParameter('location', 'Atlanta, USA');
$this->assertTrue(0 < $query->getSingleScalarResult());
```

## 错误测试

正常数据时，职位表单的创建会如我们所期望的那样验证输入的数据并保存成功，那么让我们来添加一些非法数值，从而测试一下这个验证是否有效。

```php
$crawler = $client->request('GET', '/job/new');
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

```

现在我们需要在职位预览页测试管理操作栏。当一份职位尚未激活时，你可以编辑、 删除或发布该职位。这次需要测试三个action，为此我们首先需要创建一个职位。但这又需要大量的ctrl-c和ctrl-v，记得之前我们说过的吗？重复的代码并不是好习惯。所以让我们在JobControllerTest添加一个职位创建方法。

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function createJob($values = array())
{
  $client = static::createClient();
  $crawler = $client->request('GET', '/job/new');
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
 
  return $client;
}
```

createJob()方法创建一个职位信息，完成后并进行跳转。你也可以通过给定一个数组来修改默认值。

如今测试publish action 变得如此简单：

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function testPublishJob()
{
  $client = $this->createJob(array('job[position]' => 'FOO1'));
  $crawler = $client->getCrawler();
  $form = $crawler->selectButton('Publish')->form();
  $client->submit($form);
 
  $kernel = static::createKernel();
  $kernel->boot();
  $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
  $query = $em->createQuery('SELECT count(j.id) from EnsJobeetBundle:Job j WHERE j.position = :position AND j.is_activated = 1');
  $query->setParameter('position', 'FOO1');
  $this->assertTrue(0 < $query->getSingleScalarResult());
}
```

测试删除也是类似的：

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function testDeleteJob()
{
  $client = $this->createJob(array('job[position]' => 'FOO2'));
  $crawler = $client->getCrawler();
  $form = $crawler->selectButton('Delete')->form();
  $client->submit($form);
 
  $kernel = static::createKernel();
  $kernel->boot();
  $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
  $query = $em->createQuery('SELECT count(j.id) from EnsJobeetBundle:Job j WHERE j.position = :position');
  $query->setParameter('position', 'FOO2');
  $this->assertTrue(0 == $query->getSingleScalarResult());
}
```

## 测试是一种保障机制

当一份职位信息被发布以后，你将不能做任何的修改，尽管此时编辑按钮并不显示在预览页上。让我们来添加一些测试看看是否能满足这些需求。

首先，将为createJob()方法添加一个参数使其能自动发布的职位信息。并添加一个getJobByPosition()，用来返回一份工作的职位字段(position)。

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function createJob($values = array(), $publish = false)
{
  $client = static::createClient();
  $crawler = $client->request('GET', '/job/new');
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
 
public function getJobByPosition($position)
{
  $kernel = static::createKernel();
  $kernel->boot();
  $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
 
  $query = $em->createQuery('SELECT j from EnsJobeetBundle:Job j WHERE j.position = :position');
  $query->setParameter('position', $position);
  $query->setMaxResults(1);
  return $query->getSingleResult();
}
```

如果一份职位已经被发布了, 那么打开编辑页面将会返回404代码:

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function testEditJob()
{
  $client = $this->createJob(array('job[position]' => 'FOO3'), true);
  $crawler = $client->getCrawler();
  $crawler = $client->request('GET', sprintf('/job/%s/edit', $this->getJobByPosition('FOO3')->getToken()));
  $this->assertTrue(404 === $client->getResponse()->getStatusCode());
}
```

但如果你运行这个测试，你将不会得到预期的结果，因为我们忘了昨天实现这个功能。所以你是不是觉得写一个测试非常的重要？它能帮助你发现所有的边界值，从而发现并修复BUG。

对这个BUG的修复也很简单，如果这份工作已经发布，我们只需要转发到404页面即可 ︰

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function editAction($token)
{
  $em = $this->getDoctrine()->getManager();
 
  $entity = $em->getRepository('EnsJobeetBundle:Job')->findOneByToken($token);
 
  if (!$entity) {
    throw $this->createNotFoundException('Unable to find Job entity.');
  }
 
  if ($entity->getIsActivated()) {
    throw $this->createNotFoundException('Job is activated and cannot be edited.');
  }
 
  // ...
}
```

## 回到未来

当一份职位信息在发布到期前的五天或者已经到期了，用户可以从当前日期开始再次延长30天。

要在浏览器中测试这个需求其实并不容易，因为到期时间是在职位被创建之后自动产生的。所以当你打开新创建的职位页面时，延长日期的操作并不会出现。当然你能通过一些hack手段修改数据库的到期日期字段或者更改模板使得其操作链接总是存在，但这复杂而且易错。你或许已经猜到了，我们可以编写一些测试案例来帮助我们完成这件事。

与之前一样，我们需要为extend方法添加路由

```yaml
#src/Ens/JobeetBundle/Resources/config/routing/job.yml
#...
 
ens_job_extend:
    pattern:  /{token}/extend
    defaults: { _controller: "EnsJobeetBundle:Job:extend" }
    requirements: { _method: post }
```

然后替换Extend为一个表单，具体如下：

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/admin.html.twig -->
<!-- ... -->
 
{% if job.expiresSoon %}
  <form action="{{ path('ens_job_extend', { 'token': job.token }) }}" method="post">
    {{ form_widget(extend_form) }}
    <button type="submit">Extend</button> for another 30 days
  </form>
{% endif %}
 
<!-- ... -->
```

接着添加extend action和extend 表单：

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function extendAction($token)
{
  $form = $this->createExtendForm($token);
  $request = $this->getRequest();
 
  $form->bindRequest($request);
 
  if ($form->isValid()) {
    $em = $this->getDoctrine()->getManager();
    $entity = $em->getRepository('EnsJobeetBundle:Job')->findOneByToken($token);
 
    if (!$entity) {
      throw $this->createNotFoundException('Unable to find Job entity.');
    }
 
    if (!$entity->extend()) {
      throw $this->createNotFoundException('Unable to find extend the Job.');
    }
 
    $em->persist($entity);
    $em->flush();
 
    $this->get('session')->setFlash('notice', sprintf('Your job validity has been extended until %s.', $entity->getExpiresAt()->format('m/d/Y')));
  }
 
  return $this->redirect($this->generateUrl('ens_job_preview', array(
    'company' => $entity->getCompanySlug(),
    'location' => $entity->getLocationSlug(),
    'token' => $entity->getToken(),
    'position' => $entity->getPositionSlug()
  )));
}
 
private function createExtendForm($token)
{
  return $this->createFormBuilder(array('token' => $token))
    ->add('token', 'hidden')
    ->getForm()
  ;
}
```

除此之外，我们要将extend 表单添加到 preview action中

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function previewAction($token)
{
  $em = $this->getDoctrine()->getManager();
 
  $entity = $em->getRepository('EnsJobeetBundle:Job')->findOneByToken($token);
 
  if (!$entity) {
    throw $this->createNotFoundException('Unable to find Job entity.');
  }
 
  $deleteForm = $this->createDeleteForm($entity->getId());
  $publishForm = $this->createPublishForm($entity->getToken());
  $extendForm = $this->createExtendForm($entity->getToken());
 
  return $this->render('EnsJobeetBundle:Job:show.html.twig', array(
    'entity'      => $entity,
    'delete_form' => $deleteForm->createView(),
    'publish_form' => $publishForm->createView(),
    'extend_form' => $extendForm->createView(),
  ));
}
```

在job entity中添加一个extend()方法，如果该职位已经延期过则返回true，否则返回false。

```php
// src/Ens/Jobeetbundle/Entity/Job.php
// ...
 
public function extend()
{
  if (!$this->expiresSoon())
  {
    return false;
  }
 
  $this->expires_at = new \DateTime(date('Y-m-d H:i:s', time() + 86400 * 30));
 
  return true;
}
```

最后，我们来添加测试：

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTest.php
// ...
 
public function testExtendJob()
{
  // A job validity cannot be extended before the job expires soon
  $client = $this->createJob(array('job[position]' => 'FOO4'), true);
  $crawler = $client->getCrawler();
  $this->assertTrue($crawler->filter('input[type=submit]:contains("Extend")')->count() == 0);
 
  // A job validity can be extended when the job expires soon
 
  // Create a new FOO5 job
  $client = $this->createJob(array('job[position]' => 'FOO5'), true);
  // Get the job and change the expire date to today
  $kernel = static::createKernel();
  $kernel->boot();
  $em = $kernel->getContainer()->get('doctrine.orm.entity_manager');
  $job = $em->getRepository('EnsJobeetBundle:Job')->findOneByPosition('FOO5');
  $job->setExpiresAt(new \DateTime());
  $em->flush();
  // Go to the preview page and extend the job
  $crawler = $client->request('GET', sprintf('/job/%s/%s/%s/%s', $job->getCompanySlug(), $job->getLocationSlug(), $job->getToken(), $job->getPositionSlug()));
  $crawler = $client->getCrawler();
  $form = $crawler->selectButton('Extend')->form();
  $client->submit($form);
  // Reload the job from db
  $job = $this->getJobByPosition('FOO5');
  // Check the expiration date
  $this->assertTrue($job->getExpiresAt()->format('y/m/d') == date('y/m/d', time() + 86400 * 30));
}
```
## 维护任务

尽管Symfony是一个 web 开发框架，但它配备了一个强大的命令行工具。在之前的几天中你已经使用它为你的应用程序Bundle创建了默认目录结构，并用它生成了各种Model文件。而且你还可以相当容易的为其添加新的命令。

当用户创建一份职位时，他必须手动激活它从而使其在网上发布。但并非每份职位用户都会去激活，或许他因为这样或那样的理由放弃了发布，那么随着时间的推移，数据库中这种无用的职位信息数量将会不断的增长，现在让我们来创建一个命令用来删除这些无用信息。到时你只需要在计划任务中定时运行这个命令即可。

```php
// src/Ens/JobeetBundle/Command/JobeetCleanupCommand.php
 
namespace Ens\JobeetBundle\Command;
 
use Symfony\Bundle\FrameworkBundle\Command\ContainerAwareCommand;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Ens\JobeetBundle\Entity\Job;
 
class JobeetCleanupCommand extends ContainerAwareCommand {
 
  protected function configure()
  {
    $this
      ->setName('ens:jobeet:cleanup')
      ->setDescription('Cleanup Jobeet database')
      ->addArgument('days', InputArgument::OPTIONAL, 'The email', 90)
    ;
  }
 
  protected function execute(InputInterface $input, OutputInterface $output)
  {
    $days = $input->getArgument('days');
 
    $em = $this->getContainer()->get('doctrine')->getManager();
    $nb = $em->getRepository('EnsJobeetBundle:Job')->cleanup($days);
 
    $output->writeln(sprintf('Removed %d stale jobs', $nb));
  }
}
```

我们需要在JobRepository类里面添加一个cleanup方法：

```php
// src/Ens/JobeetBundle/Repository/JobRepository.php
// ...
 
public function cleanup($days)
{
  $query = $this->createQueryBuilder('j')
    ->delete()
    ->where('j.is_activated IS NULL')
    ->andWhere('j.created_at < :created_at')     ->setParameter('created_at',  date('Y-m-d', time() - 86400 * $days))
    ->getQuery();
 
  return $query->execute();
}
```

现在你可以打开项目目录执行以下命令：

> **php app/console ens:jobeet:cleanup**

或者用如下命令删除最近10天内的

> **php app/console ens:jobeet:cleanup 10**
