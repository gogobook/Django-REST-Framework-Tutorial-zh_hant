# Tutorial 3: Class Based Views

[src](http://django-rest-framework.org/tutorial/3-class-based-views.html)

在之前基於函數的View之外，我們還可以用基於類的view來實現我們的API view。正如我們即將看到的那樣，這樣的方式可以讓我們重新使用公用功能，並使我們保持代碼DRY。

## 1.用基於類的view重寫我們的API 
我們要用基於類的view來重寫剛才的`views.py`，如下重構所示：

```python
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer #序列化器，序列化/反序列化都靠它
    from django.http import Http404
    from rest_framework.views import APIView
    from rest_framework.response import Response #取代Render / HttpResponse
    from rest_framework import status

    class SnippetList(APIView):                     # 繼承APIView類，List表示會丟出所有資料
        """
        List all snippets, or create a new snippet.
        """
        def get(self, request, format=None):                     # 方法名稱是get, 表示是跟host 要資料，注意參數中有個request
            snippets = Snippet.objects.all()                     # Snippet的資料庫物件.all() 全部的
            serializer = SnippetSerializer(snippets, many=True)  # 全部的 丟到序列化器
            return Response(serializer.data)                     # 用Response 把序列化器的返回.data 包起丟回去

        def post(self, request, format=None):                    # 方法名稱是post，表示要把資料丟過來，注意參數中有個request
            serializer = SnippetSerializer(data=request.data)    # 把丟過來的資料(request.data) 丟到序列化器。
            if serializer.is_valid():                            # 驗證一下，通過就儲存，返回201，不然就丟error，返回400
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
目前看上去不錯。它看起來和我們之前寫的很相似，但我們在不同的HTTP方法見有了更好的分隔方式，我們還需要把示例的view也重構一下：

```python
    class SnippetDetail(APIView):                                # 繼承APIView類，Detail表示會丟出個別資料
        """
        Retrieve, update or delete a snippet instance.
        """
        def get_object(self, pk):
            try:                                                 # 用一個try包起來 有就返回個別資料，沒有就給404，注意這裡沒有request，因此是給下面的方法叫用的
                return Snippet.objects.get(pk=pk)
            except Snippet.DoesNotExist:
                raise Http404

        def get(self, request, pk, format=None):                 # 方法名稱是get，有參數request，先給get_object檢查一下，並且拿出資料庫物件，然後丟進序列化器
            snippet = self.get_object(pk)                        # 最後用Response返回序列化.data
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        def put(self, request, pk, format=None):                 # 方法名稱是put,要跟get一樣，最後把資料存進資料庫
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet, data=request.DATA)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        def delete(self, request, pk, format=None):              # 方法名稱是delete，要跟get一樣，最後執行刪除的動作。
            snippet = self.get_object(pk)
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)
```
做的不錯。它和我們之前寫的基於函數的view還是有些相像。

我們還需要對`snippets/urls.py`做一些小小的改動：

```python
    from django.conf.urls import url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.SnippetList.as_view()),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```
到目前為止，已經全部完成。你可以運行開發服務器，一切應該表現如初。

## 2. 使用mixins

使用基於類的view的最大好處就是可以讓我們方便的組合與重用。

目前我們用到的`create/retrieve/update/delete`操作，與我們建立的任何的模型後端API view都會很類似。其中的公共行為的一部分在REST framework's mixin類中實作。

我們來看看，我們如何在`views.py`中使用mixin類：

```python
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins
    from rest_framework import generics

    class SnippetList(mixins.ListModelMixin,
                      mixins.CreateModelMixin,
                      generics.GenericAPIView):  #注意generics.GenericAPIView 在最後面
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.list(request, *args, **kwargs)

        def post(self, request, *args, **kwargs):
            return self.create(request, *args, **kwargs)
```
我們將花點時間來解釋一下這裡到底發生了什麼事。我們用`GenericAPIView`構建了我們的view, 然後加上了`ListModelMixin`和`CreateModelMixin`.

基類提供了核心功能，mixin類提供了 `.list()`和 `.create()` 動作。我們然後顯式的把 `get`和 `post` 方法與合適的動作綁定在一起，非常簡單。

```python
    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):  #注意generics.GenericAPIView 在最後面
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        def put(self, request, *args, **kwargs):
            return self.update(request, *args, **kwargs)

        def delete(self, request, *args, **kwargs):
            return self.destroy(request, *args, **kwargs)
```
示例部分的實現也非常類似。再一次我們用`GenericAPIView`來提供核心功能，然後用mixins來提供`.retrieve()`, `.update()` 和 `.destroy()` actions.

## 3. 使用基於泛型類的view

使用mixin類可以讓我們重寫view時寫更少的代碼，但我們還可以更進一步，REST framework提供了一系列已經mixed-in的泛型view供我們使用。

```python
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics

    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer
```
Wow, 非常簡潔. 我們輕鬆了不少，而且代碼看起來優美，乾淨和符合Django的習慣。

在第四部分 part 4 of the tutorial, 我們將看看我們的API如何處理認證和權限。
