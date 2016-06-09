# Symfony2 Jobeet Day 13: 安全

## 应用程序的安全性

应用程序的安全性其目标是防止用户访问他本不应该访问的资源。其分为两步，第一步安全认证系统会通过用户提交的某些数据来对用户进行身份认证；一旦系统确认了你的身份，那么第二步确定您是否可以访问给定资源，称之为授权（它会检查你是否有权限来执行某些操作）

你可以在app/config/security.yml文件来对应用程序的安全组件进行配置。让我们在security.yml里添加如下代码进行安全配置：

> ~~# app/config/security.yml~~  

> ~~security:~~  
> &ensp;&ensp;~~firewalls:~~  
> &ensp;&ensp;&ensp;&ensp;~~secured_area:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~pattern:    ^/~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~anonymous: ~~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~form_login:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~login_path:  /login~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~check_path:  /login_check~~  

> &ensp;&ensp;~~access_control:~~  
> &ensp;&ensp;&ensp;&ensp;~~- { path: ^/admin, roles: ROLE_ADMIN }~~  

> &ensp;&ensp;~~providers:~~  
> &ensp;&ensp;&ensp;&ensp;~~in_memory:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~users:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~admin: { password: adminpass, roles: 'ROLE_ADMIN' }~~  

> &ensp;&ensp;~~encoders:~~  
> &ensp;&ensp;&ensp;&ensp;~~Symfony\Component\Security\Core\User\User: plaintext~~  


```yaml
#app/config/security.yml

security:
    role_hierarchy:
        ROLE_ADMIN:       ROLE_USER
        ROLE_SUPER_ADMIN: [ROLE_USER, ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
 
    firewalls:
        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false
 
        secured_area:
            pattern:    ^/
            anonymous: ~
            form_login:
                login_path:  /login
                check_path:  /login_check
                default_target_path: ens_jobeet_homepage
 
    access_control:
        - { path: ^/admin, roles: ROLE_ADMIN }
 
    providers:
        in_memory:
            memory:
                users:
                    admin: { password: adminpass, roles: 'ROLE_ADMIN' }
 
    encoders:
        SymfonyComponentSecurityCoreUserUser: plaintext
```


这个配置将会对网站所有/admin开头的URL进行安全认证，并只允许ROLE_ADMIN角色的用户来访问。在此示例中配置文件中定义管理员用户和密码将不会被进行编码。 

为了对用户的身份进行验证，我们需要实现一个经典的登录表单。首先，我们需要创建两个路由 ︰ 一个用来显示登录表单(/login) 和一个用来处理登录表单的提交 (/login_check):

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml
 
login:
    pattern:   /login
    defaults:  { _controller: EnsJobeetBundle:Default:login }
login_check:
    pattern:   /login_check
 
#...
```

我们并不需要在控制器中实现/login_check，因为firewall会对这个URL进行自动捕获和提交。它虽然是可选的，但十分有用，我们可以创建一个路由，以便它可以在下面生成的登录表单中使用该URL。

下面让我们来创建一个action用以展示这个表单

```php
// src/Ens/JobeetBundle/Controller/DefaultController.php
 
namespace Ens\JobeetBundle\Controller;
 
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\Security\Core\SecurityContext;
 
class DefaultController extends Controller
{
    // ...
 
    public function loginAction()
    {
        $request = $this->getRequest();
        $session = $request->getSession();
 
        // get the login error if there is one
        if ($request->attributes->has(SecurityContext::AUTHENTICATION_ERROR)) {
            $error = $request->attributes->get(SecurityContext::AUTHENTICATION_ERROR);
        } else {
            $error = $session->get(SecurityContext::AUTHENTICATION_ERROR);
            $session->remove(SecurityContext::AUTHENTICATION_ERROR);
        }
 
        return $this->render('EnsJobeetBundle:Default:login.html.twig', array(
            // last username entered by the user
            'last_username' => $session->get(SecurityContext::LAST_USERNAME),
            'error'         => $error,
        ));
    }
}
```

当用户提交表单时，安全系统会自动处理表单并提交给你。如果用户提交了一个无效的用户名或密码，这个action会读取表单提交的错误信息，然后传递给安全系统，由它显示给用户。你唯一所要作的工作仅仅只是显示登录表单和检查任何可能发生的登录错误，而安全系统本身会检查提交的用户名和密码来进行身份验证。

最后，让我们创建相应的模板 ︰

```html
<!-- src/Ens/JobeetBundle/Resources/views/Default/login.html.twig -->
 
{% if error %}
    <div>{{ error.message }}</div>
{% endif %}
 
<form action="{{ path('login_check') }}" method="post">
    <label for="username">Username:</label>
    <input type="text" id="username" name="_username" value="{{ last_username }}" />
 
    <label for="password">Password:</label>
    <input type="password" id="password" name="_password" />
 
    <button type="submit">login</button>
