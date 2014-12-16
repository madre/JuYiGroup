# 教程5:关系与超链接的API

目前,关系在我们的 API 里是通过使用主键来体现的。在这部分的教程中我们将提高 API 的凝聚力和可发现性，而不是使用超链接的关系。

## 给根 API 创建一个入口

现在我们有 `snippets` 和 `users` 的端点，但我们的 API 没有一个单一的入口点。新建一个，我们将使用一个普通的基于方法的视图和前面介绍的 `@api_view` 装饰器。 在你的 `snippets/views.py` 文件中添加如下： 

    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from rest_framework.reverse import reverse


    @api_view(('GET',))
    def api_root(request, format=None):
        return Response({
            'users': reverse('user-list', request=request, format=format),
            'snippets': reverse('snippet-list', request=request, format=format)
        })

注意，为了返回完整的URL我们使用 REST framework 的reverse方法。这里的'user-list'是 url(r'^users/$', UserList.as_view(), name='user-list')中的name值，注意写法是否一致。

## 为高亮snippet模型创建一个入口

另一个明显的东西仍然失踪从我们Pastebin的API是代码高亮显示终端。

不同于所有其他 API 入口，我们不想使用JSON，而只是一个HTML形式。REST framework 提供了两种样式的 HTML 渲染。一个用模板处理 HTML 渲染，另一个处理是预渲染 HTML。我们这个入口要使用第二种渲染。

当创建代码高亮视图时，我们需要考虑的另一件事是，这儿没有我们可以使用的具体的 generic 视图。我们不打算返回一个对象实例，而是对象实例的一个属性。

取代使用一个具体 generic 视图，我们将使用基类去显示实例，创造我们自己的 `.get()` 方法。在 `snippets/views.py` 里添加：

    from rest_framework import renderers
    from rest_framework.response import Response

    class SnippetHighlight(generics.GenericAPIView):
        queryset = Snippet.objects.all()
        renderer_classes = (renderers.StaticHTMLRenderer,)

        def get(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

像往常一样，我们需要为已创建的视图添加 URL 的配置。我们将在 `snippets/urls.py` 中添加一个新的 URL 模式。

    url(r'^$', 'api_root'),

然后添加snippet 模型高亮的 URL 模式:

    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),

## API 的超链接

处理实体之间的关系是 Web API 设计的一个更具挑战性的方面。这有许多不同的方式，我们可以选择表现他们之间关系：

* 使用主键
* 使用实体间超链接
* 使用在相关实体上一个唯一识别字段
* 使用相关实体的默认字符串显示
* 嵌套相关实体的父类显示
* 一些自定义显示

REST framework 支持以上的所有形式，并可以将他们应用到正向或反向的关系上，或者应用到自定义管理者像 generic 的外键。

在这种情况下，我们喜欢在实体间使用超链接的形式。为了这样做，我们替换已有的 `ModelSerializer`，修改为继承序列化 `HyperlinkedModelSerializer`。

`HyperlinkedModelSerializer` 与 `ModelSerializer`有以下的不同点：

* 默认情况下它不包含  `pk` 字段
* 它包含一个 `url` 字段，用 `HyperlinkedIdentityField`
* 关系使用 `HyperlinkedRelatedField`, 而不使用 `PrimaryKeyRelatedField`

我们可以很容易地使用超链接重新编写现有的序列化，在 `snippets/serializers.py` 文件中添加：

    class SnippetSerializer(serializers.HyperlinkedModelSerializer):
        owner = serializers.Field(source='owner.username')
        highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

        class Meta:
            model = Snippet
            fields = ('url', 'highlight', 'owner',
                      'title', 'code', 'linenos', 'language', 'style')


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail')

        class Meta:
            model = User
            fields = ('url', 'username', 'snippets')

注意，我们已经添加了一个新的 `'highlight'` 字段。这个字段跟  `url` 字段的属性一样，除此之外，它还指出 URL 模式下的视图名称 `'snippet-highlight'` ，替换视图名称 `'snippet-detail'`。

因为我们包含 URL 的格式后缀，像 `'.json'`, 我们还需要表明 `highlight` 字段会返回任何格式后缀的超链接，所以应该使用`'.html'`后缀。 

## 确定我们的 URL 样式已经命名

如果我们要有一个超链接的 API，我们需要确保 URL 模式的名字。让我们看看哪种 URL 模式需要命名。

* 根级别的 API 涉及 `'user-list'` 和 `'snippet-list'`。
* 我们的 snippet 序列化包含一个字段，涉及 `'snippet-highlight'`。
* 我们的 user 序列化包含一个字段，涉及 `'snippet-detail'`。
* 我们的 snippet 和 user 包含 `'url'`字段，默认会涉及  `'{model_name}-detail'`，在这种情况下将会是 `'snippet-detail'` 和 `'user-detail'`。

在URL中添加所有这些名字后，最终看到的  `snippets/urls.py` 如下：

    # API endpoints
    urlpatterns = format_suffix_patterns([
        url(r'^$', views.api_root),
        url(r'^snippets/$',
            views.SnippetList.as_view(),
            name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$',
            views.SnippetDetail.as_view(),
            name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
            views.SnippetHighlight.as_view(),
            name='snippet-highlight'),
        url(r'^users/$',
            views.UserList.as_view(),
            name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$',
            views.UserDetail.as_view(),
            name='user-detail')
    ])

    # Login and logout views for the browsable API
    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

## 添加分页

user 模型和 snippet 模型的列表视图最终会返回大量的实例，所以我们想确定我们分页的结果，允许客户端 API 遍历每个页。

我们能修改默认分页使用的形式，修改 `settings.py` 文件。添加以下内容：

    REST_FRAMEWORK = {
        'PAGINATE_BY': 10
    }

注意REST framework 框架的设置在 settings 里所有命名空间中是一个单一的字典，名字叫 'REST_FRAMEWORK'，它能很好的区分你的其他项目的设置。

如果你需要的话，你也可以自己定义分页的样式，但是现在我们要坚持使用默认的。

## 浏览 API

如果我们打开浏览器，导航到 API，你会发现，通过以下链接你可以简单地看到正在使用的 API。

你也可以在 snippet 实例中看到 'highlight' 链接，它会以高亮的 HTML 的形式显示。

在教程[part 6][tut-6]中，我们会看到如何使用  ViewSets 和 Routers 减少我们创建 API 的大量代码。

[tut-6]: 6-viewsets-and-routers.md
