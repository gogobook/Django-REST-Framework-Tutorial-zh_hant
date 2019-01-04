# 過濾
> Manager提供的根QuerySet描述資料庫表中的所有物件。但是，通常情況下，您只需要選擇完整物件集的一部分。

> - Django文檔

REST Framework的通用列表視圖的預設行為是返回模型管理器(model manager)的整個查詢集。通常您會希望您的API限制查詢集返回的項目。

過濾任何視圖的查詢集的最簡單方法是子類化`GenericAPIView`以覆寫其`.get_queryset()`方法。

覆寫此方法允許您以多種不同方式自定義視圖返回的查詢集。

### 針對當前用戶進行過濾
您可能需要過濾查詢集以確保只返回與當前經過身份驗證的用戶發出請求相關的結果。

您可以通過基於值為`request.user`來進行篩選以完成此操作。

例如：
``` py
from myapp.models import Purchase
from myapp.serializers import PurchaseSerializer
from rest_framework import generics

class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases
        for the currently authenticated user.
        """
        user = self.request.user
        return Purchase.objects.filter(purchaser=user)
```
### 根據URL進行過濾
另一種過濾方式可能涉及限制基於URL某部分的查詢集。

例如，如果你的URL配置包含這樣的條目：

`url('^purchases/(?P<username>.+)/$', PurchaseList.as_view()),`
然後，您可以編寫一個視圖，返回由URL的用戶名部分過濾的購買查詢集：
```py
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases for
        the user as determined by the username portion of the URL.
        """
        username = self.kwargs['username']
        return Purchase.objects.filter(purchaser__username=username)
```
### 針對查詢參數進行過濾
過濾初始查詢集的最後一個例子是根據url中的查詢參數確定初始查詢集。

我們可以重寫`.get_queryset()`以處理諸如URL之類的URL `http://example.com/api/purchases?username=denvercoder9`，並且只有在username參數包含在URL中時才過濾查詢集：
```py
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        Optionally restricts the returned purchases to a given user,
        by filtering against a `username` query parameter in the URL.
        """
        queryset = Purchase.objects.all()
        username = self.request.query_params.get('username', None)
        if username is not None:
            queryset = queryset.filter(purchaser__username=username)
        return queryset
```
## 泛型過濾
除了能夠覆蓋缺省查詢集外，REST Framework還包含對通用過濾後端的支持，這些通用過濾後端允許您輕鬆構建複雜的搜索和過濾器。

通用過濾器也可以在可瀏覽的API和管理API中將自己呈現為HTML控件。

過濾示例

### 設置過濾器後端
可以使用該`DEFAULT_FILTER_BACKENDS`設置全局設置預設的過濾器後端。例如。
```py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```
您還可以使用`GenericAPIView`基於類的視圖，在每個視圖或每個視圖的基礎上設置過濾器後端。
```py
import django_filters.rest_framework
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics

class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (django_filters.rest_framework.DjangoFilterBackend,)
```
### 過濾和物件查找
請注意，如果為視圖配置了過濾器後端，那麼以及用於過濾列表視圖的過濾器後端也將用於過濾用於返回單個物件的查詢集。

例如，在前面的示例中，以及具有id的產品4675，以下URL將返回相應的物件，或返回404response ，具體取決於給定產品實例是否滿足過濾條件：

`http://example.com/api/products/4675/?category=clothing&max_price=10.00`
### 覆蓋最初的查詢集
請注意，您可以同時使用重寫`.get_queryset()`和通用篩選，並且所有內容都將按預期工作。例如，如果Product與Usernamed 具有多對多關係`purchase`，則可能需要編寫如下所示的視圖：
```py
class PurchasedProductsList(generics.ListAPIView):
    """
    Return a list of all the products that the authenticated
    user has ever purchased, with optional filtering.
    """
    model = Product
    serializer_class = ProductSerializer
    filter_class = ProductFilter

    def get_queryset(self):
        user = self.request.user
        return user.purchase_set.all()
```
## API指南
### DjangoFilterBackend
該`django-filter`庫包含一個`DjangoFilterBackend`支持REST Framework的高度可自定義字段過濾的類。

要使用`DjangoFilterBackend`，首先安裝django-filter。然後添加django_filters到Django的INSTALLED_APPS

