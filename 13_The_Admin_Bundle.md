# Symfony2 Jobeet Day 12: Admin Bundle

今天是我们在这个教程的第11天，通过之前这段时间的课程我们已经能正常的进行求职和职位发布了。因此也是时候谈谈我们应用程序的管理部分了。今天，得益于Sonata Admin Bundle，我们会为Jobeet定制一个完整的管理界面，而整个过程仅仅花费你不到一个小时的时间。是不是很鹅妹子嘤！

## 安装Admin Bundle

~~1、在deps文件中加入如下段~~

> ~~[SonataCacheBundle]~~  
> ~~&ensp;&ensp;git=http://github.com/sonata-project/SonataCacheBundle.git~~  
> ~~&ensp;&ensp;target=/bundles/Sonata/CacheBundle~~  
> ~~&ensp;&ensp;version=origin/2.0~~  

> ~~[SonataBlockBundle]~~  
> ~~&ensp;&ensp;git=http://github.com/sonata-project/SonataBlockBundle.git~~  
> ~~&ensp;&ensp;target=/bundles/Sonata/BlockBundle~~  
> ~~&ensp;&ensp;version=origin/2.0~~  

> ~~[SonatajQueryBundle]~~  
> ~~&ensp;&ensp;git=http://github.com/sonata-project/SonatajQueryBundle.git~~  
> ~~&ensp;&ensp;target=/bundles/Sonata/jQueryBundle~~  

> ~~[KnpMenu]~~  
> ~~&ensp;&ensp;git=https://github.com/KnpLabs/KnpMenu.git~~  
> ~~&ensp;&ensp;version=v1.1.2~~  

> ~~[KnpMenuBundle]~~  
> ~~&ensp;&ensp;git=https://github.com/KnpLabs/KnpMenuBundle.git~~  
> ~~&ensp;&ensp;target=bundles/Knp/Bundle/MenuBundle~~  
> ~~&ensp;&ensp;version=v1.1.0~~  

> ~~[Exporter]~~  
> ~~&ensp;&ensp;git=http://github.com/sonata-project/exporter.git~~  
> ~~&ensp;&ensp;target=/exporter~~  

> ~~[SonataDoctrineORMAdminBundle]~~  
> ~~&ensp;&ensp;git=http://github.com/sonata-project/SonataDoctrineORMAdminBundle.git~~  
> ~~&ensp;&ensp;target=/bundles/Sonata/DoctrineORMAdminBundle~~  
> ~~&ensp;&ensp;version=origin/2.0~~  

> ~~[SonataAdminBundle]~~  
> ~~&ensp;&ensp;git=git://github.com/sonata-project/SonataAdminBundle.git~~  
> ~~&ensp;&ensp;target=/bundles/Sonata/AdminBundle~~  
> ~~&ensp;&ensp;version=origin/2.0~~  

~~2、然后运行以下脚本下载扩展：~~

> ~~php bin/vendors install --reinstall~~

~~3、在autoload.php文件中添加自动加载~~

> ~~// app/autoload.php~~  

> ~~$loader->registerNamespaces(array(~~  
> &ensp;&ensp;~~// ...~~  
> &ensp;&ensp;~~'Sonata' => \__DIR__.'/../vendor/bundles',~~  
> &ensp;&ensp;~~'Exporter' => \__DIR__.'/../vendor/exporter/lib',~~  
> &ensp;&ensp;~~'Knp\Bundle' => \__DIR__.'/../vendor/bundles',~~  
> &ensp;&ensp;~~'Knp\Menu' => \__DIR__.'/../vendor/KnpMenu/src',~~  
> &ensp;&ensp;~~// ...~~  
> ~~));~~  


~~4、将Bundle注册人AppKernel.php文件~~

> ~~// app/AppKernel.php~~  

> ~~public function registerBundles()~~  
> ~~{~~  
> &ensp;&ensp;~~return array(~~  
> &ensp;&ensp;&ensp;&ensp;~~// ...~~  
> &ensp;&ensp;&ensp;&ensp;~~new Sonata\AdminBundle\SonataAdminBundle(),~~  
> &ensp;&ensp;&ensp;&ensp;~~new Sonata\BlockBundle\SonataBlockBundle(),~~  
> &ensp;&ensp;&ensp;&ensp;~~new Sonata\CacheBundle\SonataCacheBundle(),~~  
> &ensp;&ensp;&ensp;&ensp;~~new Sonata\jQueryBundle\SonatajQueryBundle(),~~  
> &ensp;&ensp;&ensp;&ensp;~~new Sonata\DoctrineORMAdminBundle\SonataDoctrineORMAdminBundle(),~~  
> &ensp;&ensp;&ensp;&ensp;~~new Knp\Bundle\MenuBundle\KnpMenuBundle(),~~  
> &ensp;&ensp;&ensp;&ensp;~~// ...~~  
> &ensp;&ensp;~~);~~  
> ~~}~~  

