# 教程 1: 序列化

## 简介

本教程将创建一个简单醒目的 Web API（网络应用程序接口）代码。它会介绍一路上整理 REST framework 的各个组成部分，并且会给你一个关于每部分如何配合在一起的综合理解。

本教程是比较深入的，所以在开始之前，你应该拿点饼干和一杯你喜爱的饮料。如果你想快速的浏览，你应该去看[quickstart]文档

---

**注意**: 在GitHub库[tomchristie/rest-framework-tutorial][repo]中的本教程代码是可用的，已经在线上完整实现，也是用于测试的沙盒版本， [可用][sandbox]。

---

## 建立一个新的环境

在我们做任何事情之前，我们应创建一个新的虚拟环境, 使用工具 [virtualenv]。这将确保我们的项目包配置与任何其他正在使用的项目包保持很好的隔离。

    :::bash
    virtualenv env
    source env/bin/activate

现在，我们在一个virtualenv环境里，安装我们需要的项目包。

    pip install django
    pip install djangorestframework
    pip install pygments  # 我们将用这个包来显示代码高亮

**注意:** 在任何时候，退出virtualenv环境, 只需输入 `deactivate`。更多信息参考 [virtualenv documentation][virtualenv].

## 开始

好的，我们已经准备好了。开始了, 我们先创建一个新的项目。

    cd ~
    django-admin.py startproject tutorial
    cd tutorial

然后，我们来创建一个应用，用于创建一个简单的 Web API。

    python manage.py startapp snippets

我们需要把新的应用名`snippets`和应用名`rest_framework`加到`INSTALLED_APPS`中。 让我们编辑文件 `tutorial/settings.py`:

    INSTALLED_APPS = (
        ...
        'rest_framework',
        'snippets',
    )

我们也需要配置url, 在文件 `tutorial/urls.py` 中, 要包含新应用 snippets 的 urls。

    urlpatterns = [
        url(r'^', include('snippets.urls')),
    ]

ok，我们准备启动。

## 创建一个模型

