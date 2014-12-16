# 教程 3: 基于类的视图

我们也可以用基于类的视图写我们的 API 视图, 而不是使用基于方法的视图。正如我们将看到这是一个能够重公用功能的强大模式，把代码简洁化 [DRY][dry]。

## 用基于类的视图重写我们的 API

我们将开始通过重写根视图作做为一个基于类的视图。涉及到的是一点 `views.py` 的重构。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from django.http import Http404
    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import status


    class SnippetList(APIView):
        """
        List all snippets, or create a new snippet.
        """
        def get(self, request, format=None):
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        def post(self, request, format=None):
            serializer = SnippetSerializer(data=request.DATA)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

到目前为止，都很好。它看起来非常类似于以前的案例，但我们依不同的HTTP请求方式做更好的分离。我们还需要在文件 `views.py` 中更新实例视图。

    class SnippetDetail(APIView):
        """
        Retrieve, update or delete a snippet instance.
        """
        def get_object(self, pk):
            try:
                return Snippet.objects.get(pk=pk)
            except Snippet.DoesNotExist:
                raise Http404

        def get(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        def put(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet, data=request.DATA)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        def delete(self, request, pk, format=None):
            snippet = self.get_object(pk)
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

看起来很好。但它仍然非常类似基于函数的视图。

现在我们使用基于类的视图，我们还需要重构 `urls.py` 。

    from django.conf.urls import patterns, url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.SnippetList.as_view()),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

ok，我们已经做完。如果您运行开发服务器，每个功能都跟以前一样在运作。

## 使用混合（mixin）

使用基于类的视图的一个巨大收获是，它允许我们轻松地组合可复用少量的行为。

到目前为止,我们已经使用创建/检索/更新/删除操作，都将是非常相似于任何我们创建的模型支持的 API 视图。那些少量共同行为都会在 REST framework 的混合类中实现。

让我们看看，如何使用 mixins 类构建这个视图。这是 `views.py` 内容。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins
    from rest_framework import generics

    class SnippetList(mixins.ListModelMixin,
                      mixins.CreateModelMixin,
                      generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.list(request, *args, **kwargs)

        def post(self, request, *args, **kwargs):
            return self.create(request, *args, **kwargs)

我们会花一点时间审视下这里到底发生了什么。我们用 `GenericAPIView`，`ListModelMixin` 和 `CreateModelMixin` 创建视图。

基类提供了核心功能，mixin 类提供了 `.list()` 和 `.create()` 方法。然后对于适当的行为明确地绑定 `get` 和 `post` 方法。足够简单吧。

    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        def put(self, request, *args, **kwargs):
            return self.update(request, *args, **kwargs)

        def delete(self, request, *args, **kwargs):
            return self.destroy(request, *args, **kwargs)

同样的。我们再次使用 `GenericAPIView` 类提供的核心功能，mixins 还提供了 `.retrieve()`， `.update()` 和 `.destroy()` 方法。

## 使用基于 generic 类的视图

使用 mixin 类重写了视图代码量比以前明显少了，但我们可以再向前走一步。REST framework 提供了一套在 generic 视图里已经混合好的类，可以更多的减少 `views.py` 内容。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics


    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer


    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

哇，那真的很简洁，我们已经得到了大量的自由，我们的代码看起来像漂亮的，干净的，地道的Django。

下一步我们将转移到 [part 4 of the tutorial][tut-4], 那我们将会看到如何处理认证和权限的 API。

[dry]: http://en.wikipedia.org/wiki/Don't_repeat_yourself
[tut-4]: 4-authentication-and-permissions.md
