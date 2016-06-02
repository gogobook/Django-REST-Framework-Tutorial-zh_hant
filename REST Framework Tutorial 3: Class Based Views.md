# Tutorial 3: Class Based Views

[src](http://django-rest-framework.org/tutorial/3-class-based-views.html)

在之前基於函數的View之外，我們還可以用基於類的view來實現我們的API view。正如我們即將看到的那樣，這樣的方式可以讓我們重用公用功能，並使我們保持代碼DRY。

用基於類的view重寫我們的API 
## 1. 我們要用基於類的view來重寫剛才的根view，如下重構所示：

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

目前看上去不錯。它看起來和我們之前寫的很相似，但我們在不同的HTTP方法見有了更好的分隔方式，我們還需要把示例的view也重構一下：

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

做的不錯。它和我們之前寫的基於函數的view還是有些相像。

我們還需要對URLconf做一些小小的改動：

    from django.conf.urls import patterns, url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = patterns('',
        url(r'^snippets/$', views.SnippetList.as_view()),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
    )

    urlpatterns = format_suffix_patterns(urlpatterns)

到目前為止，已經全部完成。你可以運行開發服務器，一切應該表現如初。

## 2. 使用mixins

使用基於類的view的最大好處就是可以讓我們方便的組合與重用。

剛才我們的`create/retrieve/update/delete`等函數實現在模型支撐API view下會很類似。其中的公共行為在REST framework's mixin類中實現了。

我們來看看，我們可以用mixin類來吧我們的view組合起來：

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins
    from rest_framework import generics

    class SnippetList(mixins.ListModelMixin,
                    mixins.CreateModelMixin,
                    generics.GenericAPIView):
        model = Snippet
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.list(request, *args, **kwargs)

        def post(self, request, *args, **kwargs):
            return self.create(request, *args, **kwargs)

我們將花點時間來解釋下這裡到底發生了什麼。我們用`GenericAPIView`構建了我們的view, 然後加上了`ListModelMixin`和`CreateModelMixin`.

基類提供了核心功能，mixin類提供了 `.list()`和 `.create()` 動作。我們然後顯式的把 `get`和 `post` 方法與合適的動作綁定在一起，非常簡單。

    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):
        model = Snippet
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        def put(self, request, *args, **kwargs):
            return self.update(request, *args, **kwargs)

        def delete(self, request, *args, **kwargs):
            return self.destroy(request, *args, **kwargs)

示例部分的實現也非常類似。這次我們用`GenericAPIView`來提供核心功能，然後用mixins來提供`.retrieve()`, `.update()` 和 `.destroy()` actions.

## 3. 使用基於泛型類的view

使用mixin類可以讓我們重寫view時寫更少的代碼，但我們還可以更進一步，REST framework提供了一系列已經mixed-in的泛型view供我們使用。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics

    class SnippetList(generics.ListCreateAPIView):
        model = Snippet
        serializer_class = SnippetSerializer

    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        model = Snippet
        serializer_class = SnippetSerializer

Wow, 非常簡潔. 我們輕鬆了不少，而且代碼看起來優美，乾淨和符合Django的習慣。

在第四部分 part 4 of the tutorial, 我們將看看我們的API如何處理認證和權限。
