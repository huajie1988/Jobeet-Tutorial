# Symfony2 Jobeet Day 3: 数据模型

## 关系模型
前一天我们主要讨论了这个项目的主体对象：职位、合作伙伴和分类。
如下是实体关系图：

![实体关系图](http://www.ens.ro/wp-content/uploads/2012/03/diagram.png)
 
除了在前一天提到的字段外，我们还增加了created_at和updated_at字段。我们将配置Symfony2自动设定它们的值当对象保存或更新的时候。

数据库配置
接下来我们将通过[Doctrine ORM](http://www.doctrine-project.org/projects/orm.html)来创建jobs, affiliates 和 categories 三张表。
首先通过修改`app/config/parameters.ini`文件来修改数据库的连接参数，如下（假定本教程使用MySQL）：

```yaml
#app/config/parameters.ini
[parameters]
    database_driver   = pdo_mysql
    database_host     = localhost
    database_name     = jobeet
    database_user     = root
    database_password = password
```

> *译者注：2.1以后改为app/config/parameters.yml文件*

现在Doctrine 已经连接了数据库并且使用如下命令创建数据库（如果你还没有创建的话）：

> **php app/console doctrine:database:create**

## 创建数据表
接下来我们将通过Doctrine创建“metadata”文件，它用来描述我们的对象将如何存储在数据库中：

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Category.orm.yml
Ens\JobeetBundle\Entity\Category:
  type: entity
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
  oneToMany:
    jobs:
      targetEntity: Job
      mappedBy: category
    category_affiliates:
      targetEntity: CategoryAffiliate
      mappedBy: category
```



```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml
Ens\JobeetBundle\Entity\Job:
  type: entity
  table: job
  id:
    id:
      type: integer
      generator: { strategy: AUTO }
  fields:
    type:
      type: string
      length: 255
      nullable: true
    company:
      type: string
      length: 255
    logo:
      type: string
      length: 255
      nullable: true
    url:
      type: string
      length: 255
      nullable: true
    position:
      type: string
      length: 255
    location:
      type: string
      length: 255
    description:
      type: text
    how_to_apply:
      type: text
    token:
      type: string
      length: 255
      unique: true
    is_public:
      type: boolean
      nullable: true
    is_activated:
      type: boolean
      nullable: true
    email:
      type: string
      length: 255
    expires_at:
      type: datetime
    created_at:
      type: datetime
    updated_at:
      type: datetime
      nullable: true
  manyToOne:
    category:
      targetEntity: Category
      inversedBy: jobs
      joinColumn:
        name: category_id
        referencedColumnName: id
  lifecycleCallbacks:
    prePersist: [ setCreatedAtValue ]
    preUpdate: [ setUpdatedAtValue ]
```

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/Affiliate.orm.yml
Ens\JobeetBundle\Entity\Affiliate:
  type: entity
  table: affiliate
  id:
    id:
      type: integer
      generator: { strategy: AUTO }
  fields:
    url:
      type: string
      length: 255
    email:
      type: string
      length: 255
      unique: true
    token:
      type: string
      length: 255
    created_at:
      type: datetime
  oneToMany:
    category_affiliates:
      targetEntity: CategoryAffiliate
      mappedBy: affiliate
  lifecycleCallbacks:
    prePersist: [ setCreatedAtValue ]
```

```yaml
#src/Ens/JobeetBundle/Resources/config/doctrine/CategoryAffiliate.orm.yml
Ens\JobeetBundle\Entity\CategoryAffiliate:
  type: entity
  table: category_affiliate
  id:
    id:
      type: integer
      generator: { strategy: AUTO }
  manyToOne:
    category:
      targetEntity: Category
      inversedBy: category_affiliates
      joinColumn:
        name: category_id
        referencedColumnName: id
    affiliate:
      targetEntity: Affiliate
      inversedBy: category_affiliates
      joinColumn:
        name: affiliate_id
        referencedColumnName: id
```


## ORM:
现在我们可以通过如下命令在Doctrine 中创建我们的对象：

> **php app/console doctrine:generate:entities EnsJobeetBundle**

> \> Generating entities for bundle "EnsJobeetBundle"
  \> backing up Job.php to Job.php~
  \> generating Ens\JobeetBundle\Entity\Job
  \> backing up Category.php to Category.php~
  \> generating Ens\JobeetBundle\Entity\Category
  \> backing up CategoryAffiliate.php to CategoryAffiliate.php~
  \> generating Ens\JobeetBundle\Entity\CategoryAffiliate
  \> backing up Affiliate.php to Affiliate.php~
  \> generating Ens\JobeetBundle\Entity\Affiliate

如果此时你查看EnsJobeetBundle下的**Entity**目录，就会发现有新生成的Affiliate.php, Category.php, CategoryAffiliate.php 和 Job.php
打开Job.php并设置created_at和的updated_at如下值：

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
public function setCreatedAtValue()
{
  if(!$this->getCreatedAt())
  {
    $this->created_at = new \DateTime();
  }
}
// ...
public function setUpdatedAtValue()
{
  $this->updated_at = new \DateTime();
}
```

在Affiliate.php中也做如下修改：

```php
// src/Ens/JobeetBundle/Entity/Affiliate.php
// ...
public function setCreatedAtValue()
{
  $this->created_at = new \DateTime();
}
```

这将使Doctrine在保存或更新对象时设置的created_at和的updated_at值。
接下来我们通过Doctrine来创建数据库表（或更新它们，以反映我们的设置）：

> **php app/console doctrine:schema:update --force**

*此方法应该只在开发过程中使用.如果将其用于你的生产环境将会很糟糕,更多请详见 [Doctrine migrations](http://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html)*


## Annotation方式创建数据库

本节为译者补充内容，annotation方式相比较传统的yaml格式的定义方式更灵活，与代码的结合也更紧密，如果初学者也可先跳过。


首先创建Ens\JobeetBundle\Entity\Category.php 文件，然后如下填写

```php
// src/JobeetBundle/Entity/Category.php
namespace JobeetBundle\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
  * @ORM\Entity(repositoryClass=” CategoryRepository”)
  * @ORM\Table(name=" category ")
  */
class Category
{
    /**
      * @ORM\Column(type="integer")
      * @ORM\Id
      * @ORM\GeneratedValue(strategy="AUTO")
      */
    protected $id;

    /**
      * @ORM\Column(type="string", length=100,unique=”true”)
      */
    protected $name;

}
```

如果有一对多或者多对多等关系时如下定义

```php
// src/JobeetBundle/Entity/Category.php
//…
use Doctrine\ORM\OneToMany;
//…
/**
  * @OneToMany(“targetEntity”=Job”, mappedBy=”category”)
  */

protected $jobs


// src/JobeetBundle/Entity/Job.php
//…
use Doctrine\ORM\ManyToOne;
use Doctrine\ORM\JoinColumn;
//…
/**
  * @ManyToOne(“targetEntity”=Category”, inversedBy=”jobs”)
  * @JoinColumn(name=”category_id”,referencedColumnName=“id”)
  */

protected $category
```


**targetEntity**代表目标表，**inversedBy**或者**mappedBy**则代表那个表中entity所定义的字段。
**JoinColumn**中那么表示本表的字段，**referencedColumnName**表示外键对应的另一个表的字段
简单说*外键* **JoinColumn** 所定义的地方是**inversedBy**而另一边是**mappedBy**

如要进行生命管理可参照下面所示：

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
/**
  * @ORM\Entity(repositoryClass=”JobRepository”) 
* @ORM\Table(name="job")
* @ORM\HasLifecycleCallbacks
*/
class Job
{
// ...
	 /**
     * @ORM\PerPersist()
     */
    public function setCreatedAtValue()
{
  		if(!$this->getCreatedAt())
  		{
    		$this->created_at = new \DateTime();
  		}
}

}

```

或者

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
/**
  * @ORM\Entity(repositoryClass=”JobRepository”) 
  * @ORM\Table(name="job")
  * @ORM\HasLifecycleCallbacks
  */
class Job
{
// ...
    public function PerPersist()
{
  		if(!$this->getCreatedAt())
  		{
    		$this->setCreatedAt(new \DateTime());
  		}
		$this->setUpdatedAt(new \DateTime());
	}
}

```

定义完之后输入
> **php app/console generate:doctrine:entities JobeetBundle**

> **php app/console doctrine:schema:update --force**

## 数据初始化:
表虽然已经建好了，但暂时其中还没有数据。对于任何Web应用，一般来说有三种类型的数据：初始数据（这是应用正常工作时必要的数据，在我们的例子中为初始化分类和管理员用户数据），测试数据（应用程序进行测试时需要的数据）和用户数据（应用程序在正常使用生命周期中由用户发布的数据）。

我们将使用[DoctrineFixturesBundle](http://symfony.com/doc/current/bundles/DoctrineFixturesBundle/index.html)为数据库填充初始数据。要使用这个包，我们必须按照下面的步骤：

~~1、	将如下代码添加到你的deps 文件~~
> ~~[doctrine-fixtures]~~  
> ~~&ensp;&ensp;git=http://github.com/doctrine/data-fixtures.git~~  
 
> ~~[DoctrineFixturesBundle]~~  
> ~~&ensp;&ensp;git=http://github.com/doctrine/DoctrineFixturesBundle.git~~  
> ~~&ensp;&ensp;target=/bundles/Symfony/Bundle/DoctrineFixturesBundle~~  
> ~~&ensp;&ensp;version=origin/2.0~~  

~~2、	升级vendor文件夹~~

> **~~php bin/vendors install –reinstall~~**

~~3、	将*Doctrine\Common\DataFixtures*命名空间注册到*app/autoload.php*里~~


> ~~// ...~~  
> ~~$loader->registerNamespaces(array(~~  
> &ensp;&ensp;~~// ...~~  
> &ensp;&ensp;~~'Doctrine\\Common\\DataFixtures' => \__DIR__.'/../vendor/doctrine-fixtures/lib',~~  
> &ensp;&ensp;~~'Doctrine\\Common' => \__DIR__.'/../vendor/doctrine-common/lib',~~  
> &ensp;&ensp;~~// ...~~  
> ~~));~~  


~~4、	将DoctrineFixturesBundle注册到app/AppKernel.php中~~


> ~~// ...~~  
> ~~public function registerBundles()~~  
> ~~{~~  
> &ensp;&ensp;~~$bundles = array(~~  
> &ensp;&ensp;~~// ...~~  
> &ensp;&ensp;&ensp;&ensp;~~new Symfony\Bundle\DoctrineFixturesBundle\DoctrineFixturesBundle(),~~  
> &ensp;&ensp;~~// ...~~  
> &ensp;&ensp;~~);~~  
> ~~// ...~~  
> ~~}~~  


使用Composer安装的方式：

1、在composer.json的require节添加如下：

```json
// ...
    "require": {
        // ...
        "doctrine/doctrine-fixtures-bundle": "dev-master",
        "doctrine/data-fixtures": "dev-master"
    },
 
// ...
```
2、运行update命令：

> **php composer.phar update**

3、将DoctrineFixturesBundle注册到app/AppKernel.php中：

```php
// ...
public function registerBundles()
{
    $bundles = array(
    // ...
        new Symfony\Bundle\DoctrineFixturesBundle\DoctrineFixturesBundle(),
    // ...
    );
  // ...
}
```


> *译者注：现在前三步可以直接用Composer安装完成。不得不说这真是一个跨时代的发明(XD)。且以后的安装方式也与上述表示相同，即原文保留，但以新的方式替代。*

现在我们已经安装好一切了，可以开始着手填充数据了。

```php
// src/Ens/JobeetBundle/DataFixtures/ORM/LoadCategoryData.php
namespace Ens\JobeetBundle\DataFixtures\ORM;
 
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Common\DataFixtures\AbstractFixture;
use Doctrine\Common\DataFixtures\OrderedFixtureInterface;
use Ens\JobeetBundle\Entity\Category;
 
class LoadCategoryData extends AbstractFixture implements OrderedFixtureInterface
{
  public function load(ObjectManager $em)
  {
    $design = new Category();
    $design->setName('Design');
 
    $programming = new Category();
    $programming->setName('Programming');
 
    $manager = new Category();
    $manager->setName('Manager');
 
    $administrator = new Category();
    $administrator->setName('Administrator');
 
    $em->persist($design);
    $em->persist($programming);
    $em->persist($manager);
    $em->persist($administrator);
 
    $em->flush();
 
    $this->addReference('category-design', $design);
    $this->addReference('category-programming', $programming);
    $this->addReference('category-manager', $manager);
    $this->addReference('category-administrator', $administrator);
  }
 
  public function getOrder()
  {
    return 1; // the order in which fixtures will be loaded
  }
}
```

```php
// src/Ens/JobeetBundle/DataFixtures/ORM/LoadJobData.php
namespace Ens\JobeetBundle\DataFixtures\ORM;
 
use Doctrine\Common\Persistence\ObjectManager;
use Doctrine\Common\DataFixtures\AbstractFixture;
use Doctrine\Common\DataFixtures\OrderedFixtureInterface;
use Ens\JobeetBundle\Entity\Job;
 
class LoadJobData extends AbstractFixture implements OrderedFixtureInterface
{
  public function load(ObjectManager $em)
  {
    $job_sensio_labs = new Job();
    $job_sensio_labs->setCategory($em->merge($this->getReference('category-programming')));
    $job_sensio_labs->setType('full-time');
    $job_sensio_labs->setCompany('Sensio Labs');
    $job_sensio_labs->setLogo('sensio-labs.gif');
    $job_sensio_labs->setUrl('http://www.sensiolabs.com/');
    $job_sensio_labs->setPosition('Web Developer');
    $job_sensio_labs->setLocation('Paris, France');
    $job_sensio_labs->setDescription('You\'ve already developed websites with symfony and you want to work with Open-Source technologies. You have a minimum of 3 years experience in web development with PHP or Java and you wish to participate to development of Web 2.0 sites using the best frameworks available.');
    $job_sensio_labs->setHowToApply('Send your resume to fabien.potencier [at] sensio.com');
    $job_sensio_labs->setIsPublic(true);
    $job_sensio_labs->setIsActivated(true);
    $job_sensio_labs->setToken('job_sensio_labs');
    $job_sensio_labs->setEmail('job@example.com');
    $job_sensio_labs->setExpiresAt(new \DateTime('2012-10-10'));
 
    $job_extreme_sensio = new Job();
    $job_extreme_sensio->setCategory($em->merge($this->getReference('category-design')));
    $job_extreme_sensio->setType('part-time');
    $job_extreme_sensio->setCompany('Extreme Sensio');
    $job_extreme_sensio->setLogo('extreme-sensio.gif');
    $job_extreme_sensio->setUrl('http://www.extreme-sensio.com/');
    $job_extreme_sensio->setPosition('Web Designer');
    $job_extreme_sensio->setLocation('Paris, France');
    $job_extreme_sensio->setDescription('Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in.');
    $job_extreme_sensio->setHowToApply('Send your resume to fabien.potencier [at] sensio.com');
    $job_extreme_sensio->setIsPublic(true);
    $job_extreme_sensio->setIsActivated(true);
    $job_extreme_sensio->setToken('job_extreme_sensio');
    $job_extreme_sensio->setEmail('job@example.com');
    $job_extreme_sensio->setExpiresAt(new \DateTime('2012-10-10'));
 
    $em->persist($job_sensio_labs);
    $em->persist($job_extreme_sensio);
 
    $em->flush();
  }
 
  public function getOrder()
  {
    return 2; // the order in which fixtures will be loaded
  }
}
```

当你完成上述步骤可使用doctrine:fixtures:load 进行安装

> **php app/console doctrine:fixtures:load**

*现在检查数据库，你将会看到数据库中已经有数据了其中需要引用两个图像。您可以通过如下网址下载它们（[sensio-labs.gif](http://www.ens.ro/downloads/sensio-labs.gif),[extreme-sensio.gif](http://www.ens.ro/downloads/extreme-sensio.gif)）并把他们放到web/uploads/jobs/目录下。*

## 在浏览器中查看效果
现在让我们使用一些魔法！输入如下命令：

> **php app/console doctrine:generate:crud --entity=EnsJobeetBundle:Job --route-prefix=ens_job --with-write --format=yml**

这将会创建一个新的控制器`src/Ens/JobeetBundle/Controllers/JobController.php`，其中包含列表，创建，编辑和删除工作（及其相应的模板，表单和路由）。
如果要在浏览器中看到这点，我们必须在主目录中导入`src/Ens/JobeetBundle/Recources/config/routing/job.yml`

```yaml
#src/Ens/JobeetBundle/Resources/config/routing.yml

EnsJobeetBundle_job:
    resource: "@EnsJobeetBundle/Resources/config/routing/job.yml"
    prefix: /job
 
EnsJobeetBundle_homepage:
    pattern:  /hello/{name}
    defaults: { _controller: EnsJobeetBundle:Default:index }
```

现在我们要添加一个toString()方法在Category 类中，这样做是为了能让Job表单下拉选择分类

```php
// src/Ens/JobeetBundle/Entity/Category.php
// ...
public function __toString()
{
  return $this->getName();
}
```

并使用如下命令清除缓存：


> **php app/console cache:clear --env=prod**  
> **php app/console cache:clear --env=dev**  

现在你能在浏览器中通过访问*http://jobeet.local/job/* 或*http://jobeet.local/app_dev.php/job/*测试job控制器
 
![职位列表](http://www.ens.ro/wp-content/uploads/2012/03/jobeet_job_list.png)

![职位编辑](http://www.ens.ro/wp-content/uploads/2012/03/jobeet_job_edit-1024x681.png)
 
现在你可以创建和编辑职位。同时可以尝试在必填项留空，或尝试输入无效数据，都是可以正确验证的，这是因为Symfony已经在数据库建立的时候就创建了基本的验证规则

## 反思
这就是我们今天写的所有php代码，虽然少，但我们已经有了可以正常工作的job模块了，并且可以定制和优化。
但是请记住`no PHP code also means no bugs`!