输入如下命令进行安装：
> **php composer.phar require sonata-project/admin-bundle**  
> \>Please provide a version constraint for the sonata-project/admin-bundle requirement: *

其中我们为了使用最新版所以使用了*，你们也可以根据需要来调整。

除此之外我们还需要安装SonataDoctrineORMADminBundle：

> **php composer.phar require sonata-project/doctrine-orm-admin-bundle**

现在我们把新下载的Bundle注册到AppKernel.php文件中：

```php
// app/AppKernel.php
// ...
public function registerBundles()
{
    $bundles = array(
        // ...
        new Sonata\AdminBundle\SonataAdminBundle(),
        new Sonata\BlockBundle\SonataBlockBundle(),
        new Sonata\jQueryBundle\SonatajQueryBundle(),
        new Sonata\DoctrineORMAdminBundle\SonataDoctrineORMAdminBundle(),
        new Knp\Bundle\MenuBundle\KnpMenuBundle(),
    );
}
 
// ...
```

现在我们需要修改你的app/config/config.yml文件，在其最后加上如下代码：

```yaml
#...
sonata_admin:
    title: Jobeet Admin
 
sonata_block:
    default_contexts: [cms]
    blocks:
        sonata.admin.block.admin_list:
            contexts:   [admin]
 
        sonata.block.service.text:
        sonata.block.service.action:
        sonata.block.service.rss:
```

同时查找translator项，将其注释取消：

```yaml
framework:
    translator:      { fallback: %locale% }
```

为了能使你的程序正常工作我们还需要在`app/config/routing.yml`下为这个bundle添加路由：

```yaml
admin:
    resource: '@SonataAdminBundle/Resources/config/routing/sonata_admin.xml'
    prefix: /admin
 
_sonata_admin:
    resource: .
    type: sonata_admin
    prefix: /admin
 
# ...
```

完成之后记得更新资源并清除缓存：

> **php app/console assets:install web --symlink**
> **php app/console cache:clear --env=dev**
> **php app/console cache:clear --env=prod**

如今你就可以访问如下网址来实际感受一下了：

*http://jobeet.local/app_dev.php/admin/dashboard*

## CRUD控制器

CRUD 控制器包含了最基本的 CRUD 操作。它可以将一个Admin类的实例正确的映射到一个同名的控制器中。其中的任何操作都可以被重写以此来满足我们的需求。控制器使用Admin类来构造不同的操作。在控制器中，我们可以通过configuration属性来访问Admin对象。

现在让我们为每个entity创建一个控制器。首先是Category：

```php
// src/Ens/JobeetBundle/Controller/CategoryAdminController.php
namespace Ens\JobeetBundle\Controller;
 
use Sonata\AdminBundle\Controller\CRUDController as Controller;
 
class CategoryAdminController extends Controller
{
 
}
```

然后是Job：

```php
// src/Ens/JobeetBundle/Controller/JobAdminController.php
namespace Ens\JobeetBundle\Controller;
 
use Sonata\AdminBundle\Controller\CRUDController as Controller;
 
class JobAdminController extends Controller
{
 
}
```

## 创建Admin类

Admin类用来表示模型的映射与管理分区(表单、 列表、 显示)。其中创建admin类最简单的方式就是去继承`Sonata\AdminBundle\Admin\Admin`类。为此我们会在我们的bundle下创建一个admin文件夹，并在此进行操作。

category：

```php
// src/Ens/JobeetBundle/Admin/CategoryAdmin.php
 
namespace Ens\JobeetBundle\Admin;
 
use Sonata\AdminBundle\Admin\Admin;
use Sonata\AdminBundle\Datagrid\ListMapper;
use Sonata\AdminBundle\Datagrid\DatagridMapper;
use Sonata\AdminBundle\Validator\ErrorElement;
use Sonata\AdminBundle\Form\FormMapper;
 
class CategoryAdmin extends Admin
{
}
```