`pip install django-filter`
您現在應該將過濾器後端添加到您的設置中：
```py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```
或者將過濾器後端添加到單個View或ViewSet。
```py
from django_filters.rest_framework import DjangoFilterBackend

class UserListView(generics.ListAPIView):
    ...
    filter_backends = (DjangoFilterBackend,)
```
如果您只需要簡單的基於等式的過濾，則可以filter_fields在視圖或視圖集上設置一個屬性，列出您要過濾的一組字段。
```py
class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = (DjangoFilterBackend,)
    filter_fields = ('category', 'in_stock')
```
這將自動FilterSet為給定字段創建一個類，並允許您提出如下請求：

`http://example.com/api/products?category=clothing&in_stock=True`
對於更高級的過濾要求，您可以指定FilterSet視圖應該使用的類。你可以FilterSet在django-filter文檔中閱讀更多關於s的內容。還建議您閱讀DRF集成部分。

## SearchFilter
該SearchFilter級支持簡單單的查詢參數基於搜索和基於該admin界面的搜索功能。

在使用時，可瀏覽的API將包含一個SearchFilter控件：

搜索過濾器

所述SearchFilter如果視圖有一類將只應用於search_fields屬性集。該search_fields屬性應該是模型上文本類型字段的名稱列表，例如CharField或TextField。
```py
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.SearchFilter,)
    search_fields = ('username', 'email')
```
這將允許客戶端通過查詢來過濾列表中的項目，例如：

`http://example.com/api/users?search=russell`
您還可以使用查找API雙下劃線表示法對ForeignKey或ManyToManyField執行相關查找：

search_fields = ('username', 'email', 'profile__profession')
預設情況下，搜索將使用不區分大小寫的部分匹配。搜索參數可能包含多個搜索詞，它們應該是空格和/或逗號分隔的。如果使用多個搜索條件，則只有在所有提供的條件匹配的情況下，物件才會返回到列表中。

搜索行為可能受到各種字符預先限制search_fields。

- '^'開始 - 搜索。
- '='完全匹配。
- '@'全文搜索。（目前只支持Django的MySQL後端。）
- '$'正則表達式搜索。
例如：

`search_fields = ('=username', '=email')`
預設情況下，搜索參數名為'search'，但可能會被SEARCH_PARAM設置覆蓋。

有關更多詳細信息，請參閱Django文檔。

### OrderingFilter
本`OrderingFilter`類支持控制結果的排序簡單的查詢參數。

排序過濾器

預設情況下，查詢參數是命名的`'ordering'`，但這可能會被ORDERING_PARAM設置覆蓋。

例如，要通過用戶名來排序用戶：

http://example.com/api/users?ordering=username
客戶端也可以通過在字段名稱前加' - '來指定反向排序，如下所示：

http://example.com/api/users?ordering=-username
也可以指定多個訂單：

http://example.com/api/users?ordering=account,username

#### 指定可以針對哪些字段進行排序
建議您明確指定API應允許在排序過濾器中使用哪些字段。您可以通過`ordering_fields`在視圖上設置屬性來完成此操作，如下所示：
```py
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
```
這有助於防止意外的數據洩漏，例如允許用戶針對密碼哈希字段或其他敏感數據進行訂購。

如果您未`ordering_fields`在視圖上指定屬性，那麼過濾器類將預設允許用戶過濾`serializer_class`屬性指定的序列化程序中的任何可讀字段。

如果您確信視圖使用的查詢集不包含任何敏感數據，則還可以通過使用特殊值明確指定視圖應允許在任何模型字段或查詢集合上進行排序`'__all__'`。
```py
class BookingsListView(generics.ListAPIView):
    queryset = Booking.objects.all()
    serializer_class = BookingSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = '__all__'
```
#### 指定預設順序
如果`ordering`視圖上設置了屬性，則將用作預設排序。

通常情況下，您應該通過設置order_by初始查詢集來控制此操作，但ordering在視圖上使用參數允許您指定排序，然後可以將其作為上下文自動傳遞到呈現的模板。這可以自動呈現列標題，如果它們用於排序結果。
```py
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
    ordering = ('username',)
```
該`ordering`屬性可以是字符串的字符串或列表/元組。

### DjangoObjectPermissionsFilter
該軟件`DjangoObjectPermissionsFilter`旨在與django-guardian軟件包一起使用，並`'view'`添加了自定義權限。過濾器將確保查詢集僅返回用戶具有適當查看權限的物件。

