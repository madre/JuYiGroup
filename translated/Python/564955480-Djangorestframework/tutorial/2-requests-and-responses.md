# 教程 2: 请求和响应

从这章开始，我们要真正接触 REST framework 的核心。我们先介绍两个基本的构建块。

## 请求对象

REST framework 介绍 `Request` 对象，它继承于 `HttpRequest`, 并且提供了许多灵活的请求解析。 `Request` 对象的核心功能是 `request.DATA` 属性，它很像 `request.POST`, 但是在 Web APIs 中更实用。

    request.POST  # Only handles form data.  Only works for 'POST' method.
    request.DATA  # Handles arbitrary data.  Works for 'POST', 'PUT' and 'PATCH' methods.

## 响应对象

REST framework 也介绍 `Response` 对象，这是一种 `TemplateResponse` 类型，获取没有被渲染的内容并用内容协议去确定正确的内容类型返回给客户端。

    return Response(data)  # Renders to content type as requested by the client.

## 状态码

HTTP 的状态码是数字码形式，在视图中识别不是很明显，如果你有一个代码错误，它不容易被注意到。REST framework 为状态码提供了很多明确的标识，像 `status` 模块中的 `HTTP_400_BAD_REQUEST`。使用这些看起来要比使用数字码要好。

## 封装 API 的视图

REST framework 提供了两个封装方法，你可以用它们来写 API 的视图。

1. `@api_view` 装饰器作用于视图的基本方法。
2. `APIView` 类作用于视图的基本类。

这些封装提供了几个功能，像确保在视图中你收到了实例，为 `Response` 对象添加内容以便协议内容可以被使用。

当非法使用的时候，封装提供了返回 `405 Method Not Allowed` 响应的行为。 当 `request.DATA` （请求）的格式不正确的时候，它会处理为 `ParseError` 的异常。

## 拼凑起来

ok，让我们用这些新的东西写几个视图吧。

在 `views.py` 文件中再也不需要 `JSONResponse` 类了，所以从现在开始删除它。按照这样做了，我们可以开始稍微重构视图了。

    from rest_framework import status
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer


    @api_view(['GET', 'POST'])
    def snippet_list(request):
        """
        List all snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        elif request.method == 'POST':
            serializer = SnippetSerializer(data=request.DATA)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

这个视图是前面视图例子的一个改进。它略有简明，如果我们使用 Forms API 的话，就会感觉到他们的相似了。我们使用了状态码，它使响应的意思更明显。

下面是针对一条 snippet 实例操作的视图,在 `views.py` 文件中。

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a snippet instance.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        elif request.method == 'PUT':
            serializer = SnippetSerializer(snippet, data=request.DATA)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        elif request.method == 'DELETE':
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

这些应该都会感到很熟悉吧，它和Django的视图没有太大的不同。

注意，我们不再明确地把我们的请求或响应转成一个给定的内容类型。 `request.DATA` 能处理 `json` 形式的请求, 也可以处理 `yaml` 形式和其他形式。同样，我们返回对象是有数据携带的，而且允许 REST framework 渲染返回数据成正确的内容类型。

## 给 URLs 添加可选格式后缀 

对于一个单一的内容类型，我们的响应不再是固定的，利用这样的事实，给 API 结尾添加格式后缀。使用格式后缀可以明确设定URLs的格式，意味着我们的 API 能够处理这样的URL [http://example.com/api/items/4.json][json-url].

像这样，给两个视图添加一个关键字参数 `format` 。

    def snippet_list(request, format=None):

and

    def snippet_detail(request, pk, format=None):

轻微更新下 `urls.py` 文件, 附加一套 `format_suffix_patterns` 除了现有的URLs。

    from django.conf.urls import patterns, url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

这里不一定要加上这些额外的URL模式, 但它为我们提供了一个简单的，清晰的创建特殊格式的方法。

## 看起来怎么样？

先从命令行测试的API，就像教程 [tutorial part 1][tut-1]中那样。一切都是十分的相似， 如果我们发送无效的请求，那么我们已经有了一些更好的错误处理。

我们可以像以前一样列出snippet模型的所有实例数据。

	curl http://127.0.0.1:8000/snippets/

	[{"id": 1, "title": "", "code": "foo = \"bar\"\n", "linenos": false, "language": "python", "style": "friendly"}, {"id": 2, "title": "", "code": "print \"hello, world\"\n", "linenos": false, "language": "python", "style": "friendly"}]

我们可以控制得到的响应数据的格式，或者使用 `Accept` 标题:

    curl http://127.0.0.1:8000/snippets/ -H 'Accept: application/json'  # Request JSON
    curl http://127.0.0.1:8000/snippets/ -H 'Accept: text/html'         # Request HTML

或者追加一个格式后缀:

    curl http://127.0.0.1:8000/snippets/.json  # JSON suffix
    curl http://127.0.0.1:8000/snippets/.api   # Browsable API suffix

一样的, 我们能控制发送请求的格式，使用 `Content-Type` 标题。

    # POST using form data
    curl -X POST http://127.0.0.1:8000/snippets/ -d "code=print 123"

    {"id": 3, "title": "", "code": "print 123", "linenos": false, "language": "python", "style": "friendly"}

    # POST using JSON
    curl -X POST http://127.0.0.1:8000/snippets/ -d '{"code": "print 456"}' -H "Content-Type: application/json"

    {"id": 4, "title": "", "code": "print 456", "linenos": true, "language": "python", "style": "friendly"}

现在在Web浏览器中打开API，这样访问 [http://127.0.0.1:8000/snippets/][devserver].

### 浏览功能

因为API将选择基于客户端请求的响应内容类型，当浏览器请求资源时，默认返回一个HTML格式的资源。允许 API 返回一个完全基于Web浏览HTML形式。

有一个Web浏览的 API 是一个巨大的应用性胜利，使开发和使用你的 API 更容易。它也大大的降低了其他开发人员检查和使用你的 API 的难度。

查看 [browsable api][browsable-api] 课题，有关浏览API功能的更多信息和如何自定制它。

## 下步做什么

在 [tutorial part 3][tut-3], 我们将开始使用基于类的视图和如何用 generic views 减少我们需要写的代码量。

[json-url]: http://example.com/api/items/4.json
[devserver]: http://127.0.0.1:8000/snippets/
[browsable-api]: ../topics/browsable-api.md
[tut-1]: 1-serialization.md
[tut-3]: 3-class-based-views.md
