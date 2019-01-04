# 路由器
> 資源路由允許您快速聲明給定資源控制器的所有公共路由。而不是為索引聲明單獨的路由......資源豐富的路由在一行代碼中聲明它們。

> - Ruby on Rails文檔

某些Web框架（如Rails）提供了自動確定應用程序以決定URL應如何映射到傳入請求的邏輯的功能。

REST framework增加了對Django的自動URL路由的支持，並為您提供了一種簡單，快速和一致的方法，將您的視圖邏輯連接到一組URL。

## 用法
這是一個使用簡單URL conf的示例SimpleRouter。
```py
    from rest_framework import routers

    router = routers.SimpleRouter()
    router.register(r'users', UserViewSet)
    router.register(r'accounts', AccountViewSet)
    urlpatterns = router.urls
```
該`register()`方法有兩個必需參數：

* `prefix`- 用於此組路由的URL前綴。
* `viewset` - 視圖集類。

（可選）您還可以指定其他參數：
* `base_name` - 用於創建的URL名稱的基礎。如果未設置，將根據視圖集的`queryset`屬性自動生成基本名稱（如果有）。請注意，如果視圖集不包含`queryset`屬性，則必須`base_name`在註冊視圖集時進行設置。

上面的示例將生成以下URL模式：

* URL pattern: `^users/$`  Name: `'user-list'`
* URL pattern: `^users/{pk}/$`  Name: `'user-detail'`
* URL pattern: `^accounts/$`  Name: `'account-list'`
* URL pattern: `^accounts/{pk}/$`  Name: `'account-detail'`

**注意**：該`base_name`參數用於指定視圖名稱模式的初始部分。在上面的例子中，這是user或account部分。

通常，您**不需要**指定`base_name`參數，但如果您有一個已定義自定義`get_queryset`方法的視圖集，則視圖集可能沒有設置`.queryset`屬性。如果您嘗試註冊該視圖集，您將看到如下錯誤：

    'base_name' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.
這意味著您需要`base_name`在註冊視圖集時顯式設置參數，因為它無法從模型名稱自動確定。

### 使用include路由器
.urls路由器實例上的屬性只是URL模式的標準列表。有關如何包含這些URL的方式有很多種。

例如，您可以附加router.urls到現有視圖列表中......
```py
    router = routers.SimpleRouter()
    router.register(r'users', UserViewSet)
    router.register(r'accounts', AccountViewSet)

    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
    ]

    urlpatterns += router.urls
```
或者你也可以使用Django的include功能......
```py
    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
        url(r'^', include(router.urls)),
    ]
```
您可以使用include應用程序命名空間：
```py
    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
        url(r'^api/', include((router.urls, 'app_name'))),
    ]
```
或者是應用程序和實例命名空間：
```py
    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
        url(r'^api/', include((router.urls, 'app_name'), namespace='instance_name')),
    ]
```
有關更多詳細信息，請參閱Django的URL命名空間文檔和includeAPI參考。

**注意**：如果將命名空間與超鏈接序列化程序一起使用，則還需要確保view_name序列化程序上的任何參數都能正確反映命名空間。在上面的示例中，您需要包含一個參數，例如 view_name='app_name:user-detail'超鏈接到用戶詳細信息視圖的序列化程序字段。

自動view_name生成使用類似的模式%(model_name)-detail。除非您的模型名稱實際發生衝突，否則在使用超鏈接序列化程序時，最好不要命名Django REST Framework視圖。

### 額外的action 的路由
視圖集可以通過使用裝飾器裝飾方法來標記用於路由的額外動作@action。這些額外的操作將包含在生成的路由中。例如，給定類set_password上的方法UserViewSet：
```py
    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import action

    class UserViewSet(ModelViewSet):
        ...

        @action(methods=['post'], detail=True, permission_classes=[IsAdminOrIsSelf])
        def set_password(self, request, pk=None):
            ...
```
將生成以下路由：

* URL pattern: `^users/{pk}/set_password/$`
* URL name: `'user-set-password'`

默認情況下，URL模式基於方法名稱，URL名稱是ViewSet.base_name連字符和方法名稱的組合。如果您不想對這些值中的任何一個使用默認值，則可以為裝飾器提供url_path和url_name參數@action。