本教程的目的是通过创建一个简单的存储代码片段的`Snippet` 模型而开始我们的项目。接下来编辑`snippets/models.py`文件。 注意: 优秀的程序是有注释的。虽然你会在我们教程的代码库版本里找到它们，但在这里我们已经省略了，还是让我们把重点放在代码本身上吧。

    from django.db import models
    from pygments.lexers import get_all_lexers
    from pygments.styles import get_all_styles

    LEXERS = [item for item in get_all_lexers() if item[1]]
    LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
    STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


    class Snippet(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        title = models.CharField(max_length=100, blank=True, default='')
        code = models.TextField()
        linenos = models.BooleanField(default=False)
        language = models.CharField(choices=LANGUAGE_CHOICES,
                                    default='python',
                                    max_length=100)
        style = models.CharField(choices=STYLE_CHOICES,
                                 default='friendly',
                                 max_length=100)

        class Meta:
            ordering = ('created',)

我们也需要为snippet模型创建一个初始的迁移（会看到app下有相应的文件夹和文件生成），用于首次同步数据库。

    python manage.py makemigrations snippets
    python manage.py migrate

## 创建一个序列化类

我们需要开始的第一件事情是，Web API 提供了一种序列化和反序列化snippet模型实例成类似 `json`样子的方法。 我们可以声明序列化，它的工作原理跟Django的表单很相似。在`snippets`（app名字）的目录下创建一个文件，命名为 `serializers.py` 并添加以下内容。

    from django.forms import widgets
    from rest_framework import serializers
    from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


    class SnippetSerializer(serializers.Serializer):
        pk = serializers.IntegerField(read_only=True)
        title = serializers.CharField(required=False,
                                      max_length=100)
        code = serializers.CharField(style={'type': 'textarea'})
        linenos = serializers.BooleanField(required=False)
        language = serializers.ChoiceField(choices=LANGUAGE_CHOICES,
                                           default='python')
        style = serializers.ChoiceField(choices=STYLE_CHOICES,
                                        default='friendly')

        def create(self, validated_attrs):
            """
            Create and return a new `Snippet` instance, given the validated data.
            """
            return Snippet.objects.create(**validated_attrs)

        def update(self, instance, validated_attrs):
            """
            Update and return an existing `Snippet` instance, given the validated data.
            """
            instance.title = validated_attrs.get('title', instance.title)
            instance.code = validated_attrs.get('code', instance.code)
            instance.linenos = validated_attrs.get('linenos', instance.linenos)
            instance.language = validated_attrs.get('language', instance.language)
            instance.style = validated_attrs.get('style', instance.style)
            instance.save()
            return instance

序列化类的第一部分定义序列化/反序列化的域（字段）。当调用 `serializer.save()`方法时，`create()` 或 `update()`方法就会去创建或者修改完整的实例。

序列化类和Django的 `Form` 类非常相似，包括在各种字段上的相似验证标志，像 `required`, `max_length` 和 `default`。

字段的标示也可以控制序列化在特定的情况下怎样去显示，例如去渲染HTML的时候。在Django的 `Form`类中，`style={'type': 'textarea'}`和 `widget=widgets.Textarea`的作用是等价的。对于如何控制显示API的样子，这点是非常有用的。我们在之后的教程中会看到。

使用 `ModelSerializer` 类，确实可以节省我们自己的时间。我们之后会看到，但是现在我们要全面的学习序列化（从基础学起）。  

## 使用序列化

在我们深入学习之前，我们要熟悉使用新的序列化类，让我们进入到Django的shell命令行（交换层）。

    python manage.py shell

ok, 首先我们要写几个导入（from...import...）语句，然后让我们来创建两条Snippet模型的数据。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser

    snippet = Snippet(code='foo = "bar"\n')
    snippet.save()

    snippet = Snippet(code='print "hello, world"\n')
    snippet.save()

现在我们已经有几个Snippet模型的实例了，我们来看下那些实例中的一条序列化数据。

    serializer = SnippetSerializer(snippet)
    serializer.data
    # {'pk': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}

此时，我们已经把模型实例转化成python的原生数据类型了。最后，我们把数据渲染成 `json`就是序列化的过程了。

    content = JSONRenderer().render(serializer.data)
    content
    # '{"pk": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'

反序列化也是一样的。首先我们解析一个数据流成python的原生数据类型...

    # 这里的导入将使用 `StringIO.StringIO` 或者 `io.BytesIO`
    # 分别对应Python 2 或者 Python 3的版本。
    from rest_framework.compat import BytesIO

    stream = BytesIO(content)
    data = JSONParser().parse(stream)

...然后我们恢复原生数据类型为一个完整的对象实例。

    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.object
    # <Snippet: Snippet object>

注意到了么，API的工作原理跟forms是多么的相似啊。 当用我们的序列化写views时，它们的相似就变的更明显了。

我们也能序列化模型实例的集合，只需在序列化类中简单地添加一个 `many=True` 参数。

    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data
    # [{'pk': 1, 'title': u'', 'code': u'foo = "bar"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}, {'pk': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}]

## 使用模型序列化

我们的 `SnippetSerializer` 类正在复制 `Snippet` 模型的大量信息。如果我们能保持我们的代码更简洁一点就好了。

与Django提供的 `Form` 类和 `ModelForm` 类同样的方式，REST framework 也提供了 `Serializer` 类和 `ModelSerializer` 类。

来看看用 `ModelSerializer` 类来重构我们的序列化。再次打开 `snippets/serializers.py` 文件, 编辑 `SnippetSerializer` 类。

    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ('id', 'title', 'code', 'linenos', 'language', 'style')

序列化有一个很好的属性是你可以检查一个序列化实例的所有字段，你可以打印出来。进入Django的shell交互页面，操作如下：

    >>> from snippets.serializers import SnippetSerializer
    >>> serializer = SnippetSerializer()
    >>> print repr(serializer)  # In python 3 use `print(repr(serializer))`
    SnippetSerializer():
        id = IntegerField(label='ID', read_only=True)
        title = CharField(allow_blank=True, max_length=100, required=False)
        code = CharField(style={'type': 'textarea'})
        linenos = BooleanField(required=False)
        language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
        style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...

记住， `ModelSerializer` 类做不了什么魔术般的变化，它只是一个简单创建序列化类的快捷方式：

* 自动确认字段的设置。
* 默认实现 `create()` 和 `update()` 方法。

## 用序列化写具有Django规范的视图

我们来看下用新的序列化类如何写一些API的视图。此刻，我们不使用 REST framework 的任何特性，用Django的规范来写一个视图。

我们先创建一个 HttpResponse 的子类，用来把所有的返回数据转成 `json` 格式。

编辑 `snippets/views.py` 文件，添加以下内容。

    from django.http import HttpResponse
    from django.views.decorators.csrf import csrf_exempt
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

    class JSONResponse(HttpResponse):
        """
        An HttpResponse that renders its content into JSON.
        """
        def __init__(self, data, **kwargs):
            content = JSONRenderer().render(data)
            kwargs['content_type'] = 'application/json'
            super(JSONResponse, self).__init__(content, **kwargs)

API的本质就是一个支持展示所有现存 snippet 实例数据或者创建一个 snippet 实例的视图。

    @csrf_exempt
    def snippet_list(request):
        """
        List all code snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return JSONResponse(serializer.data)

        elif request.method == 'POST':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data, status=201)
            return JSONResponse(serializer.errors, status=400)

值得注意的是，因为我们要把数据从客户端提交到这个视图，而客户端没有一个 CSRF（Cross-site request forgery 跨站请求伪造）的验证，所以要给视图的这个方法加一个修饰器 `csrf_exempt`。 这通常不是你要做的，REST framework 视图实际上用了比这更明智的行为，但是现在这么写是为了实现我们的目的。

我们也需要对应某一个 snippet 实例操作的视图，查询，更新或者删除这个 snippet 实例。

    @csrf_exempt
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a code snippet.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JSONResponse(serializer.data)

        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data)
            return JSONResponse(serializer.errors, status=400)

        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)

最后，我们需要把这些视图连起来。创建一个 `snippets/urls.py` 文件：

    from django.conf.urls import patterns, url
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
    ]

值得注意的是，这里有两个边缘情况我们此刻还不能适当的处理。 如果我们发送一个不规则的 `json` 或者一个由方法生成的请求，这个视图是无法处理的，然后我们的服务器就会返回一个 500 "server error" 的错误信息。但是现在的仍然要这么做。

## 测试第一个Web API

现在我们启动一个简单的服务。

退出shell的交互界面。。。

	quit()

。。。启动Django的开发服务。

	python manage.py runserver

	Validating models...

	0 errors found
	Django version 1.4.3, using settings 'tutorial.settings'
	Development server is running at http://127.0.0.1:8000/
	Quit the server with CONTROL-C.

在另一个终端窗口测试服务接口。

我们可以得到snippet所有实例的一个列表。

	curl http://127.0.0.1:8000/snippets/

	[{"id": 1, "title": "", "code": "foo = \"bar\"\n", "linenos": false, "language": "python", "style": "friendly"}, {"id": 2, "title": "", "code": "print \"hello, world\"\n", "linenos": false, "language": "python", "style": "friendly"}]

或者我们能通过 snippet 实例的 id 得到一个详细的 snippet 实例信息。

	curl http://127.0.0.1:8000/snippets/2/

	{"id": 2, "title": "", "code": "print \"hello, world\"\n", "linenos": false, "language": "python", "style": "friendly"}

同样的，你可以在浏览器地址中键入那些URLs得到相同的 json 数据。

## 我们现在在哪

到目前为止，我们做的都很好，我们已经有了一个感觉跟 Django 的 Forms API 十分相似的序列化 API，还有一些 Django 规范的视图。

此刻，我们的 API 视图还做不了其他特别的事，除了对 `json` 响应, 在处理边缘情况还有些错误，我们想清理，但是这是 Web API 的一个功能。

我们会在[part 2 of the tutorial][tut-2]看到怎么样去改善这些。

[quickstart]: quickstart.md
[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[virtualenv]: http://www.virtualenv.org/en/latest/index.html
[tut-2]: 2-requests-and-responses.md