Job：

```php
// src/Ens/JobeetBundle/Admin/JobAdmin.php
 
namespace Ens\JobeetBundle\Admin;
 
use Sonata\AdminBundle\Admin\Admin;
use Sonata\AdminBundle\Datagrid\ListMapper;
use Sonata\AdminBundle\Datagrid\DatagridMapper;
use Sonata\AdminBundle\Validator\ErrorElement;
use Sonata\AdminBundle\Form\FormMapper;
 
class JobAdmin extends Admin
{
}
```

现在我们需要为每个admin类在services.yml中进行配置：

> ~~#src/Ens/JobeetBundle/Resources/config/services.yml~~  
> ~~services:~~  
> &ensp;&ensp;~~ens.jobeet.admin.category:~~  
> &ensp;&ensp;&ensp;&ensp;~~class: Ens\JobeetBundle\Admin\CategoryAdmin~~  
> &ensp;&ensp;&ensp;&ensp;~~tags:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;~~- { name: sonata.admin, manager_type: orm, group: jobeet, label: Categories }~~  
> &ensp;&ensp;&ensp;&ensp;~~arguments: [null, Ens\JobeetBundle\Entity\Category, EnsJobeetBundle:CategoryAdmin]~~  
> &ensp;&ensp;~~ens.jobeet.admin.job:~~  
> &ensp;&ensp;&ensp;&ensp;~~class: Ens\JobeetBundle\Admin\JobAdmin~~  
> &ensp;&ensp;&ensp;&ensp;~~tags:~~  
> &ensp;&ensp;&ensp;&ensp;&ensp;~~- { name: sonata.admin, manager_type: orm, group: jobeet, label: Jobs }~~  
> &ensp;&ensp;&ensp;&ensp;~~arguments: [null, Ens\JobeetBundle\Entity\Job, EnsJobeetBundle:JobAdmin]~~  

```yaml
#src/Ens/JobeetBundle/Resources/config/services.yml

services:
    ens.jobeet.admin.category:
        class: EnsJobeetBundleAdminCategoryAdmin
        tags:
            - { name: sonata.admin, manager_type: orm, group: jobeet, label: Categories }
        arguments:
            - ~
            - EnsJobeetBundleEntityCategory
            - 'EnsJobeetBundle:CategoryAdmin'
 
    ens.jobeet.admin.job:
        class: EnsJobeetBundleAdminJobAdmin
        tags:
            - { name: sonata.admin, manager_type: orm, group: jobeet, label: Jobs }
        arguments:
            - ~
            - EnsJobeetBundleEntityJob
            - 'EnsJobeetBundle:JobAdmin'

```

现在我们能在Jobeet的管理面板上看到效果了。 你能找到Job和Category模块以及他们的相关操作链接---新增与列表。