</form>
```

现在，如果你尝试访问*http://jobeet.local/app_dev.php/admin/dashboard*，首先会显示一个登录页面，而你则必须输入在security.yml定义的用户名和密码 (admin/adminpass) 才能进入Jobeet的管理部分。

## 用户来源(User Providers)

在身份验证期间，用户提交一组凭据 （通常是用户名和密码），身份验证系统对该组凭据进行验证匹配。那么用来匹配的用户数据从哪里来呢？

在 Symfony2，用户可以来自任何地方 --- 一个配置文件、 数据库表、 web service或其他任何你能想到的地方。所以任何能够提供一个或多个用户给身份验证系统的事物，我们就将其称之为"User Providers"。Symfony2中有两个标配的User Providers︰分别是配置文件和数据库表。

在上面的例子中，我们使用了第一种方式︰在配置文件中指定用户。


> ~~providers:~~  
> &ensp;&ensp;~~in_memory:~~  
> &ensp;&ensp;&ensp;&ensp;~~users:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;~~admin: { password: adminpass, roles: 'ROLE_ADMIN' }~~  


```yaml
providers:
    in_memory:
        memory:
            users:
                admin: { password: adminpass, roles: 'ROLE_ADMIN' }
```

但是，你通常会希望用户存储在数据库表中。要做到这一点，我们需要向jobeet 数据库中添加新的user表。首先让我们通过orm创建这个新表:

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/User.orm.yml
 
Ens\JobeetBundle\Entity\User:
  type: entity
  table: user
  id:
    id:
      type: integer
      generator: { strategy: AUTO }
  fields:
    username:
      type: string
      length: 255
    password:
      type: string
      length: 255
```

接下来向之前所做过的那样生成entity和数据库表：

> **php app/console doctrine:generate:entities EnsJobeetBundle**

> **php app/console doctrine:schema:update --force**

对于新生成的user类来说，唯一的要求就是它需要实现UserInterface接口。这就意味着对你来说"用户"的概念可以是任何东西---只要它实现了此接口。让我们打开User.php文件并编辑它，如下所示 ︰

```php
// src Ens/JobeetBundle/Entity/User.php
 
namespace Ens\JobeetBundle\Entity;
 
use Symfony\Component\Security\Core\User\UserInterface;
use Doctrine\ORM\Mapping as ORM;
 
class User implements UserInterface
{
    private $id;
 
    private $username;
 
    private $password;
 
    public function getId()
    {
        return $this->id;
    }
 
    public function setUsername($username)
    {
        $this->username = $username;
    }
 
    public function getUsername()
    {
        return $this->username;
    }
 
    public function setPassword($password)
    {
        $this->password = $password;
    }
 
    public function getPassword()
    {
        return $this->password;
    }
 
    public function getRoles()
    {
        return array('ROLE_ADMIN');
    }
 
    public function getSalt()
    {
        return null;
    }
 
    public function eraseCredentials()
    {
 
    }
 
    public function equals(UserInterface $user)
    {
        return $user->getUsername() == $this->getUsername();
    }
}
```

我们为生成的entity添加UserInterface所需的方法：getRoles, getSalt, eraseCredentials 和 equals。

接下来配置一个 entity user provider ，使其指向user类：

```yaml
# app/config/security.yml
# ...
 
    providers:
        main:
            entity: { class: Ens\JobeetBundle\Entity\User, property: username }
 
    encoders:
        Ens\JobeetBundle\Entity\User: sha512
```

我们也改变了编码设置，使我们能在新的user类中使用`sha512`算法来对密码进行加密。

现在一切都都已就绪，但我们还缺少用户。为此我们将创建一个新的Symfony命令来添加我们的第一个用户︰

```php
// src/Ens/JobeetBundle/Command/JobeetUsersCommand.php
 
namespace Ens\JobeetBundle\Command;
 
use Symfony\Bundle\FrameworkBundle\Command\ContainerAwareCommand;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Ens\JobeetBundle\Entity\User;
 
class JobeetUsersCommand extends ContainerAwareCommand
{
    protected function configure()
    {
        $this
            ->setName('ens:jobeet:users')
            ->setDescription('Add Jobeet users')
            ->addArgument('username', InputArgument::REQUIRED, 'The username')
            ->addArgument('password', InputArgument::REQUIRED, 'The password')
        ;
    }
 
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $username = $input->getArgument('username');
        $password = $input->getArgument('password');
 
        $em = $this->getContainer()->get('doctrine')->getManager();
 
        $user = new User();
        $user->setUsername($username);
        // encode the password
        $factory = $this->getContainer()->get('security.encoder_factory');
        $encoder = $factory->getEncoder($user);
        $encodedPassword = $encoder->encodePassword($password, $user->getSalt());
        $user->setPassword($encodedPassword);
        $em->persist($user);
        $em->flush();
 
        $output->writeln(sprintf('Added %s user with password %s', $username, $password));
    }
}
```

现在我们使用以下命令来添加一个新的用户：

> **php app/console ens:jobeet:users admin admin**

