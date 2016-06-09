# Symfony2 Jobeet Day 10: 表单

对于任何网站来说它们都会有表单，它们有可能是简单的联系人表单，也有可能是复杂的多表单域表单，林林总总，花样繁多。因此对于web开发人员来说创建表单也是一项既繁琐又复杂的工作：首先你需要写一个 HTML 表单，然后为每个字段创建验证规则，再将表单值处理之后保存进数据库，如果有错误还需要向前台用户显示错误信息，同时把其他字段重新填回表单中，诸如此类……

在第三天的教程中我们使用 `doctrine:generate:crud` 命令来为我们的Job entity 生成简单的CRUD控制器。它也为我们创建了一个简单的Job表单，我们可以在/src/Ens/JobeetBundle/Form/JobType.php下面找到它。

## 自定义一个Job表单

Job表单是用来学习自定义方式的一个很好的例子。现在跟着我来，一步步的定制它。

首先我们要修改"Post a Job"链接的布局使得你能在浏览器上直接进行检验：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->
<a href="{{ path('ens_job_new') }}">Post a Job</a>
```

然后修改 JobController 下的 createAction 方法中的 ens_job_show 路由参数。使它能匹配我们第五天所建立的新路由。

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function createAction()
{
  $entity  = new Job();
  $request = $this->getRequest();
  $form    = $this->createForm(new JobType(), $entity);
  $form->bindRequest($request);
 
  if ($form->isValid()) {
    $em = $this->getDoctrine()->getManager();
 
    $em->persist($entity);
    $em->flush();
 
    return $this->redirect($this->generateUrl('ens_job_show', array(
      'company' => $entity->getCompanySlug(),
      'location' => $entity->getLocationSlug(),
      'id' => $entity->getId(),
      'position' => $entity->getPositionSlug()
    )));
  }
 
  return $this->render('EnsJobeetBundle:Job:new.html.twig', array(
    'entity' => $entity,
    'form'   => $form->createView()
  ));
}
```

默认情况下Doctrine生成器产生的表单将会显示相关表的所有字段。但对于Job表单，其中一些字段不应该由最终用户编辑。像下面展示的那样修改你的表单︰

```php
// src/Ens/JobeetBundle/Form/JobType.php
 
namespace Ens\JobeetBundle\Form;
 
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilder;
 
class JobType extends AbstractType
{
    public function buildForm(FormBuilder $builder, array $options)
    {
        $builder->add('category');
        $builder->add('type');
        $builder->add('company');
        $builder->add('logo');
        $builder->add('url');
        $builder->add('position');
        $builder->add('location');
        $builder->add('description');
        $builder->add('how_to_apply');
        $builder->add('token');
        $builder->add('is_public');
        $builder->add('email');
    }
 
    public function getName()
    {
        return 'ens_jobeetbundle_jobtype';
    }
}
```

然而有时候我们所需要的表单配置将会比从数据库中映射的更为精确。比如，email列在数据库中我们设计的是varchar型的，但我们需要验证这个列它是否符合email格式。在 Symfony2 中验证是附加于基础对象上的(例如：Job)。换句话说，问题不是"form"是否被验证，而是这个表单数据被提交之后这个Job对象是否被验证。要做到这一点，我们要在相应的Bundle下的Resources/config目录下新建一个validation.yml：

```yaml
#src/Ibw/JobeetBundle/Resources/config/validation.yml
Ibw\JobeetBundle\Entity\Job:
    properties:
        email:
            - NotBlank: ~
            - Email: ~
```

尽管type列我们在数据库中定义的varchar，但我们希望其值被限制在列表中选择 ︰ 全职、 兼职或自由职业者。

```php
// src/Ens/JobeetBundle/Form/JobType.php
// ...
 
use Ens\JobeetBundle\Entity\Job;
 
class JobType extends AbstractType
{
    public function buildForm(FormBuilder $builder, array $options)
    {
        // ...
 
        $builder->add('type', 'choice', array('choices' => Job::getTypes(), 'expanded' => true));
 
        // ...
    }
 
    // ...
 
}
```
为了能让其正常工作，我们还需要在Job entity中添加如下方法：

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
 
public static function getTypes()
{
  return array('full-time' => 'Full time', 'part-time' => 'Part time', 'freelance' => 'Freelance');
}
 
public static function getTypeValues()
{
  return array_keys(self::getTypes());
}
 
// ...
```

getTypes方法使表单获取可能使用的类型，getTypeValues方法将会验证传来的type字段是否正确：

```php
# src/Ens/JobeetBundle/Resources/config/validation.yml
 
