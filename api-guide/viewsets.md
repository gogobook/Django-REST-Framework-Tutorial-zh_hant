# ViewSets

> 在路由確定了要用於請求的控制器後，您的控制器負責理解請求並生成適當的輸出。

- Ruby on Rails文檔。

Django REST framwork允許您將一組相關視圖的邏輯組合在一個類中，稱為ViewSet。在其他框架中，您可能會發現概念上類似的實現，命名為「資源」或「控制器」。

`ViewSet`類只是一種基於類的視圖，它不提供任何方法處理程序，例如`.get()`或`.post()`，而是提供諸如`.list()`和`.create()`之類的動作(action)。

ViewSet的方法處理程序僅在使用.as_view()方法最後確定視圖的時候綁定到相應的操作。

通常，您不會在urlconf的viewset中顯式地註冊視圖，而是使用一個路由器類註冊viewset，它會自動地為您決定urlconf。

### Example
讓我們定義一個簡單的viewset，它可以用於列表或檢索系統中的所有用戶。

```py
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
    """
    A simple ViewSet for listing or retrieving users.
    """
    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```
If we need to, we can bind this viewset into two separate views, like so:
假如我們需要，我們可以連結這個viewset到二個不同的視圖，就像這樣。
```py
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```
Typically we wouldn't do this, but would instead register the viewset with a router, and allow the urlconf to be automatically generated.
通常我們不需要這麼做，而是用一個router 註冊這個viewset，以允許urlconf被自動地的產生。
```py
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, base_name='user')
urlpatterns = router.urls
```
Rather than writing your own viewsets, you'll often want to use the existing base classes that provide a default set of behavior. For example:
相對於撰寫你自己的viewsets, 你通常想要使用既有的基礎類別以提供一組預設的行為，例如。

```py
class UserViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing user instances.
    """
    serializer_class = UserSerializer
    queryset = User.objects.all()
```
使用ViewSet類比使用視圖類有兩個主要優點。

- 重複的邏輯可以組合成一個類。在上面的例子中，我們只需要指定queryset一次，它將在多個視圖中使用。
- 通過使用路由器，我們不再需要處理連接自己的URL。

這兩種情況都需要權衡。使用常規視圖和URL confs更加明確，並給予您更多的控制。如果您想要快速地啟動和運行，或者當您有一個大型的API，並且想要始終執行一致的URL配置時，ViewSets是很有幫助的。

### ViewSet actions
The default routers included with REST framework will provide routes for a standard set of create/retrieve/update/destroy style actions, as shown below:
REST framework 所帶有的預設路由器將為create/retrieve/update/destroy style actions 的標準組合提供路由，如下所示。
```py
class UserViewSet(viewsets.ViewSet):
    """
    Example empty viewset demonstrating the standard
    actions that will be handled by a router class.

    If you're using format suffixes, make sure to also include
    the `format=None` keyword argument for each action.
    """

    def list(self, request):
        pass

    def create(self, request):
        pass

    def retrieve(self, request, pk=None):
        pass

    def update(self, request, pk=None):
        pass

    def partial_update(self, request, pk=None):
        pass

    def destroy(self, request, pk=None):
        pass
```
### Introspecting ViewSet actions
### 查看 ViewSet actions

During dispatch, the following attributes are available on the ViewSet.
當在分派時，下列屬性將可用於ViewSet

- base_name - the base to use for the URL names that are created.
- action - the name of the current action (e.g., list, create).
- detail - boolean indicating if the current action is configured for a list or detail view.
- suffix - the display suffix for the viewset type - mirrors the detail attribute.
- name - the display name for the viewset. This argument is mutually exclusive to suffix.
- description - the display description for the individual view of a viewset.
You may inspect these attributes to adjust behaviour based on the current action. For example, you could restrict permissions to everything except the list action similar to this:
你可以在現有action 上查看這些屬性以調整行為，例如 你可對所有東西限制權限除了列表 action，相似如下。

