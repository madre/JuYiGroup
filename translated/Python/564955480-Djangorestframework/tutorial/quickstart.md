# 快速开始

我们将要创建一个简单的 API ，允许管理员用户去访问，编辑用户和组信息。

## 项目创建

创建一个新的 Django 项目，名字叫 `tutorial`, 然后在项目里建一个应用叫 `quickstart`。

    # Create the project directory
    mkdir tutorial
    cd tutorial

    # Create a virtualenv to isolate our package dependencies locally
    virtualenv env
    source env/bin/activate  # On Windows use `env\Scripts\activate`

    # Install Django and Django REST framework into the virtualenv
    pip install django
    pip install djangorestframework

    # Set up a new project with a single application
    django-admin.py startproject tutorial
    cd tutorial
    django-admin.py startapp quickstart
	cd ..

第一次同步数据库:

    python manage.py migrate

我们也要创建一个初始的用户名 `admin` ，密码 `password`。 在后面的例子中我们将会认证用户。

    python manage.py createsuperuser

如果我们建立完一个数据库并且初始用户，那么准备开始吧，打开应用目录开始写代码。。。

## 序列化

首先，我们先定义一些序列化类。我们创建一个文件 `tutorial/quickstart/serializers.py` ，将用于数据展示。

    from django.contrib.auth.models import User, Group
    from rest_framework import serializers


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = User
            fields = ('url', 'username', 'email', 'groups')


    class GroupSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Group
            fields = ('url', 'name')

注意，在这种情况下，我们使用了超链接关系 `HyperlinkedModelSerializer`。你也可以是使用主键和其他各种关系，但是超链接是 RESTful 很棒的设计。

## 视图

我们最好写点视图，打开 `tutorial/quickstart/views.py` 文件键入以下内容。

    from django.contrib.auth.models import User, Group
    from rest_framework import viewsets
    from tutorial.quickstart.serializers import UserSerializer, GroupSerializer


    class UserViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows users to be viewed or edited.
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer


    class GroupViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows groups to be viewed or edited.
        """
        queryset = Group.objects.all()
        serializer_class = GroupSerializer

我们要组合所有常用的行为成为类叫 `ViewSets`，而不是写许多的视图。

如果我们需要的话，能很容易的把这些变成单个的视图，但是使用 viewsets 可以保持视图的逻辑性组织又很简明。

注意，这的 viewset 类和 [frontpage example][readme-example-api] 中的有一点不同， 包括 `queryset` 和 `serializer_class` 属性，而不是 `model` 属性。

对于一般的情况，你可以在` viewset `类简单的设置一个 `model` 属性值，序列化和查询集合的属性值会自动为你生成。设置 `queryset` 或 `serializer_class` 属性可以更明确 API 的控制行为, 并且这也是大多数应用推荐的风格（其实这里就是扩展的地方）。

## 统一资源定位器

ok，现在我们把 API URLs连通。在这个文件 `tutorial/urls.py`。。。

    from django.conf.urls import url, include
    from rest_framework import routers
    from tutorial.quickstart import views

    router = routers.DefaultRouter()
    router.register(r'users', views.UserViewSet)
    router.register(r'groups', views.GroupViewSet)

    # Wire up our API using automatic URL routing.
    # Additionally, we include login URLs for the browseable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

由于我们用viewsets 代替 views，所以对于我们的 API 接口可以自动生成 URL 的配置，使用一个路由（router）类来简单的注册viewsets。

其次, 如果在 API 的 URL 中有很多控制，我们可以简单地降低到使用常规的基于类的视图，并明确地写明 URL 配置.

最后, 使用可浏览的 API ，默认提供了登录和登出视图，这是可选的，但是，如果你的 API 请求认证并且你想使用可浏览的 API 时就很有用。

## 设置

我们还要在 settings 文件里设置几个全局变量。如设置分页功能, 只有管理员用户访问功能。具体内容在 `tutorial/settings.py` 文件中。

    INSTALLED_APPS = (
        ...
        'rest_framework',
    )

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': ('rest_framework.permissions.IsAdminUser',),
        'PAGINATE_BY': 10
    }

ok，我们做完了

---

## 测试接口

现在我们测试下我们创建的 API。从命令行启动服务器。

    python ./manage.py runserver

也是使用命令行进入我们的 API ，linux系统下可以使用 `curl` 工具。

    bash: curl -H 'Accept: application/json; indent=4' -u admin:password http://127.0.0.1:8000/users/
    {
        "count": 2,
        "next": null,
        "previous": null,
        "results": [
            {
                "email": "admin@example.com",
                "groups": [],
                "url": "http://127.0.0.1:8000/users/1/",
                "username": "admin"
            },
            {
                "email": "tom@example.com",
                "groups": [                ],
                "url": "http://127.0.0.1:8000/users/2/",
                "username": "tom"
            }
        ]
    }

或者使用浏览器。。。

![Quick start image][image]

如果你用的是浏览器的话，确保在右上角可以控制登录。

很好，就这么简单。

如果你想更深入的理解 REST framework 是怎样结合覆盖参考 [the tutorial][tutorial], 或者浏览 [API guide][guide].

[readme-example-api]: ../#example
[image]: ../img/quickstart.png
[tutorial]: 1-serialization.md
[guide]: ../#api-guide