![仪表盘](http://www.ens.ro/wp-content/uploads/2012/07/jobeet_admin_011-1024x214.png)

## 配置Admin类

如果此时我们去点击任何一个链接，你会发现并没有什么卯月。。。这是因为我们还没有配置列表和表单的字段。让我从一个最基础的配置开始。依旧先是category：

```php
// src Ens/JobeetBundle/Admin/CategoryAdmin.php
namespace Ens\JobeetBundle\Admin;
 
use Sonata\AdminBundle\Admin\Admin;
use Sonata\AdminBundle\Datagrid\ListMapper;
use Sonata\AdminBundle\Datagrid\DatagridMapper;
use Sonata\AdminBundle\Validator\ErrorElement;
use Sonata\AdminBundle\Form\FormMapper;
 
class CategoryAdmin extends Admin
{
    // setup the default sort column and order
    protected $datagridValues = array(
        '_sort_order' => 'ASC',
        '_sort_by' => 'name'
    );
 
    protected function configureFormFields(FormMapper $formMapper)
    {
        $formMapper
            ->add('name')
            ->add('slug')
        ;
    }
 
    protected function configureDatagridFilters(DatagridMapper $datagridMapper)
    {
        $datagridMapper
            ->add('name')
        ;
    }
 
    protected function configureListFields(ListMapper $listMapper)
    {
        $listMapper
            ->addIdentifier('name')
            ->add('slug')
        ;
    }
}
```

然后是job

```php
// src Ens/JobeetBundle/Admin/JobAdmin.php
namespace Ens\JobeetBundle\Admin;
 
use Sonata\AdminBundle\Admin\Admin;
use Sonata\AdminBundle\Datagrid\ListMapper;
use Sonata\AdminBundle\Datagrid\DatagridMapper;
use Sonata\AdminBundle\Validator\ErrorElement;
use Sonata\AdminBundle\Form\FormMapper;
use Sonata\AdminBundle\Show\ShowMapper;
use Ens\JobeetBundle\Entity\Job;
 
class JobAdmin extends Admin
{
    // setup the defaut sort column and order
    protected $datagridValues = array(
        '_sort_order' => 'DESC',
        '_sort_by' => 'created_at'
    );
 
    protected function configureFormFields(FormMapper $formMapper)
    {
        $formMapper
            ->add('category')
            ->add('type', 'choice', array('choices' => Job::getTypes(), 'expanded' => true))
            ->add('company')
            ->add('file', 'file', array('label' => 'Company logo', 'required' => false))
            ->add('url')
            ->add('position')
            ->add('location')
            ->add('description')
            ->add('how_to_apply')
            ->add('is_public')
            ->add('email')
            ->add('is_activated')
        ;
    }
 
    protected function configureDatagridFilters(DatagridMapper $datagridMapper)
    {
        $datagridMapper
            ->add('category')
            ->add('company')
            ->add('position')
            ->add('description')
            ->add('is_activated')
            ->add('is_public')
            ->add('email')
            ->add('expires_at')
        ;
    }
 
    protected function configureListFields(ListMapper $listMapper)
    {
        $listMapper
            ->addIdentifier('company')
            ->add('position')
            ->add('location')
            ->add('url')
            ->add('is_activated')
            ->add('email')
            ->add('category')
            ->add('expires_at')
            ->add('_action', 'actions', array(
                'actions' => array(
                    'view' => array(),
                    'edit' => array(),
                    'delete' => array(),
                )
            ))
        ;
    }
 
    protected function configureShowField(ShowMapper $showMapper)
    {
        $showMapper
            ->add('category')
            ->add('type')
            ->add('company')
            ->add('webPath', 'string', array('template' => 'EnsJobeetBundle:JobAdmin:list_image.html.twig'))
            ->add('url')
            ->add('position')
            ->add('location')
            ->add('description')
            ->add('how_to_apply')
            ->add('is_public')
            ->add('is_activated')
            ->add('token')
            ->add('email')
            ->add('expires_at')
        ;
    }
}
```

在show action中我们需要使用自定义模板使其显示公司的logo

```html
<!-- src/Ens/JobeetBundle/Resources/views/JobAdmin/list_image.html.twig -->
 
<tr>
    <th>Logo</th>
    <td><img src="{{ asset(object.webPath) }}" /></td>
</tr>
```

现在我们创建了一个基本的管理模块用来对职位和分类进行CRUD操作，当你使用时将会发现以下特点：

* 列表可以分页
* 列表可以排序
* 列表可以筛选
* 可以创建、 编辑和删除对象
* 可以在批处理删除选中的对象
* 启用了表单验证
* 向用户提供即时的反馈

## 批量操作

批量操作是在一组选定的模型 （所有的或只是其中的一部分） 上触发的操作。在列表视图中，你可以轻松地添加一些自定义批量操作处理。默认情况下删除操作允许你一次删除多个条目。

若要添加新的批处理操作，我们必须重写(override)从 Admin 类 继承的getBatchActions方法。我们在这里将定义一个新的extend 操作︰

```php
// src/Ens/JobeetBundle/Admin/JobAdmin.php
// ...
 
public function getBatchActions()
{
    // retrieve the default (currently only the delete action) actions
    $actions = parent::getBatchActions();
 
    // check user permissions
    if($this->hasRoute('edit') && $this->isGranted('EDIT') && $this->hasRoute('delete') && $this->isGranted('DELETE')) {
        $actions['extend'] = array(
            'label'            => 'Extend',
            'ask_confirmation' => true // If true, a confirmation will be asked before performing the action
        );
 
    }
 
    return $actions;
}
```

我们会在JobAdminController中的batchActionExtend方法中实现这次所需要的核心逻辑。这个方法中会传递一个查询参数用来选择适当的Model来进行操作。但如果由于某种原因，不使用默认的选择方法来执行批处理操作（例如你定义另一种方法，在模板级别，以较低的粒度来选择Model），那么你可以传递一个null值。

> *译者注：这段原文就特别的拗口————反正我看了几遍也不知道在说啥，求翻译，求帮助———— 你们就看过算过，直接看代码比较容易理解<_<*

```php
// src/Ens/JobeetBundle/Controller/JobAdminController.php
namespace Ens\JobeetBundle\Controller;
 
use Sonata\AdminBundle\Controller\CRUDController as Controller;
use Sonata\DoctrineORMAdminBundle\Datagrid\ProxyQuery as ProxyQueryInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
 
class JobAdminController extends Controller
{
    public function batchActionExtend(ProxyQueryInterface $selectedModelQuery)
    {
        if ($this->admin->isGranted('EDIT') === false || $this->admin->isGranted('DELETE') === false)
        {
            throw new AccessDeniedException();
        }
 
        $request = $this->get('request');
        $modelManager = $this->admin->getModelManager();
 
        $selectedModels = $selectedModelQuery->execute();
 
        try {
            foreach ($selectedModels as $selectedModel) {
                $selectedModel->extend();
                $modelManager->update($selectedModel);
            }
        } catch (\Exception $e) {
            $this->get('session')->setFlash('sonata_flash_error', $e->getMessage());
 
            return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
        }
 
        $this->get('session')->setFlash('sonata_flash_success',  sprintf('The selected jobs validity has been extended until %s.', date('m/d/Y', time() + 86400 * 30)));
 
        return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
    }
}
```

> *译者注：其实主要用到的就是20~26行的代码,应该能看懂。*

再让我们添加一个新的批处理操作，用来删除所有超过60天但尚未激活的职位信息。对这个action，我们不需要从列表中选择任何职位信息，因为action的逻辑会把搜索匹配的记录自动删除。

```php
// src/Ens/JobeetBundle/Admin/JobAdmin.php
// ...
 
public function getBatchActions()
{
    // retrieve the default (currently only the delete action) actions
    $actions = parent::getBatchActions();
 
    // check user permissions
    if($this->hasRoute('edit') && $this->isGranted('EDIT') && $this->hasRoute('delete') && $this->isGranted('DELETE')){
        $actions['extend'] = array(
            'label'            => 'Extend',
            'ask_confirmation' => true // If true, a confirmation will be asked before performing the action
        );
 
        $actions['deleteNeverActivated'] = array(
            'label'            => 'Delete never activated jobs',
            'ask_confirmation' => true // If true, a confirmation will be asked before performing the action
        );
    }
 
    return $actions;
}
```

另外我们还要创建 batchActionDeleteNeverActivated 和batchActionDeleteNeverActivatedIsRelevant两个方法，后者将会在任何确认**之前**执行，以确保用户确实要进行该操作（在本例中它将始终返回 true，因为选择要删除的职位信息的逻辑在 JobRepository::cleanup() 方法中。）

> *译者注：此处"之前"这个词本身对用户来说应该是"之后"，即点击了按钮之后执行，但对于程序来说是在其在"确认"这个操作之前进行的，因为IsRelevant操作会将此值传递给sonata进行判断，以确保"确认"这个操作实际上是有值的,并根据此值来进行继续还是输出错误信息。故这里采用原文意思。*

```php
// src/Ens/JobeetBundle/Controller/JobAdminController.php
// ...
 
public function batchActionDeleteNeverActivatedIsRelevant()
{
    return true;
}
 
public function batchActionDeleteNeverActivated()
{
    if ($this->admin->isGranted('EDIT') === false || $this->admin->isGranted('DELETE') === false) {
        throw new AccessDeniedException();
    }
 
    $em = $this->getDoctrine()->getManager();
    $nb = $em->getRepository('EnsJobeetBundle:Job')->cleanup(60);
 
    if ($nb) {
        $this->get('session')->setFlash('sonata_flash_success',  sprintf('%d never activated jobs have been deleted successfully.', $nb));
    } else {
        $this->get('session')->setFlash('sonata_flash_info',  'No job to delete.');
    }
 
    return new RedirectResponse($this->admin->generateUrl('list',$this->admin->getFilterParameters()));
}
```

> *译者注：关于IsRelevant详细方法可以参照[BATCH ACTIONS](https://sonata-project.org/bundles/admin/2-0/doc/reference/batch_actions.html)*

这就是今天的所有内容了，明天，我们将看到如何安全的管理使用用户名和密码。到了该聊聊Symfony安全方面的时候了。