Ens\JobeetBundle\Entity\Job:
    properties:
        type:
            - NotBlank: ~
            - Choice: { callback: getTypeValues }
        email:
            - NotBlank: ~
            - Email: ~
```

对于每个字段，Symfony 都会自动创建一个与其字段名相同的label标签，我们可以通过参数来修改它：
```php
$builder->add('logo', null, array('label' => 'Company logo'));
$builder->add('how_to_apply', null, array('label' => 'How to apply?'));
$builder->add('is_public', null, array('label' => 'Public?'));
```

我们还应当为其他字段添加验证规则 ︰

```yaml
#src/Ens/JobeetBundle/Resources/config/validation.yml
 
Ens\JobeetBundle\Entity\Job:
    properties:
        category:
            - NotBlank: ~
        type:
            - NotBlank: ~
            - Choice: {callback: getTypeValues}
        company:
            - NotBlank: ~
        position:
            - NotBlank: ~
        location:
            - NotBlank: ~
        description:
            - NotBlank: ~
        how_to_apply:
            - NotBlank: ~
        token:
            - NotBlank: ~
        email:
            - NotBlank: ~
            - Email: ~
```

## 在Symfony2中处理文件上传

为了处理表单上传文件，我们需要使用一个"虚拟"的文件字段。为此我们将向Job entity中添加新的文件属性：

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
 
public $file;
```

现在我们需要将logo替换为文件上传组件：

```php
// src/Ens/JobeetBundle/Form/JobType.php
// ...
 
$builder->add('file', 'file', array('label' => 'Company logo', 'required' => false));
 
// ...
```

为了确保上传的是图片类型，我们需要添加Image规则验证：

```yaml
#src/Ens/JobeetBundle/Resources/config/validation.yml
Ens\JobeetBundle\Entity\Job:
    properties:
        #...
        file:
            - Image: ~
```

