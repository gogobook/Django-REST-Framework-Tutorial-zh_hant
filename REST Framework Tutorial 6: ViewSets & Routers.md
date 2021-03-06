# Tutorial 6:ViewSets & Routers

REST framework包含一個抽象概念來處理`ViewSets`，它使得開發者可以集中精力對 API的state和interactions建模，留下URL構造被自動處理，基於共同約定。

`ViewSet`類幾乎和`View`類一樣，除了它們提供像`read`或`update`操作，但沒有處理`get`和`put`的方法。

一個`ViewSet`類只是在最後一刻綁定到一組方法處理程序，在它被實例化到一組視圖的時候，通常是使用一個`Router`類——為你處理定義`URL conf`的複雜性。
## 使用ViewSets來重構

讓我們取出當前集合的views，使用view sets將它們重構。

首先讓我們重構我們的`UserList`和`UserDetail`成為單一個`UserViewSet`。我們可以移除兩個views，用一個類來替換它們。

```python    
    from rest_framework import viewsets
    
    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        This viewset automatically provides `list` and `detail` actions.
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer
```
在這裡我們將使用`ReadOnlyModelViewSet`類來自動地提供默認的'read-only'操作。正如我們在使用一般的views做的，我們還是會設置`queryset`和`serializer_class`屬性，但我們不再需要提供相同的信息給兩個獨立的類。

下一步我們將替換`SnippetList`,`SnippetDetial`和`SnippetHighlight`類。我們可以移除這三個views，再次用一個類來替換它們。
```py
from rest_framework.decorators import action
from rest_framework.response import Response

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

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```
這次我們將使用`ModelViewSet`類為了得到默認read和write操作的完整集合。

注意我們還使用`@action`修飾符來創建一個自定義動作名為`highlight`。這個修飾符可以用來添加任何自定義endpoints，不用符合標準的`create`/`update`/`delete`樣式。

用`@action`修飾符創建的自定義動作將會對`GET`做出response 。我們也可以使用`method`參數代替如果我們想要一個對`POST`請求做出response 的動作。

The URLs for custom actions by default depend on the method name itself. If you want to change the way url should be constructed, you can include `url_path` as a decorator keyword argument.

## 明確地绑定ViewSets到URLs

handler method僅僅在我們定義URLConf的時候綁定到動作(actions)上。去看看蓋子下發生了什麼事，首先從我們的ViewSets明確地創建一個views集合。

在`snippets/urls.py`文件中我們綁定了我們的`ViewSet`類到一個具體views的集合。

    from snippets.views import SnippetViewSet, UserViewSet，api_root
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

注意我們是如何從每個`ViewSet`類創建多個views，通過綁定http methods到每個view所需的動作(by binding the http methods to the required aciotn for each view)

現在我們已經綁定我們的資源到具體的views，我們可以像往常一樣註冊views和URL conf。

    urlpatterns = format_suffix_patterns([
        url(r'^$', api_root),
        url(r'^snippets/$', snippet_list, name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
        url(r'^users/$', user_list, name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
    ])

## 使用Routers

因為我們使用`ViewSet`類而不是`View`類，我們實際上不用自己設計URL conf。連接resources到views和urls的約定可以使用`Router`類自動處理。我們要做的僅僅是用一個router註冊適當的view集合，and let it do the rest

這裡我們重連接`snippets/urls.py`文件

```python    
    from django.conf.urls import url, include
    from snippets import views
    from rest_framework.routers import DefaultRouter

    # Create a router and register our viewsets with it.
    router = DefaultRouter()
    router.register(r'snippets', views.SnippetViewSet)
    router.register(r'users', views.UserViewSet)

    # The API URLs are now determined automatically by the router.
    # Additionally, we include the login URLs for the browsable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
    ]
```
用router註冊viewsets和提供一個urlpattern很像。我們有兩個參數——給views的URL前綴和viewset自身。

`DefaultRouter`類自動為我們創建`API root vies`，所以我們現在可以從我們的`views`方法中刪除`api_root`方法。

## 權衡 views VS viewsets

使用viewset可以是一個真正有用的抽象。它幫助確保URL約定可以對你的API始終如一，使你需要編寫的代碼數量最小化，使得你可以集中精力在你API的交互和表現上，而不是URL conf的細節上。

這並不意味著它總是正確的方法。一個類似的權衡在使用class-based views代替function-based views上。單獨構建views比使用viewsets更加清楚。

在教程的第7部分，我們將檢視如何增加一個api schema，並且使用client library 或 command line tool來與我們的api互動。