这将创建一个用户名密码均为admin的用户，你可以使用其进行登录试试效果。


## 登出

firewall将会自动处理注销操作，所以你所要做的仅仅只是一些参数的配置。

```yaml
#app/config/security.yml
security:
    firewalls:
        secured_area:
            #...
            logout:
                path:   /logout
                target: /
    #...
```

一旦这个firewall配置生效之后，用户就可以通过访问/logout （或任何你配置的路径）解除对当前用户的身份验证。然后跳转到主页 （或其他你在target字段所指定的路径）。

你不需要在控制器中实现/logout因为firewall已经完全处理好了。然而，你还是需要配置一条路由，以便你可以使用它来生成 URL:

```yaml
# src/Ens/JobeetBundle/Resources/config/routing.yml
# ...
 
logout:
    pattern:   /logout
 
# ...
```

剩下了要做的就是将这个链接添加到我们的管理部分。为此我们将重写从 SonataAdminBundle user_block.html.twig模板，在`app/Resources/SonataAdminBundle/views/Core`文件夹下新建一个 user_block.html.twig 文件：

```html
<!-- app/Resources/SonataAdminBundle/views/Core/user_block.html.twig -->
 
{% block user_block %}<a href="{{ path('logout') }}">Logout</a>{% endblock %}
```

现在，如果您尝试进入管理部分，将要求您提供用户名和密码，然后在右上角将显示注销链接。

## 用户会话

Symfony2提供了一个优秀的会话对象，可以帮助你存储用户有关的请求信息。默认情况下，Symfony2 通过使用原生的PHP session 将属性存储在cookie 中。

你可以在控制器中轻松的存取session中的数据

```php
$session = $this->getRequest()->getSession();
 
// store an attribute for reuse during a later user request
$session->set('foo', 'bar');
 
// in another controller for another request
$foo = $session->get('foo');
```

不幸的是我们在当初设计的时候并没有什么地方需要使用这个功能，为此让我们来添加一个新的需求：为了更方便的浏览职位信息，我们将会在职位详情页的菜单上添加用户最后浏览的三份工作的链接。

当用户访问一个职位详情页时，会将展示的job对象存入到历史记录的session中

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function showAction($id)
{
    $em = $this->getDoctrine()->getManager();
 
    $entity = $em->getRepository('EnsJobeetBundle:Job')->getActiveJob($id);
 
    if (!$entity) {
        throw $this->createNotFoundException('Unable to find Job entity.');
    }
 
    $session = $this->getRequest()->getSession();
 
    // fetch jobs already stored in the job history
    $jobs = $session->get('job_history', array());
 
    // store the job as an array so we can put it in the session and avoid entity serialize errors
    $job = array('id' => $entity->getId(), 'position' =>$entity->getPosition(), 'company' => $entity->getCompany(), 'companyslug' => $entity->getCompanySlug(), 'locationslug' => $entity->getLocationSlug(), 'positionslug' => $entity->getPositionSlug());
 
    if (!in_array($job, $jobs)) {
        // add the current job at the beginning of the array
        array_unshift($jobs, $job);
 
        // store the new job history back into the session
        $session->set('job_history', array_slice($jobs, 0, 3));
    }
 
    $deleteForm = $this->createDeleteForm($id);
 
    return $this->render('EnsJobeetBundle:Job:show.html.twig', array(
        'entity'      => $entity,
        'delete_form' => $deleteForm->createView(),
    ));
}
```

将如下代码添加到布局文件的#content div之前：

```html
<!-- src/End/JobeetBundle/Resources/views/layout.html.twig -->
<!-- ... -->
 
<div id="job_history">
    Recent viewed jobs:
    <ul>
        {% for job in app.session.get('job_history') %}
            <li>
                <a href="{{ path('ens_job_show', { 'id': job.id, 'company': job.companyslug, 'location': job.locationslug, 'position': job.positionslug }) }}">{{ job.position }} - {{ job.company }}</a>
            </li>
        {% endfor %}
    </ul>
</div>
 
<div id="content">
 
<!-- ... -->

```

## Flash Messages

Flash messages是一个简短的消息文本，你可以将它存到session中用于额外的请求。这对表单处理来说是很有用的：当你想要重定向并有一个特殊的信息需要显示在下一个请求时。之前当我们发布一份职位信息的时候已经在项目中使用过Flash messages了︰

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function publishAction($token)
{
    // ...
 
    $this->get('session')->setFlash('notice', 'Your job is now online for 30 days.');
 
    // ...
}
```

setFlash函数的第一个参数是个键值，用来标识这个消息文本，第二个是要显示的消息文本。我们可以定义任何你想要的类型标识符，这其中notice 和error是最常见的两种。

为了显示flash messages我们需要在模板文件中包含它。打开layout.html.twig文件，添加如下：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->
<!-- ... -->
 
{% if app.session.hasFlash('notice') %}
    <div>
        {{ app.session.flash('notice') }}
    </div>
{% endif %}
 
<!-- ... -->
```