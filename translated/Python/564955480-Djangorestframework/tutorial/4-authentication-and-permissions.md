# 教程 4: 认证和权限

目前我们的 API 没有任何限制谁可以编辑或删除 snippets 实例。我们想要一些更高级的行为来确保:

* snippets 实例与创建者关联。
* 只有认证过的用户可以创建 snippets 实例。
* 只有 snippet 的创建者可以更新和删除。
* 未认证通过的请求应该有只读入口。

## 为模型添加信息

我们要对 `Snippet` 模型类做两个改变。一个是，加两个字段，一个字段用来代表创建snippet 实例的用户。另一个用来存储表示高亮HTML的代码。

添加以下两个字段到 `Snippet` 应用中的 `models.py`文件中。

    owner = models.ForeignKey('auth.User', related_name='snippets')
    highlighted = models.TextField()

当模型保存时我们还需要确认，我们将突出显示高亮的字段，这里使用 `pygments` 代码高亮库。

我们需要一些额外的导入:

    from pygments.lexers import get_lexer_by_name
    from pygments.formatters.html import HtmlFormatter
    from pygments import highlight

在模型类里添加 `.save()` 方法:

    def save(self, *args, **kwargs):
        """
        Use the `pygments` library to create a highlighted HTML
        representation of the code snippet.
        """
        lexer = get_lexer_by_name(self.language)
        linenos = self.linenos and 'table' or False
        options = self.title and {'title': self.title} or {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos,
                                  full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super(Snippet, self).save(*args, **kwargs)

当一切都做完后，我们需要更新数据库表。通常我们会创造一个数据库迁移，就是为了表字段的变化，但是本篇教程的目的只是删除数据库并重新开始创建。

    rm tmp.db
    rm -r snippets/migrations
    python manage.py makemigrations snippets
    python manage.py migrate

您可能还需要创建一些不同的用户，用于测试 API。最快的方法是使用 `createsuperuser` 命令。

    python manage.py createsuperuser

## 给 User 模型添加端点 endpoints 

现在，我们已经有了一些用户，我们最好把这些用户信息体现在我们的 API 中。创建一个序列化是容易的。在 `serializers.py` 中添加以下内容:

    from django.contrib.auth.models import User

    class UserSerializer(serializers.ModelSerializer):
        snippets = serializers.PrimaryKeyRelatedField(many=True)

        class Meta:
            model = User
            fields = ('id', 'username', 'snippets')

由于 `'snippets'` 在 User 模型中是一个 *reverse* 关系，当使用 `ModelSerializer` 类时，默认是不包含的，所以我们需要添加一个明确字段。

在 `views.py` 中需要添加两个视图。 对于user的信息显示我们只提供只读视图，所以我们将使用generic里的 `ListAPIView` 和 `RetrieveAPIView` 类。

    from django.contrib.auth.models import User


    class UserList(generics.ListAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer


    class UserDetail(generics.RetrieveAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

确定导入 `UserSerializer` 类。

	from snippets.serializers import UserSerializer

最后，我们需要添加这些视图到 API 里，通过 URL 配置引用。在 `urls.py` 中添加下面的样品。

    url(r'^users/$', views.UserList.as_view()),
    url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),

## 关联 Snippets 和 Users 模型

现在，如果我们创造了一个 snippet 实例，也没有办法创建关联 snippet 的用户。用户信息没有像序列化那样形式发送过来，而是传入请求的属性。

处理这个问题的方法就是在 snippet 视图中重写 `.perform_create()` 方法，允许我们修改如何保存实例，并处理任何隐藏在请求或请求的 URL 中的信息。

在 `SnippetList` 视图类里，添加以下方法：

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

序列化的` create() `方法通过一个额外的` 'owner”`字段来保存，其验证过的数据来自于请求。

## 更新序列化

现在，snippet 模型与创建它的用户相关联起来，更新我们的 ` snippetserializer ` 显示那些。在 `serializers.py`中添加序列化字段:

    owner = serializers.ReadOnlyField(source='owner.username')

**注意**: 确定你已经在 `Meta` 类里列出 `'owner',` 字段。

此字段正在做一些很有趣的事。该 `source` 参数控制的属性是用来填充字段，并可以指向序列化实例的任何属性。它也能采用虚线法表示以上内容，在这种情况下，它将遍历给定的属性，跟使用Django模板语言方式类似。

我们添加的这个字段是非类型化的 `ReadOnlyField` 类，对比下其他类型的字段，例如 `CharField`, `BooleanField` 等...  非类型化的 `ReadOnlyField` 总是只读的，并且将会被用在序列化的数据的显示上，但是当反向序列化时，它是不能用于更新模型实例的。这里我们也可以使用 `CharField(read_only=True)`。

## 给视图添加必要的权限

现在，snippet 模型是与用户模型相关联的，我们要确保只有授权的用户能够创建，更新和删除 snippet 数据。

REST framework 包含多个权限类我们可以用来限制谁可以访问一个给定的视图。在这种情况下，我们寻找的是 `IsAuthenticatedOrReadOnly`，它将确保认证过的请求有读写权限，未认证过的请求有只读权限。

首先在视图中添加下面的导入

    from rest_framework import permissions

给两个视图类 `SnippetList` 和 `SnippetDetail` 添加以下内容。

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,)

## 为可浏览的 API 添加登录

如果你打开浏览器并导航到浏览 API 的时刻，你将发现你不能创建新的 snippet 实例了。为了这样做，我们需要像一个用户一样能够登录。
我们可以为用户添加一个登录视图，编辑在项目级目录下的 `urls.py` 文件。

在文件头部添加一下导入内容：

    from django.conf.urls import include

在文件结尾为可浏览API添加一个模式包括登录和注销的视图。

    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

模式的这部分 `r'^api-auth/'` 你可以定义成任何你想要的值。唯一的限制是 included 的部分值必须使用 `'rest_framework'` 做命名空间。

现在，如果你打开浏览器再次刷新页面，你会在页面的右上角看到“登录”链接。如果你用之前创建的用户登录，你就可以再次创建 snippet 模型实例了。

你已经创建了一些 snippet 实例，找到 '/users/' 的终点,注意，snippet 实例列表中包括一些主键，它们分别对应用户模型中的每一个 'snippets' 字段。

## 对象级权限

我们真的希望所有的模型数据是任何人都可以看到，但也要确保只有创建这个模型数据的用户才可以更新或删除它。

我们需要去创建一个自定义权限。

在 snippet 应用里创建一个新文件 `permissions.py`。

    from rest_framework import permissions


    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        Custom permission to only allow owners of an object to edit it.
        """

        def has_object_permission(self, request, view, obj):
            # Read permissions are allowed to any request,
            # so we'll always allow GET, HEAD or OPTIONS requests.
            if request.method in permissions.SAFE_METHODS:
                return True

            # Write permissions are only allowed to the owner of the snippet.
            return obj.owner == request.user

现在我们可以添加自定义权限到 snippet 实例后端，在 `SnippetDetail` 类里编辑 `permission_classes` 的属性值。

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

确认已经导入 `IsOwnerOrReadOnly` 类。

    from snippets.permissions import IsOwnerOrReadOnly

现在，你再打开一个浏览器，如果你用创建这个实例的用户登录，你会发现“删除”和“修改”的行为会出现在snippet实例后端。

## API 的认证

因为我们现在有了 API 的一组权限，如果我们要编辑 snippet 模型数据，我们需要验证要求。我们还没有建立任何认证类 [authentication classes][authentication]，所以默认的 `SessionAuthentication`  和 `BasicAuthentication` 目前被应用。

当我们使用API通过Web浏览器进行交互时，我们要登录，浏览器会话将提供对请求所需的认证。

如果我们正在用API程序进行互动，需要为每一次的请求提供认证凭据。

如果我们试图不携带认证信息而去创造一个snippet实例，我们会得到一个错误：

    curl -i -X POST http://127.0.0.1:8000/snippets/ -d "code=print 123"

    {"detail": "Authentication credentials were not provided."}

我们可以通过包括之前创建过的用户中的一个用户名和密码信息使一个请求成功。

    curl -X POST http://127.0.0.1:8000/snippets/ -d "code=print 789" -u tom:password

    {"id": 5, "owner": "tom", "title": "foo", "code": "print 789", "linenos": false, "language": "python", "style": "friendly"}

## 总结

现在我们的 Web API 有了一个相当细粒度的权限集,他们已经为系统用户和 snippet 模型创建了结点。

在教程的第5部分 [part 5][tut-5] ，将会看到我们如何通过为我们的高亮代码创造一个HTML终端而把所有都联系起来，利用超链接的关系在系统提高我们的API的凝聚力。

[authentication]: ../api-guide/authentication.md
[tut-5]: 5-relationships-and-hyperlinked-apis.md