例如，如果要將自定義操作的URL更改為^users/{pk}/change-password/$，則可以編寫：
```py
    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import action

    class UserViewSet(ModelViewSet):
        ...

        @action(methods=['post'], detail=True, permission_classes=[IsAdminOrIsSelf],
                url_path='change-password', url_name='change_password')
        def set_password(self, request, pk=None):
            ...
```
上面的示例現在將生成以下URL模式：

* URL path: `^users/{pk}/change-password/$`
* URL name: `'user-change_password'`

# API指南
## SimpleRouter
該路由器包括一套標準的途徑`list`，`create`，`retrieve`，`update`，`partial_update`和`destroy`行動。視圖集還可以使用`@action`裝飾器標記要路由的其他方法。

上述
```html
<table border=1>
    <tr><th>URL Style</th><th>HTTP Method</th><th>Action</th><th>URL Name</th></tr>
    <tr><td rowspan=2>{prefix}/</td><td>GET</td><td>list</td><td rowspan=2>{base_name}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{url_path}/</td><td>GET, or as specified by `methods` argument</td><td>`@action(detail=False)` decorated method</td><td>{base_name}-{url_name}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/</td><td>GET</td><td>retrieve</td><td rowspan=4>{base_name}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{url_path}/</td><td>GET, or as specified by `methods` argument</td><td>`@action(detail=True)` decorated method</td><td>{base_name}-{url_name}</td></tr>
</table>
```
默認情況下，創建的URL SimpleRouter附加一個尾部斜杠。通過在實例化路由器時將trailing_slash參數設置為可以修改此行為False。例如：

`router = SimpleRouter(trailing_slash=False)`

在Django中，尾部斜杠是常規的，但在某些其他框架（如Rails）中默認不使用。您選擇使用哪種樣式主要是偏好問題，儘管某些javascript框架可能需要特定的路由樣式。

路由器將匹配包含除斜線和句點字符之外的任何字符的查找值。對於更嚴格（或寬鬆）的查找模式，請lookup_value_regex在視圖集上設置該屬性。例如，您可以將查找限制為有效的UUID：

    class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
        lookup_field = 'my_model_id'
        lookup_value_regex = '[0-9a-f]{32}'

## DefaultRouter
此路由器與`SimpleRouter`上述類似，但另外包括一個默認的API根視圖，它返回一個包含所有列表視圖的超鏈接的response 。它還為可選的.json樣式格式後綴生成路由。

上述
<table border=1>
    <tr><th>URL Style</th><th>HTTP Method</th><th>Action</th><th>URL Name</th></tr>
    <tr><td>[.format]</td><td>GET</td><td>automatically generated root view</td><td>api-root</td></tr></tr>
    <tr><td rowspan=2>{prefix}/[.format]</td><td>GET</td><td>list</td><td rowspan=2>{base_name}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{url_path}/[.format]</td><td>GET, or as specified by `methods` argument</td><td>`@action(detail=False)` decorated method</td><td>{base_name}-{url_name}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/[.format]</td><td>GET</td><td>retrieve</td><td rowspan=4>{base_name}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{url_path}/[.format]</td><td>GET, or as specified by `methods` argument</td><td>`@action(detail=True)` decorated method</td><td>{base_name}-{url_name}</td></tr>
</table>

與`SimpleRouter`一樣URL上的尾部斜杠，可以通過在實例化路由器時將trailing_slash參數設置為刪除False路徑。

`router = DefaultRouter(trailing_slash=False)`
## 自定義路由器
實現自定義路由器不是您經常需要做的事情，但如果您對API的URL結構有特定要求，那麼它可能很有用。這樣做允許您以可重用的方式封裝URL結構，以確保您不必為每個新視圖顯式編寫URL模式。

實現自定義路由器的最簡單方法是子類化現有路由器類之一。該.routes屬性用於模擬將映射到每個視圖集的URL模式。該.routes屬性是一個Route名為元組的列表。

Route命名元組的參數是：

url：表示要路由的URL的字符串。可能包含以下格式字符串：

* `{prefix}` - The URL prefix to use for this set of routes.
* `{lookup}` - The lookup field used to match against a single instance.
* `{trailing_slash}` - Either a '/' or an empty string, depending on the `trailing_slash` argument.

mapping：HTTP方法名稱到視圖方法的映射

