# 教程6 视图集（ViewSets）和路由（Routers）

REST framework 处理 `ViewSets` 时包含一个抽象概念，它允许开发者专注于建模状态和 API 的互动，基于共同的约定，许可 URL 结构自动被处理。

`ViewSets` 类几乎跟 `View` 类相同，除此之外，他们提供的操作如 `read` 或 `update`，没有 `get` 或 `put` 方法。

当 `ViewSet` 类被实例化为一组视图时，它只绑定一套处理方法。使用 `Router` 类的典型例子是定义的 URL 配置很复杂。

## 重构 ViewSets

把我们当前的所有视图，重构成视图集。

首先，重构 `UserList` 和 `UserDetail` 成一个单独的 `UserViewSet`。我们可以删除两个视图，用一个单独的类替换：

    from rest_framework import viewsets

    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        This viewset automatically provides `list` and `detail` actions.
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

这里我们使用了 `ReadOnlyModelViewSet` 类默认自动提供只读操作。当我们使用这个规范视图时，仍然要准确设置  `queryset`  和 `serializer_class` 的属性值，但我们不需要提供两个不同的类相同的信息。

其次，我们将要替换  `SnippetList`, `SnippetDetail` 和 `SnippetHighlight` 视图类。我们可以删除这三个视图了，并用一个单独的类替换。

    from rest_framework.decorators import detail_route

    class SnippetViewSet(viewsets.ModelViewSet):
        """
        This viewset automatically provides `list`, `create`, `retrieve`,
        `update` and `destroy` actions.

        Additionally we also provide an extra `highlight` action.
        """
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer
        permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                              IsOwnerOrReadOnly,)

        @detail_route(renderer_classes=[renderers.StaticHTMLRenderer])
        def highlight(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

        def pre_save(self, obj):
            obj.owner = self.request.user

为了获得默认的读和写的全套操作，我们使用 `ModelViewSet` 类。

注意，在创建一个名为 `highlight` 的自定义操作时，我们使用了 `@detail_route` 修饰器。这个修饰器能用于添加任何自定义的不符合标准的 `create`/`update`/`delete` 形式的入口。

自定义操作使用 `@detail_route` 修饰器将会响应 `GET` 请求。如果我们想要一个响应 `POST` 请求的操作，我们要使用 `methods` 参数。

## 视图集（ViewSets）绑定 URL

当我们定义URL配置的时候，这处理方法只能绑定了作用。让我们先明确地创建一套出自'ViewSets'的视图，看下背后发生了什么。

在 `urls.py` 文件中，我们绑定 `ViewSet` 类到一套具体视图。

    from snippets.views import SnippetViewSet, UserViewSet, api_root
    from rest_framework import renderers

    snippet_list = SnippetViewSet.as_view({
        'get': 'list',
        'post': 'create'
    })
    snippet_detail = SnippetViewSet.as_view({
        'get': 'retrieve',
        'put': 'update',
        'patch': 'partial_update',
        'delete': 'destroy'
    })
    snippet_highlight = SnippetViewSet.as_view({
        'get': 'highlight'
    }, renderer_classes=[renderers.StaticHTMLRenderer])
    user_list = UserViewSet.as_view({
        'get': 'list'
    })
    user_detail = UserViewSet.as_view({
        'get': 'retrieve'
    })

注意我们是如何从每个视图集类创建多个视图，通过 HTTP 方式绑定所需作用的每个视图。

现在我们已经把我们的资源投入到具体的视图，我们可以像往常一样在 URL 中注册视图。 

    urlpatterns = format_suffix_patterns([
        url(r'^$', api_root),
        url(r'^snippets/$', snippet_list, name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
        url(r'^users/$', user_list, name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
    ])

## 使用路由

因为我们使用的是 `ViewSet` 类而不是 `View` 类，我们实际上不需要设计自己的 URL。资源到视图和 URL 的绑定是可以被自动处理的，使用 `Router` 类。我们需要做的就是用一个路由注册这个应用下的所有视图，静等它来自动绑定。

这是我们的 `urls.py` 文件内容。

    from django.conf.urls import url, include
    from snippets import views
    from rest_framework.routers import DefaultRouter

    # Create a router and register our viewsets with it.
    router = DefaultRouter()
    router.register(r'snippets', views.SnippetViewSet)
    router.register(r'users', views.UserViewSet)

    # The API URLs are now determined automatically by the router.
    # Additionally, we include the login URLs for the browseable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

用路由注册viewsets与提供一个URL模式很类似。包含两个参数-视图的 URL 前缀和viewsets本身。

我们使用的  `DefaultRouter` 类可以自动为我们的根级视图创建 API，所以现在可以从视图中删除 `api_root` 方法。

## 权衡 views 和 viewsets

使用 viewsets 是一个非常有用的抽象概念。它有助于确保 URL 协议将在您的 API 是一致的，最大限度地减少你需要编写的代码量，让您专注于你的 API 提供的作用和显示，而不是分心于 URL 中的细节。

这并不意味着它总是采取正确的方法。有一个类似的权衡考虑使用基于类的视图还是基于方法的视图。使用 viewsets 比创建你的视图少了明确性。

## 回顾我们的工作

一个令人难以置信的少量的代码，我们现在有了一个完整的 Web API，这是完全的Web浏览，来完成认证，每个对象的权限，和多个渲染格式。

我们已经走过了设计过程的每一步，并看到如果我们需要定制任何，我们如何逐步简单地使用规范的Django视图来完成。

你可以在GitHub上审查最后的教程代码[tutorial code][repo]，或尝试沙盒测试 [the sandbox][sandbox]。

## 继续向前

我们已经达到了教程的结束点。如果你想获得更多的参与REST framework 项目，这里有几个地方你可以开始：

* 在GitHub[GitHub][github]贡献--审查和提交问题和拉取请求。
* 加入 REST framework 的讨论组[REST framework discussion group][group]，并帮助
* 在 Twitter [the author][twitter]上跟作者说你好。

**现在去建立好东西**


[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[github]: https://github.com/tomchristie/django-rest-framework
[group]: https://groups.google.com/forum/?fromgroups#!forum/django-rest-framework
[twitter]: https://twitter.com/_tomchristie
