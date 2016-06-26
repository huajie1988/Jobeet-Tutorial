# Symfony2 Jobeet Day 18: AJAX

昨天我们为Jobeet实现了一个非常强大的搜索引擎，这都应该归功于Zend Lucene库。接下来，为了提高搜素引擎的响应性，我们将利用AJAX技术使其转换为另一种方式。

首先我们必须明确一点，就是无论客户端是否启用javascript，我们的表单搜索功能都应该能够正常工作。为此我们将使用[unobtrusive JavaScript](http://en.wikipedia.org/wiki/Unobtrusive_JavaScript)方式来实现这一功能。除此之外unobtrusive JavaScript方式还能使客户端代码更好的分离。

## 安装jQuery

前往jQuery官网下载最新版本的jQuery，并将其放到js目录下`src/Ens/JobeetBundle/Resources/public/js/`
之后运行以下命令，确保Symfony能正确的引用这个js资源

> **php app/console assets:install web --symlink**

## 包含jQuery

因为我们要在所有页面中引用jQuery，所以需要在layout中将其包含进来：

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->

<!-- ... -->
    {% block javascripts %}
        <script type="text/javascript" src="{{ asset('bundles/ensjobeet/js/jquery-2.0.3.min.js') }}"></script>
    {% endblock %}
<!-- ... -->
```

## 事件绑定

要实现实时搜索意味着每次用户在搜索栏输入之后都会自动触发去服务器调用搜索功能，服务器将会返回需要的信息并更新一些页面内容而非将整个页面刷新。

jQuery的主要思想是当页面完全加载后向其中的DOM节点进行事件绑定，而非通过在HTML中的on*()属性为其添加事件操作。这种方式的好处在于，即使你在浏览器中禁用了JavaScript，仅仅只会造成事件未被注册，但你的表单仍然可以像以前一样正常工作。

第一步是拦截用户在搜索框中输入的每一个按键︰

```javascript
$('#search_keywords').keyup(function(key)
{
    if (this.value.length >= 3 || this.value == '')
    {
        // do something
    }
});
```

> *请不要现在就添加代码，因为我们还要修改它。最终完成的JavaScript 代码将会在下一节中添加到layout中*

每次用户输入一个按键，jQuery都会执行上面定义的匿名函数，但只有当用户输入超过3个字符或者全部清空搜索框时才会真正执行里面的代码。

AJAX向服务器发出调用非常简单，只需要在DOM元素上使用load()方法︰

```javascript
$('#search_keywords').keyup(function(key)
{
    if (this.value.length >= 3 || this.value == '')
    {
        $('#jobs').load(
            $(this).parents('form').attr('action'), { query: this.value + '*' }
        );
    }
});
```

为了处理AJAX请求调用，我们将使用和原来一样的action，并在稍后修改它。

最后但重要的是，如果我们已启用了JavaScript，则将要删除搜索按钮︰

```javascript
$('.search input[type="submit"]').hide();
```

## 用户反馈

每当你执行一次AJAX调用，页面都将不会立即更新。因为浏览器会在页面更新之前等待服务器的响应。在此期间，我们需要为用户提供一个视觉反馈，以使他能知道有些正在获取数据。

一个公认的方法是在AJAX调用期间显示一个loader图标。为此我们需要更新`layout`并添加theloader图片，且其在默认情况下隐藏。

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twig -->

<!-- ... -->
    <div class="search">
        <h2>Ask for a job</h2>
        <form action="{{ path('ens_job_search') }}" method="get">
            <input type="text" name="query" value="{{ app.request.get('query') }}" id="search_keywords" />
            <input type="submit" value="search" />
            <img id="loader" src="{{ asset('bundles/ensjobeet/images/loader.gif') }}" style="vertical-align: middle; display: none" />
            <div class="help">
                Enter some keywords (city, country, position, ...)
            </div>
        </form>
    </div>
<!-- ... -->
```

现在你已经有了全部所需的HTML代码，接着创建一个search.js文件，并把我们之前所写的JavaScript包含进来。

```javascript
// src/Ens/JobeetBundle/Resources/public/js/search.jsJavaScript

$(document).ready(function()
{
    $('.search input[type="submit"]').hide();

    $('#search_keywords').keyup(function(key)
    {
        if(this.value.length >= 3 || this.value == '') {
            $('#loader').show();
            $('#jobs').load(
                $(this).parent('form').attr('action'),
                { query: this.value ? this.value + '*' : this.value },
                function() {
                    $('#loader').hide();
                }
            );
        }
    });
});
```

别忘了运行命令行让Symfony能找到资源

> **php app/console assets:install web --symlink**

对了还有layout文件需要更新一下，把js包含进来

```html
<!-- src/Ens/JobeetBundle/Resources/views/layout.html.twigXHTML -->

<!-- ... -->
    {% block javascripts %}
        <script type="text/javascript" src="{{ asset('bundles/ensjobeet/js/jquery-2.0.3.min.js') }}"></script>
        <script type="text/javascript" src="{{ asset('bundles/ensjobeet/js/search.js') }}"></script>
    {% endblock %}
<!-- ... -->
```

## AJAX的后台响应

如果用户启用了JavaScript，则jQuery会拦截搜索框中的所有输入，并调用search action。否则，当用户按回车提交表单之后相同的action也应该能被使用。

所以search action现在需要确定是否来自AJAX调用。为此我们可以通过request对象的isXmlHttpRequest()方法来判断，每当请求是来自AJAX调用时，它将返回true。

```php
// src/Ens/JobeetBundle/Controller/JobController.php

use Symfony\Component\HttpFoundation\Response;

class JobController extends Controller
{  
    // ...

    public function searchAction(Request $request)
    {
        $em = $this->getDoctrine()->getManager();
        $query = $this->getRequest()->get('query');

        if(!$query) {
            if(!$request->isXmlHttpRequest()) {
                return $this->redirect($this->generateUrl('ens_job'));
            } else {
                return new Response('No results.');
            }
        }

        $jobs = $em->getRepository('EnsJobeetBundle:Job')->getForLuceneQuery($query);

        if($request->isXmlHttpRequest()) {

            return $this->render('EnsJobeetBundle:Job:list.html.twig', array('jobs' => $jobs));
        }

        return $this->render('EnsJobeetBundle:Job:search.html.twig', array('jobs' => $jobs));
    }
}
```

如果搜索不返回任何结果，我们需要显示一个消息而不是一个空白页。这里我们将只返回一个简单的字符串 ︰

```php
// src/Ens/JobeetBundle/Controller/JobController.php

    public function searchAction(Request $request)
    {
        $em = $this->getDoctrine()->getManager();
        $query = $this->getRequest()->get('query');

        if(!$query) {
            if(!$request->isXmlHttpRequest()) {
                return $this->redirect($this->generateUrl('ens_job'));
            } else {
                return new Response('No results.');
            }
        }

        $jobs = $em->getRepository('EnsJobeetBundle:Job')->getForLuceneQuery($query);

        if($request->isXmlHttpRequest()) {
            if('*' == $query || !$jobs || $query == '') {
                return new Response('No results.');
            }

            return $this->render('EnsJobeetBundle:Job:list.html.twig', array('jobs' => $jobs));
        }

        return $this->render('EnsJobeetBundle:Job:search.html.twig', array('jobs' => $jobs));
    }
```

## 测试AJAX

由于Symfony不能模拟JavaScript，所以当你要用它测试AJAX时需要手动添加jQuery和其他主要的 JavaScript库所需的请求头。

```php
// src/Ens/JobeetBundle/Tests/Controller/JobControllerTst.php

class JobControllerTest extends WebTestCase
{
    // ...

    public function testSearch()
    {
        $client = static::createClient();

        $crawler = $client->request('GET', '/job/search');
        $this->assertEquals('EnsJobeetBundleControllerJobController::searchAction', $client->getRequest()->attributes->get('_controller'));

        $crawler = $client->request('GET', '/job/search?query=sens*', array(), array(), array(
            'X-Requested-With' => 'XMLHttpRequest',
        ));
        $this->assertTrue($crawler->filter('tr')->count()== 2);
    }
}
```

在第17天的课程时，我们用 Zend Lucene 库来实现搜索引擎。今天我们使用jQuery来使它具有更好的响应性。Symfony框架提供了所有基本工具来帮助我们更轻松的创建MVC应用程序，同时也提供了很多好用的组件。这使得我们可以使用最好的工具来完成这项工作。

明天，我们将解释如何国际化Jobeet网站。