```py
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdmin]
    return [permission() for permission in permission_classes]
```
### Marking extra actions for routing
如果您有可以路由的特別方法，您可以將它們標記為`@action` decorator。與常規action一樣，可以為物件列表或單個實例指定額外的操作。為了表示這一點，將`detail`參數設置為`True`或`False`。路由器將相應地配置它的URL模式。例如，`DefaultRouter`將配置細節操作以在其URL模式中包含`pk`。
A more complete example of extra actions:
```py
from django.contrib.auth.models import User
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    A viewset that provides the standard actions
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @action(methods=['post'], detail=True)
    def set_password(self, request, pk=None):
        user = self.get_object()
        serializer = PasswordSerializer(data=request.data)
        if serializer.is_valid():
            user.set_password(serializer.data['password'])
            user.save()
            return Response({'status': 'password set'})
        else:
            return Response(serializer.errors,
                            status=status.HTTP_400_BAD_REQUEST)

    @action(detail=False)
    def recent_users(self, request):
        recent_users = User.objects.all().order('-last_login')

        page = self.paginate_queryset(recent_users)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(recent_users, many=True)
        return Response(serializer.data)
```
The decorator can additionally take extra arguments that will be set for the routed view only. For example:
裝飾器可取得額外的參數，這僅可被routed view 所設定。例如
```py
    @action(methods=['post'], detail=True, permission_classes=[IsAdminOrIsSelf])
    def set_password(self, request, pk=None):
       ...
```
These decorator will route GET requests by default, but may also accept other HTTP methods by setting the methods argument. For example:
```py
    @action(methods=['post', 'delete'], detail=True)
    def unset_password(self, request, pk=None):
       ...
```
The two new actions will then be available at the urls `^users/{pk}/set_password/$` and `^users/{pk}/unset_password/$`

To view all extra actions, call the `.get_extra_actions()` method.

## API參考
### ViewSet
`ViewSet`類繼承自`APIView`。您可以使用任何標準屬性，如`permission_classes`、`authentication_classes`來控制viewset中的API策略。

`ViewSet`類不提供任何action的實作。為了使用ViewSet類，您將重寫類並顯式地定義action實作。

### GenericViewSet
`GenericViewSet`類繼承了`GenericAPIView`，並提供了預設的`get_object`、`get_queryset`方法和其他通用視圖基本行為，但不包括任何預設的操作。

為了使用`GenericViewSet`類，您將重寫類，或者在需要的mixin類中使用mixin，或者顯式地定義action實作。

### ModelViewSet
`ModelViewSet`類繼承了`GenericAPIView`，並包含了各種操作的實現，混合了各種mixin類的行為。

`ModelViewSet`類提供的操作是`.list()`， `.retrieve()`， `.create()`， `.update()`， `.partial_update()`和`.destroy()`。

#### 例子
因為`ModelViewSet`擴展了`GenericAPIView`，所以通常需要至少提供`queryset`和`serializer_class`屬性。例如:
```py
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]
```
注意，您可以使用`GenericAPIView`提供的任何標準屬性或方法重寫。例如，要使用一個動態地確定它應該運行的`queryset`的`viewSet`，您可以這樣做:
```py
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing the accounts
    associated with the user.
    """
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]

    def get_queryset(self):
        return self.request.user.accounts.all()
```
注意，如果從您的`ViewSet`中刪除了`queryset`屬性，任何關聯的路由器將無法自動派生您的模型的`base_name`，因此您將不得不指定`base_name` kwarg作為路由器註冊的一部分。

還要注意,雖然這類提供了全套的create/list/retrieve/update/destroy actions 在預設情況下,您可以通過使用標準的限制可用操作許可類。

### ReadOnlyModelViewSet
`ReadOnlyModelViewSet`類也繼承了`GenericAPIView`。與`ModelViewSet`一樣，它還包括各種操作的實現，但與`ModelViewSet`不同的是，它只提供「只讀」操作、`.list()`和`.retrieve()`。

#### 例子
與ModelViewSet一樣，您通常需要至少提供`queryset`和`serializer_class`屬性。例如:
```py
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A simple ViewSet for viewing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
```
同樣，與`ModelViewSet`一樣，您可以使用任何標準屬性和方法覆蓋到`GenericAPIView`。

## 自定義ViewSet基類
您可能需要提供定製的`ViewSet`類，這些類沒有完整的`ModelViewSet`操作集，或者以其他方式定製該行為。

### 例子
要創建提供創建、列表和檢索操作的基本`ViewSet`類，從GenericViewSet繼承，並混合所需的操作:
```py
from rest_framework import mixins

class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                mixins.ListModelMixin,
                                mixins.RetrieveModelMixin,
                                viewsets.GenericViewSet):
    """
    A viewset that provides `retrieve`, `create`, and `list` actions.

    To use it, override the class and set the `.queryset` and
    `.serializer_class` attributes.
    """
    pass
```
通過創建您自己的基本`ViewSet`類，您可以提供可以在您的API上的多個ViewSet中重用的公共行為。