name：reverse調用中使用的URL的名稱。可能包含以下格式字符串：

* `{base_name}` - 用於創建的URL名稱的基礎。

initkwargs：實例化視圖時應傳遞的任何其他參數的字典。需要注意的是detail，base_name和suffix參數保留給視圖集中內省和也使用可瀏覽的API生成視圖名稱和麵包屑鏈接。

## 自定義動態路由
您還可以自定義@action裝飾器的路由方式。DynamicRoute在.routes列表中包含命名元組，將detail參數設置為適合基於列表和基於詳細信息的路由。除此之外detail，參數DynamicRoute是：

url：表示要路由的URL的字符串。可以包含與之相同的格式字符串Route，並且還接受{url_path}格式字符串。

name：reverse調用中使用的URL的名稱。可能包含以下格式字符串：

* `{base_name}` - The base to use for the URL names that are created.
* `{url_name}` - The `url_name` provided to the `@action`.

initkwargs：實例化視圖時應傳遞的任何其他參數的字典。

## 例
以下示例將僅路由到list和retrieveactions，並且不使用尾部斜杠約定。
```py
    from rest_framework.routers import Route, DynamicRoute, SimpleRouter

    class CustomReadOnlyRouter(SimpleRouter):
        """
        A router for read-only APIs, which doesn't use trailing slashes.
        """
        routes = [
            Route(
                url=r'^{prefix}$',
                mapping={'get': 'list'},
                name='{base_name}-list',
                detail=False,
                initkwargs={'suffix': 'List'}
            ),
            Route(
                url=r'^{prefix}/{lookup}$',
                mapping={'get': 'retrieve'},
                name='{base_name}-detail',
                detail=True,
                initkwargs={'suffix': 'Detail'}
            ),
            DynamicRoute(
                url=r'^{prefix}/{lookup}/{url_path}$',
                name='{base_name}-{url_name}',
                detail=True,
                initkwargs={}
            )
        ]
```
讓我們來看看我們CustomReadOnlyRouter為簡單視圖集生成的路線。

`views.py`：
```py
    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        A viewset that provides the standard actions
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer
        lookup_field = 'username'

        @action(detail=True)
        def group_names(self, request, pk=None):
            """
            Returns a list of all the group names that the given
            user belongs to.
            """
            user = self.get_object()
            groups = user.groups.all()
            return Response([group.name for group in groups])
```
urls.py：
```py
    router = CustomReadOnlyRouter()
    router.register('users', UserViewSet)
    urlpatterns = router.urls
```
將生成以下映射...

<table border=1>
    <tr><th>URL</th><th>HTTP Method</th><th>Action</th><th>URL Name</th></tr>
    <tr><td>/users</td><td>GET</td><td>list</td><td>user-list</td></tr>
    <tr><td>/users/{username}</td><td>GET</td><td>retrieve</td><td>user-detail</td></tr>
    <tr><td>/users/{username}/group_names</td><td>GET</td><td>group_names</td><td>user-group-names</td></tr>
</table>

有關設置.routes屬性的另一個示例，請參閱SimpleRouter該類的源代碼。

## 進階定制路由器
如果要提供完全自定義的行為，可以覆蓋BaseRouter並覆蓋該get_urls(self)方法。該方法應檢查已註冊的視圖集並返回URL模式列表。可以通過訪問self.registry屬性來檢查已註冊的前綴，視圖集和基本名稱元組。

您可能還希望覆蓋該get_default_base_name(self, viewset)方法，或者base_name在向路由器註冊視圖集時始終顯式設置參數。

# 第三方包
還提供以下第三方軟件包。

DRF嵌套路由器
的DRF-嵌套路由器包提供路由器和關係字段用於與嵌套資源工作。

ModelRouter（wq.db.rest）
所述wq.db包提供了一個先進ModelRouter延伸類（和單個實例）DefaultRouter用register_model()的API。就像Django一樣admin.site.register，唯一需要的參數rest.router.register_model是模型類。將從模型和全局配置推斷出url前綴，序列化程序和視圖集的合理默認值。
```py
    from wq.db import rest
    from myapp.models import MyModel

    rest.router.register_model(MyModel)
```
DRF-擴展
該DRF-extensions軟件包提供了路由器創建嵌套viewsets，收藏級控制器與定制的端點名。