如果您正在使用`DjangoObjectPermissionsFilter`，您可能還需要添加適當的物件權限類，以確保用戶只能在具有適當物件權限的實例上進行操作。做到這一點的最簡單方法是對屬性進行子類化DjangoObjectPermissions和添加'view'權限`perms_map`。

一個完整的例子使用兩者DjangoObjectPermissionsFilter，DjangoObjectPermissions可能看起來像這樣。

permissions.py：
```py
class CustomObjectPermissions(permissions.DjangoObjectPermissions):
	"""
	Similar to `DjangoObjectPermissions`, but adding 'view' permissions.
	"""
    perms_map = {
        'GET': ['%(app_label)s.view_%(model_name)s'],
        'OPTIONS': ['%(app_label)s.view_%(model_name)s'],
        'HEAD': ['%(app_label)s.view_%(model_name)s'],
        'POST': ['%(app_label)s.add_%(model_name)s'],
        'PUT': ['%(app_label)s.change_%(model_name)s'],
        'PATCH': ['%(app_label)s.change_%(model_name)s'],
        'DELETE': ['%(app_label)s.delete_%(model_name)s'],
    }
```
views.py：
```py
class EventViewSet(viewsets.ModelViewSet):
	"""
	Viewset that only lists events if user has 'view' permissions, and only
	allows operations on individual events if user has appropriate 'view', 'add',
	'change' or 'delete' permissions.
	"""
    queryset = Event.objects.all()
    serializer_class = EventSerializer
    filter_backends = (filters.DjangoObjectPermissionsFilter,)
    permission_classes = (myapp.permissions.CustomObjectPermissions,)
```
有關添加更多的信息，'view'權限模型，請參閱相關章節中的django-guardian文檔和我這篇文章。

### 自定義通用篩選
您還可以提供自己的通用過濾後端，或者編寫一個可供其他開發人員使用的可安裝應用程序。

這樣做覆蓋`BaseFilterBackend`，並覆蓋該`.filter_queryset(self, request, queryset, view)`方法。該方法應該返回一個新的，過濾的查詢集。

除了允許客戶端執行搜索和過濾外，通用過濾器後端可用於限制哪些物件應該對任何給定的請求或用戶可見。

#### 例
例如，您可能需要限制用戶只能看到他們創建的物件。
```py
class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """
    Filter that only allows users to see their own objects.
    """
    def filter_queryset(self, request, queryset, view):
        return queryset.filter(owner=request.user)
```
我們可以通過覆蓋get_queryset()視圖來實現相同的行為，但使用過濾器後端可以讓您更輕鬆地將此限制添加到多個視圖，或者將其應用於整個API。

#### 定制界面
通用過濾器也可以在可瀏覽的API中呈現接口。為此，您應該實現一個to_html()返回過濾器的呈現HTML表示形式的方法。此方法應具有以下簽名：

`to_html(self, request, queryset, view)`

該方法應該返回一個呈現的HTML字符串。

#### 分頁和模式
通過實現一個`get_schema_fields()`方法，您還可以使過濾器控件可用於REST Framework提供的模式自動生成。此方法應具有以下簽名：

`get_schema_fields(self, view)`

該方法應該返回一個coreapi.Field實例列表。

### 第三方軟件包
以下第三方包提供了額外的過濾器實現。

#### Django REST framework filters package
`django-rest-framework-filters package`封裝與一起工作DjangoFilterBackend類，並允許您輕鬆地在關係創建過濾器，或在指定字段創建多個過濾器查找類型。

#### Django REST framework full word search filter
`djangorestframework-word-filter`開發作為替代方案filters.SearchFilter，可以在文本中搜索整個單詞，或完全匹配。

#### Django URL Filter
`django-url-filter`提供了一種安全的方式來通過人性化網址過濾數據。它與DRF序列化器和字段非常相似，因為它們可以嵌套，除了它們被稱為過濾器和過濾器。這為過濾相關數據提供了簡便的方法。此外，這個庫是通用的，所以它可以用來過濾其他數據源，而不僅僅是Django QuerySet。

#### drf-url-filters
`drf-url-filters`是一個簡單的Django應用程序適用於DRF過濾器ModelViewSet的Queryset一個清楚，簡單和可配置的方式。它還支持傳入查詢參數及其值的驗證。一個漂亮的python包Voluptuous正用於傳入查詢參數的驗證。關於性感的最好的部分是你可以根據你的查詢參數要求定義你自己的驗證。