当表单提交时，file字段将自动生成一个 [UploadedFile](http://api.symfony.com/2.3/Symfony/Component/HttpFoundation/File/UploadedFile.html) 的实例。我们可以通过它将文件移动到某一位置。在此之后，我们只需将logo属性设置为上传的文件名即可。

```php
// src/Ens/JobeedBundle/Controller/JobController.php
// ...
 
public function createAction()
{
  // ...
 
  if ($form->isValid()) {
    $em = $this->getDoctrine()->getManager();
 
    $entity->file->move(__DIR__.'/../../../../web/uploads/jobs', $entity->file->getClientOriginalName());
    $entity->setLogo($entity->file->getClientOriginalName());
 
    $em->persist($entity);
    $em->flush();
 
    return $this->redirect($this->generateUrl('ens_job_show', array(
      'company' => $entity->getCompanySlug(),
      'location' => $entity->getLocationSlug(),
      'id' => $entity->getId(),
      'position' => $entity->getPositionSlug()
    )));
  }
  // ...
}
```

> *你需要创建一个logo目录(在*web/uploads/jobs/*)并确认在服务器上有写的权限。*

尽管这个已经能实现要求了，但更好的做法是使用 Doctrine Job entity 来处理文件上传。

首先在Job entity中添加如下代码：

```php
protected function getUploadDir()
{
    return 'uploads/jobs';
}
 
protected function getUploadRootDir()
{
    return __DIR__.'/../../../../web/'.$this->getUploadDir();
}
 
public function getWebPath()
{
    return null === $this->logo ? null : $this->getUploadDir().'/'.$this->logo;
}
 
public function getAbsolutePath()
{
    return null === $this->logo ? null : $this->getUploadRootDir().'/'.$this->logo;
}
```

logo属性会将文件的相对路径保存到数据库中。使用getAbsolutePath()方法可以轻松的获得文件的绝对路径，同时你也可以使用getWebPath()方法来获得一个相对于web目录的相对路径---当你需要在模板中使用上传图片的时候。

我们将要实现文件在数据库中的操作和移动，并确保其具有原子性：即当保存人数据库发生错误或者文件不能在相应目录下保存，那么应当相当于什么都没有发生。要做到这一点，我们需要将移动文件的权利转交给Doctrine，当它保存到数据库时自动完成。我们可以通过在Job entity中的lifecycle callback里添加hook方法来完成这个工作。就像我们第三天所做的那样，在Job.orm.yml中添加preUpload, upload 和 removeUpload 三个回调函数：

```php
#src/Ens/JobeetBundle/Resources/config/doctrine/Job.orm.yml
Ens\JobeetBundle\Entity\Job:
#...
 
  lifecycleCallbacks:
    prePersist: [ preUpload, setCreatedAtValue, setExpiresAtValue ]
    preUpdate: [ preUpload, setUpdatedAtValue ]
    postPersist: [ upload ]
    postUpdate: [ upload ]
    postRemove: [ removeUpload ]
```

然后运行generate:entities可以将新的方法添加到Job entity中：

> **php app/console doctrine:generate:entities EnsJobeetBundle**

打开Job entity并编辑新添加的方法：

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
 
/**
* @ORM\prePersist
*/
public function preUpload()
{
  if (null !== $this->file) {
    // do whatever you want to generate a unique name
    $this->logo = uniqid().'.'.$this->file->guessExtension();
  }
}
 
/**
* @ORM\postPersist
*/
public function upload()
{
  if (null === $this->file) {
    return;
  }
 
  // if there is an error when moving the file, an exception will
  // be automatically thrown by move(). This will properly prevent
  // the entity from being persisted to the database on error
  $this->file->move($this->getUploadRootDir(), $this->logo);
 
  unset($this->file);
}
 
/**
* @ORM\postRemove
*/
public function removeUpload()
{
  if ($file = $this->getAbsolutePath()) {
    unlink($file);
  }
}
 
// ...
```

现在这个类有了我们操作所需的一切：在`persisting`之前创建一个唯一的文件名，在`persisting`之后移动上传文件，当entity被删除后自动移除相应的文件。如今文件的操作是以原子的方式被entity所掌控了。我们需要删除早先在控制器中所写的上传代码，并修改如下：

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function createAction()
{
  $entity  = new Job();
  $request = $this->getRequest();
  $form    = $this->createForm(new JobType(), $entity);
  $form->bindRequest($request);
 
  if ($form->isValid()) {
    $em = $this->getDoctrine()->getManager();
 
    $em->persist($entity);
    $em->flush();
 
    return $this->redirect($this->generateUrl('ens_job_show', array(
      'company' => $entity->getCompanySlug(),
      'location' => $entity->getLocationSlug(),
      'id' => $entity->getId(),
      'position' => $entity->getPositionSlug()
    )));
  }
 
  return $this->render('EnsJobeetBundle:Job:new.html.twig', array(
    'entity' => $entity,
    'form'   => $form->createView()
  ));
}
 
// ...
```

## form 模板

现在我们的form类已经定制完成，需要将其显示出来。打开并编辑new.html.twig：

```html
<!-- /src/Ens/JobeetBundle/Resources/views/Job/new.html.twig -->
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% form_theme form _self %}
 
{% block field_errors %}
{% spaceless %}
  {% if errors|length > 0 %}
    <ul class="error_list">
      {% for error in errors %}
        <li>{{ error.messageTemplate|trans(error.messageParameters, 'validators') }}</li>
      {% endfor %}
    </ul>
  {% endif %}
{% endspaceless %}
{% endblock field_errors %}
 
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/job.css') }}" type="text/css" media="all" />
{% endblock %}
 
{% block content %}
  <h1>Job creation</h1>
  <form action="{{ path('ens_job_create') }}" method="post" {{ form_enctype(form) }}>
    <table id="job_form">
      <tfoot>
        <tr>
          <td colspan="2">
            <input type="submit" value="Preview your job" />
          </td>
        </tr>
      </tfoot>
      <tbody>
        <tr>
          <th>{{ form_label(form.category) }}</th>
          <td>
            {{ form_errors(form.category) }}
            {{ form_widget(form.category) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.type) }}</th>
          <td>
            {{ form_errors(form.type) }}
            {{ form_widget(form.type) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.company) }}</th>
          <td>
            {{ form_errors(form.company) }}
            {{ form_widget(form.company) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.file) }}</th>
          <td>
            {{ form_errors(form.file) }}
            {{ form_widget(form.file) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.url) }}</th>
          <td>
            {{ form_errors(form.url) }}
            {{ form_widget(form.url) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.position) }}</th>
          <td>
            {{ form_errors(form.position) }}
            {{ form_widget(form.position) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.location) }}</th>
          <td>
            {{ form_errors(form.location) }}
            {{ form_widget(form.location) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.description) }}</th>
          <td>
            {{ form_errors(form.description) }}
            {{ form_widget(form.description) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.how_to_apply) }}</th>
          <td>
            {{ form_errors(form.how_to_apply) }}
            {{ form_widget(form.how_to_apply) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.token) }}</th>
          <td>
            {{ form_errors(form.token) }}
            {{ form_widget(form.token) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.is_public) }}</th>
          <td>
            {{ form_errors(form.is_public) }}
            {{ form_widget(form.is_public) }}
            <br /> Whether the job can also be published on affiliate websites or not.
          </td>
        </tr>
        <tr>
          <th>{{ form_label(form.email) }}</th>
          <td>
            {{ form_errors(form.email) }}
            {{ form_widget(form.email) }}
          </td>
        </tr>
      </tbody>
    </table>
 
    {{ form_rest(form) }}
  </form>
{% endblock %}
```

我们可以通过如下方式来显示表单，但如果想要更多定制内容的话只能手工来完成每个表单的字段渲染：

```html
{{ form_widget(form) }}
```

直接使用form_widget(form)的话，他将会呈现每个字段的标签和错误信息。这么做虽然很方便，但却不够灵活。通常，我们会想要单独呈现每个表单域，进而可以控制该表单的外观样式。

我们也可以使用[form theming](http://symfony.com/doc/2.3/book/forms.html#form-theming)来[定制如何展示错误信息](http://symfony.com/doc/2.3/cookbook/form/form_customization.html#customizing-error-output)。你可以在Symfony2的官方文档中阅读更多关于这个的信息。

接下来我们将 edit.html.twig进行同样的修改：

```html
<!-- /src/Ens/JobeetBundle/Resources/views/Job/edit.html.twig -->
{% extends 'EnsJobeetBundle::layout.html.twig' %}
 
{% form_theme edit_form _self %}
 
{% block field_errors %}
{% spaceless %}
  {% if errors|length > 0 %}
    <ul>
      {% for error in errors %}
        <li>{{ error.messageTemplate|trans(error.messageParameters, 'validators') }}</li>
      {% endfor %}
    </ul>
  {% endif %}
{% endspaceless %}
{% endblock field_errors %}
 
{% block stylesheets %}
  {{ parent() }}
  <link rel="stylesheet" href="{{ asset('bundles/ensjobeet/css/job.css') }}" type="text/css" media="all" />
{% endblock %}
 
{% block content %}
  <h1>Job edit</h1>
  <form action="{{ path('ens_job_update', { 'id': entity.id }) }}" method="post" {{ form_enctype(edit_form) }}>
    <table id="job_form">
      <tfoot>
        <tr>
          <td colspan="2">
            <input type="submit" value="Preview your job" />
          </td>
        </tr>
      </tfoot>
      <tbody>
        <tr>
          <th>{{ form_label(edit_form.category) }}</th>
          <td>
            {{ form_errors(edit_form.category) }}
            {{ form_widget(edit_form.category) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.type) }}</th>
          <td>
            {{ form_errors(edit_form.type) }}
            {{ form_widget(edit_form.type) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.company) }}</th>
          <td>
            {{ form_errors(edit_form.company) }}
            {{ form_widget(edit_form.company) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.file) }}</th>
          <td>
            {{ form_errors(edit_form.file) }}
            {{ form_widget(edit_form.file) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.url) }}</th>
          <td>
            {{ form_errors(edit_form.url) }}
            {{ form_widget(edit_form.url) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.position) }}</th>
          <td>
            {{ form_errors(edit_form.position) }}
            {{ form_widget(edit_form.position) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.location) }}</th>
          <td>
            {{ form_errors(edit_form.location) }}
            {{ form_widget(edit_form.location) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.description) }}</th>
          <td>
            {{ form_errors(edit_form.description) }}
            {{ form_widget(edit_form.description) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.how_to_apply) }}</th>
          <td>
            {{ form_errors(edit_form.how_to_apply) }}
            {{ form_widget(edit_form.how_to_apply) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.token) }}</th>
          <td>
            {{ form_errors(edit_form.token) }}
            {{ form_widget(edit_form.token) }}
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.is_public) }}</th>
          <td>
            {{ form_errors(edit_form.is_public) }}
            {{ form_widget(edit_form.is_public) }}
            <br /> Whether the job can also be published on affiliate websites or not.
          </td>
        </tr>
        <tr>
          <th>{{ form_label(edit_form.email) }}</th>
          <td>
            {{ form_errors(edit_form.email) }}
            {{ form_widget(edit_form.email) }}
          </td>
        </tr>
      </tbody>
    </table>
 
    {{ form_rest(edit_form) }}
  </form>
{% endblock %}
```

## 表单Action

现在我们已经有了form表单类和相应的输出模板，是时候使其正常工作了。对于这个 Job 表单我们需要在 JobController 中添加四个方法。

* newAction: 展示一个空白表单用来展示新项目。
* createAction: 处理表单(验证信息, 表单回填) 并创建一个新的职位。
* editAction: 展示并编辑一个已存在的职位。
* updateAction: 处理表单(验证信息, 表单回填) 并更新一个已存在的职位。

当你在浏览器中访问/job/new页面时，createForm方法会调用一个新的Job对象的表单实例，并将其传递给模板(newAction)

当用户提交表单(createAction)，表单会将用户传过来的值与相关验证进行绑定(bindRequest() 方法)，并自动触发。

一旦form被绑定，就能使用isValid()方法检查其有效性 ：如果表单验证通过(返回true)，这条职位信息就会被保存到数据库中($em->persist($entity))，并将用户重定向到职位预览页面;否则，new.html.twig 模板将再次显示用户提交的信息并提示关联的错误消息。

修改现有的职位也是类似操作。新建和编辑操作的唯一区别就是---编辑需要被修改的Job对象作为createForm 方法的第二个参数传递进去，并会在表单中替换模板相应的默认值。

您还可以定义创建表单的默认值。为此我们将会传递一个预修改的Job对象给createForm，使type的默认值设置为全职 ︰

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function newAction()
{
  $entity = new Job();
  $entity->setType('full-time');
  $form   = $this->createForm(new JobType(), $entity);
 
  return $this->render('EnsJobeetBundle:Job:new.html.twig', array(
    'entity' => $entity,
    'form'   => $form->createView()
  ));
}
 
// ...
```

## 使用Token保护表单

到目前为止，一切都进行的很顺利，唯独用户需要自己输入Token这项工作。但Token应该是在职位保存之后自动生成的。因为我们不能指望于用户可以提供一个唯一的标识。为此我们需要创建一个 setTokenValue() 方法，在保存职位之前会自动生成一个Token。

```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
 
public function setTokenValue()
{
  if(!$this->getToken())
  {
    $this->token = sha1($this->getEmail().rand(11111, 99999));
  }
}
 
// ...
```

并将这个方法添加到Job entity的lifecycleCallbacks下的prePersist中：

```yaml
#src/End/JobeetBundle/Resources/config/doctrine/Job.orm.yml
#...
 
  lifecycleCallbacks:
    prePersist: [ setTokenValue, preUpload, setCreatedAtValue, setExpiresAtValue ]
    #...
```

也别忘了重新运行一下doctrine:generate:entities命令：

> **php app/console doctrine:generate:entities EnsJobeetBundle**

现在你可以在表单中移除Token了：

```php
// src/Ens/JobeetBundle/Form/JobType.php
// ...
 
public function buildForm(FormBuilder $builder, array $options)
{
    $builder->add('category');
    $builder->add('type', 'choice', array('choices' => Job::getTypes(), 'expanded' => true));
    $builder->add('company');
    $builder->add('file', 'file', array('label' => 'Company logo', 'required' => false));
    $builder->add('url');
    $builder->add('position');
    $builder->add('location');
    $builder->add('description');
    $builder->add('how_to_apply', null, array('label' => 'How to apply?'));
    $builder->add('is_public', null, array('label' => 'Public?'));
    $builder->add('email');
}
 
// ...
```
同时在new.html.twig 和 edit.html.twig中也移除相关代码

```html
<!-- src/Ens/JobeetBundle/Resources/views/Job/new.html/twig -->
<tr>
  <th>{{ form_label(form.token) }}</th>
  <td>
    {{ form_errors(form.token) }}
    {{ form_widget(form.token) }}
  </td>
</tr>
 
<!-- src/Ens/JobeetBundle/Resources/views/Job/edit.html/twig -->
<tr>
  <th>{{ form_label(edit_form.token) }}</th>
  <td>
    {{ form_errors(edit_form.token) }}
    {{ form_widget(edit_form.token) }}
  </td>
</tr>
```

也别漏了validation.yml 文件：
```yaml
token:
    - NotBlank: ~
```

如果你还记得[第二天](03_The_Project.md)所说的话，一份工作只有当用户知道指定的Token时才能被修改。
但现在可以很容易就编辑或删除一个职位---仅通过猜测URL就可以办到。这是因为编辑的URL类似/job/ID/edit,其中ID是job表的主键。

现在让我们来修改路由，使你可以编辑或删除职位仅仅通过天知地知你知而其余人均不知的Token才能办到︰

```yaml
#src/End/JobeetBundle/Resources/config/routing/job.yml
#...
 
ens_job_edit:
    pattern:  /{token}/edit
    defaults: { _controller: "EnsJobeetBundle:Job:edit" }
 
ens_job_update:
    pattern:  /{token}/update
    defaults: { _controller: "EnsJobeetBundle:Job:update" }
    requirements: { _method: post }
 
ens_job_delete:
    pattern:  /{token}/delete
    defaults: { _controller: "EnsJobeetBundle:Job:delete" }
    requirements: { _method: post }
```

修改JobController让Token代替ID：

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
 
  $editForm = $this->createForm(new JobType(), $entity);
  $deleteForm = $this->createDeleteForm($token);
 
  return $this->render('EnsJobeetBundle:Job:edit.html.twig', array(
    'entity'      => $entity,
    'edit_form'   => $editForm->createView(),
    'delete_form' => $deleteForm->createView(),
  ));
}
 
public function updateAction($token)
{
  $em = $this->getDoctrine()->getManager();
 
  $entity = $em->getRepository('EnsJobeetBundle:Job')->findOneByToken($token);
 
  if (!$entity) {
    throw $this->createNotFoundException('Unable to find Job entity.');
  }
 
  $editForm   = $this->createForm(new JobType(), $entity);
  $deleteForm = $this->createDeleteForm($token);
 
  $request = $this->getRequest();
 
  $editForm->bindRequest($request);
 
  if ($editForm->isValid()) {
    $em->persist($entity);
    $em->flush();
 
    return $this->redirect($this->generateUrl('ens_job_edit', array('token' => $token)));
  }
 
  return $this->render('EnsJobeetBundle:Job:edit.html.twig', array(
    'entity'      => $entity,
    'edit_form'   => $editForm->createView(),
    'delete_form' => $deleteForm->createView(),
  ));
}
 
public function deleteAction($token)
{
  $form = $this->createDeleteForm($token);
  $request = $this->getRequest();
 
  $form->bindRequest($request);
 
  if ($form->isValid()) {
    $em = $this->getDoctrine()->getManager();
    $entity = $em->getRepository('EnsJobeetBundle:Job')->findOneByToken($token);
 
    if (!$entity) {
      throw $this->createNotFoundException('Unable to find Job entity.');
    }
 
    $em->remove($entity);
    $em->flush();
  }
 
  return $this->redirect($this->generateUrl('ens_job'));
}
 
private function createDeleteForm($token)
{
  return $this->createFormBuilder(array('token' => $token))
    ->add('token', 'hidden')
    ->getForm()
  ;
}
```

打开show.html.twig模板，找到并修改ens_job_edit的路由：

```html
<a href="{{ path('ens_job_edit', { 'token': entity.token }) }}">
```

edit.html.twig的ens_job_update也别忘了：

```html
<form action="{{ path('ens_job_update', { 'token': entity.token }) }}" method="post" {{ form_enctype(edit_form) }}>
```

现在，所有与职位相关的路由---除了job_show_user都使用了Token。譬如要编辑一个职位，现在路由会是这个样子的：

> *http://jobeet.local/job/TOKEN/edit*

## 职位预览页

预览页与职位详情页显示是相同的。唯一的区别是，预览页使用Token代替了ID ︰

```yaml
#src/End/JobeetBundle/Resources/config/routing/job.yml
#...
 
ens_job_show:
    pattern:  /{company}/{location}/{id}/{position}
    defaults: { _controller: "EnsJobeetBundle:Job:show" }
    requirements:
        id:  \d+
 
ens_job_preview:
    pattern:  /{company}/{location}/{token}/{position}
    defaults: { _controller: "EnsJobeetBundle:Job:preview" }
    requirements:
        token:  \w+
 
#...
```

预览的action也要做相同的替换：

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
 
  return $this->render('EnsJobeetBundle:Job:show.html.twig', array(
    'entity'      => $entity,
    'delete_form' => $deleteForm->createView(),
  ));
}
 
// ...
```

当用户使用token的URL来访问时，我们需要在页面的顶部加一个管理操作栏，show.html.twig模板的最开始处包含这个管理操作栏的模板，并将底部的修改链接删除。

```html
<!-- /src/Ens/JobeetBundle/Resources/views/Job/show.html.twig -->
<!-- ... -->
 
{% block content %}
  {% if app.request.get('token') %}
    {% include 'EnsJobeetBundle:Job:admin.html.twig' with {'job': entity} %}
  {% endif %}
 
  <!-- ... -->
 
{% endblock %}
```

新建一个admin.html.twig模板：

```html
<!-- /src/Ens/JobeetBundle/Resources/views/Job/admin.html.twig -->
 
<div id="job_actions">
  <h3>Admin</h3>
  <ul>
    {% if not job.isActivated %}
      <li><a href="{{ path('ens_job_edit', { 'token': job.token }) }}">Edit</a></li>
      <li><a href="{{ path('ens_job_edit', { 'token': job.token }) }}">Publish</a></li>
    {% endif %}
    <li>
      <form action="{{ path('ens_job_delete', { 'token': job.token }) }}" method="post">
        {{ form_widget(delete_form) }}
        <button type="submit" onclick="if(!confirm('Are you sure?')) { return false; }">Delete</button>
      </form>
    </li>
    {% if job.isActivated %}
      <li {% if job.expiresSoon %} class="expires_soon" {% endif %}>
        {% if job.isExpired %}
          Expired
        {% else %}
          Expires in <strong>{{ job.getDaysBeforeExpires }}</strong> days
        {% endif %}
 
        {% if job.expiresSoon %}
          - <a href="">Extend</a> for another 30 days
        {% endif %}
      </li>
    {% else %}
      <li>
        [Bookmark this <a href="{{ url('ens_job_preview', { 'token': job.token, 'company': job.companyslug, 'location': job.locationslug, 'position': job.positionslug }) }}">URL</a> to manage this job in the future.]
      </li>
    {% endif %}
  </ul>
</div>
```

虽然代码量很多，但大部分都通俗易懂。

为了使模板更具可读性，我们需要为Job entity类中添加一些便捷方法︰

```php
// src/Ibw/JobeetBundle/Entity/Job.php
// ...
 
public function isExpired()
{
    return $this->getDaysBeforeExpires() < 0;
}
 
public function expiresSoon()
{
    return $this->getDaysBeforeExpires() < 5;    
}
 
public function getDaysBeforeExpires()
{
    return ceil(($this->getExpiresAt()->format('U') - time()) / 86400);
}
 
// ...
```

管理操作栏会根据该职位的不同状态展现不同的action操作：

![admin-bar-1](http://www.ens.ro/wp-content/uploads/2012/05/admin-bar-1.png)

![admin-bar-2](http://www.ens.ro/wp-content/uploads/2012/05/admin-bar-2.png)


我们现在将 JobController 中的 create 和 update action重定向到新的职位预览页：

```php
public function createAction()
{
  // ...
 
  if ($form->isValid()) {
    // ...
 
    return $this->redirect($this->generateUrl('ens_job_preview', array(
      'company' => $entity->getCompanySlug(),
      'location' => $entity->getLocationSlug(),
      'token' => $entity->getToken(),
      'position' => $entity->getPositionSlug()
    )));
  }
 
  // ...
}
 
public function updateAction($token)
{
  // ...
 
  if ($editForm->isValid()) {
    // ...
 
    return $this->redirect($this->generateUrl('ens_job_preview', array(
      'company' => $entity->getCompanySlug(),
      'location' => $entity->getLocationSlug(),
      'token' => $entity->getToken(),
      'position' => $entity->getPositionSlug()
    )));
  }
 
  // ...
}
```

## 激活与发布职位

在上一节中，有一个发布职位的链接，现在我们需要为它添加一个路由，使得它能跳转到publish action。

```yaml
#src/Ens/JobeetBundle/Resources/config/routing/job.yml
#...
 
ens_job_publish:
    pattern:  /{token}/publish
    defaults: { _controller: "EnsJobeetBundle:Job:publish" }
    requirements: { _method: post }
```

我们现在来修改"发布"链接(我们将会像删除职位时一样需要使用一个表单，其方式为POST请求)：

```html
<!-- src/Ens/JobeetBundle/Resources/views/job/admin.html.twig -->
<!-- ... -->
 
{% if not job.isActivated %}
  <li><a href="{{ path('ens_job_edit', { 'token': job.token }) }}">Edit</a></li>
  <li>
    <form action="{{ path('ens_job_publish', { 'token': job.token }) }}" method="post">
      {{ form_widget(publish_form) }}
      <button type="submit">Publish</button>
    </form>
  </li>
{% endif %}
 
<!-- ... -->
```

在最后一步我们需要创建一个publish action，并在preview action中发送 publish 表单到模板中。

```php
// src/Ens/JobeetBundle/Controller/JobController.php
// ...
 
public function previewAction($token)
{
  // ...
 
  $deleteForm = $this->createDeleteForm($entity->getId());
  $publishForm = $this->createPublishForm($entity->getToken());
 
  return $this->render('EnsJobeetBundle:Job:show.html.twig', array(
    'entity'      => $entity,
    'delete_form' => $deleteForm->createView(),
    'publish_form' => $publishForm->createView(),
  ));
}
 
public function publishAction($token)
{
  $form = $this->createPublishForm($token);
  $request = $this->getRequest();
 
  $form->bindRequest($request);
 
  if ($form->isValid()) {
    $em = $this->getDoctrine()->getManager();
    $entity = $em->getRepository('EnsJobeetBundle:Job')->findOneByToken($token);
 
    if (!$entity) {
      throw $this->createNotFoundException('Unable to find Job entity.');
    }
 
    $entity->publish();
    $em->persist($entity);
    $em->flush();
 
    $this->get('session')->setFlash('notice', 'Your job is now online for 30 days.');
  }
 
  return $this->redirect($this->generateUrl('ens_job_preview', array(
    'company' => $entity->getCompanySlug(),
    'location' => $entity->getLocationSlug(),
    'token' => $entity->getToken(),
    'position' => $entity->getPositionSlug()
  )));
}
 
private function createPublishForm($token)
{
  return $this->createFormBuilder(array('token' => $token))
    ->add('token', 'hidden')
    ->getForm()
  ;
}
 
// ...
```

 其中publishAction()方法会调用新的publish()方法，其代码如下：

 ```php
// src/Ens/JobeetBundle/Entity/Job.php
// ...
 
public function publish()
{
  $this->setIsActivated(true);
}
 
// ...
 ```

现在你能在你的浏览器中测试新的发布方法啦！

尽管已经很完善了，但我们还有一些细节需要调整。比如非有效工作必须是不可访问的，这也意味着他们不能再Jobeet主页上出现，同时也不能通过URL访问。为此我们需要编辑 JobRepository 来完善这个要求 ︰

```php
// src Ens/JobeetBundle/Repository/JobRepository.php
 
namespace Ens\JobeetBundle\Repository;
use Doctrine\ORM\EntityRepository;
 
class JobRepository extends EntityRepository
{
  public function getActiveJobs($category_id = null, $max = null, $offset = null)
  {
    $qb = $this->createQueryBuilder('j')
    ->where('j.expires_at > :date')
    ->setParameter('date', date('Y-m-d H:i:s', time()))
    ->andWhere('j.is_activated = :activated')
    ->setParameter('activated', 1)
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
 
  public function countActiveJobs($category_id = null)
  {
    $qb = $this->createQueryBuilder('j')
    ->select('count(j.id)')
    ->where('j.expires_at > :date')
    ->setParameter('date', date('Y-m-d H:i:s', time()))
    ->andWhere('j.is_activated = :activated')
    ->setParameter('activated', 1);
 
    if($category_id)
    {
      $qb->andWhere('j.category = :category_id')
        ->setParameter('category_id', $category_id);
    }
 
    $query = $qb->getQuery();
 
    return $query->getSingleScalarResult();
  }
 
  public function getActiveJob($id)
  {
    $query = $this->createQueryBuilder('j')
      ->where('j.id = :id')
      ->setParameter('id', $id)
      ->andWhere('j.expires_at > :date')
      ->setParameter('date', date('Y-m-d H:i:s', time()))
      ->andWhere('j.is_activated = :activated')
      ->setParameter('activated', 1)
      ->setMaxResults(1)
      ->getQuery();
 
    try {
      $job = $query->getSingleResult();
    } catch (\Doctrine\Orm\NoResultException $e) {
      $job = null;
    }
 
    return $job;
  }
}
```

同理，CategoryRepository的getWithJobs()方法也别忘了：

```php
// src/Ens/JobeetBundle/Repository/CategoryRepository.php
 
namespace Ens\JobeetBundle\Repository;
use Doctrine\ORM\EntityRepository;
 
class CategoryRepository extends EntityRepository
{
  public function getWithJobs()
  {
    $query = $this->getManager()->createQuery(
      'SELECT c FROM EnsJobeetBundle:Category c LEFT JOIN c.jobs j WHERE j.expires_at > :date AND j.is_activated = :activated'
    )->setParameter('date', date('Y-m-d H:i:s', time()))->setParameter('activated', 1);
 
    return $query->getResult();
  }
}
```

到此为止今天的内容基本上已经完成了，你可以在浏览器上尽情的测试它们。所有无效的职位都已经从主页上消失了;即使你知道它们的 URL，也不再可以访问。但同时依旧可以访问使用Token的URL，在这种情况下，职位预览页还能根据职位状态显示相应的管理操作栏。