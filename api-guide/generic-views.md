#通用視圖 (即generic.GenericAPIView)
> Django的通用視圖......被開發為常見用法模式的快捷方式......它們採用視圖開發中的某些常見習語和模式並對其進行抽象，以便您可以快速編寫數據的常用視圖，而無需重複自己。

>- Django文檔

基於類的視圖的一個主要好處是它們允許您組合可重用行為的方式。REST framework通過提供許多預先構建的視圖來提供常用模式來利用這一點。

REST framework提供的通用視圖允許您快速構建與您的數據庫模型緊密相關的API視圖。

如果通用視圖不適合您的API需求，您可以直接使用常規APIView類，或者重用通用視圖使用的mixins和基類來組成您自己的可重用通用視圖集。

例子
通常，在使用通用視圖時，您將覆蓋視圖，並設置多個類屬性。
```py
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)
```
對於更複雜的情況，您可能還希望覆蓋視圖類上的各種方法。例如。
```py
class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)

    def list(self, request):
        # Note the use of `get_queryset()` instead of `self.queryset`
        queryset = self.get_queryset()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)
```
對於非常簡單的情況，您可能希望使用該.as_view()方法傳遞任何類屬性。例如，您的URLconf可能包含以下條目：

`url(r'^/users/', ListCreateAPIView.as_view(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')`
## API參考
### GenericAPIView
此類擴展了REST framework的APIView類，為標準列表和詳細信息視圖添加了常用的行為。

提供的每個具體通用視圖是通過GenericAPIView與一個或多個mixin類組合而構建的。

#### 屬性
基本設置：

以下屬性控制基本視圖行為。

- queryset - 應該用於從此視圖返回對象的查詢集。通常，您必須設置此屬性，或覆蓋該get_queryset()方法。如果要覆蓋視圖方法，則必須調用get_queryset()而不是直接訪問此屬性，因為queryset將進行一次評估，並且將為所有後續請求緩存這些結果。
- serializer_class - 應該用於驗證和反序列化輸入以及序列化輸出的序列化程序類。通常，您必須設置此屬性，或覆蓋該get_serializer_class()方法。
- lookup_field - 應用於執行單個模型實例的對象查找的模型字段。默認為'pk'。請注意，使用超鏈接的API時，您需要確保雙方的API意見和串行類設置查找字段，如果你需要使用一個自定義值。
- lookup_url_kwarg - 應該用於對象查找的URL關鍵字參數。URL conf應包含與此值對應的關鍵字參數。如果未設置，則默認使用相同的值lookup_field。
分頁：

與列表視圖一起使用時，以下屬性用於控制分頁。

- pagination_class - 分頁列表結果時應使用的分頁類。默認為與DEFAULT_PAGINATION_CLASS設置相同的值，即'rest_framework.pagination.PageNumberPagination'。設置pagination_class=None將禁用此視圖上的分頁。
過濾：

- filter_backends - 應該用於過濾查詢集的過濾器後端類列表。默認值與DEFAULT_FILTER_BACKENDS設置相同。
方法
基本方法：

`get_queryset(self)`
返回應該用於列表視圖的查詢集，該查詢集應該用作詳細視圖中查找的基礎。默認返回queryset屬性指定的查詢集。

應始終使用此方法而不是self.queryset直接訪問，因為self.queryset只進行一次評估，並為所有後續請求緩存這些結果。

可以重寫以提供動態行為，例如返回查詢集，該查詢集特定於發出請求的用戶。

例如：
```py
def get_queryset(self):
    user = self.request.user
    return user.accounts.all()
```
`get_object(self)`

返回應該用於詳細視圖的對象實例。默認使用lookup_field參數來過濾基本查詢集。

可以重寫以提供更複雜的行為，例如基於多個URL kwarg的對象查找。

例如：
```py
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)
    return obj
```
請注意，如果您的API不包含任何對象級別權限，您可以選擇性地排除self.check_object_permissions，並簡單地從get_object_or_404查找中返回該對象。

filter_queryset(self, queryset)
給定一個查詢集，使用正在使用的任何過濾後端過濾它，返回一個新的查詢集。

例如：

def filter_queryset(self, queryset):
    filter_backends = (CategoryFilter,)

    if 'geo_route' in self.request.query_params:
        filter_backends = (GeoRouteFilter, CategoryFilter)
    elif 'geo_point' in self.request.query_params:
        filter_backends = (GeoPointFilter, CategoryFilter)

    for backend in list(filter_backends):
        queryset = backend().filter_queryset(self.request, queryset, view=self)

    return queryset
get_serializer_class(self)
返回應該用於序列化程序的類。默認返回serializer_class屬性。

可以重寫以提供動態行為，例如使用不同的序列化程序進行讀寫操作，或者為不同類型的用戶提供不同的序列化程序。

例如：

def get_serializer_class(self):
    if self.request.user.is_staff:
        return FullAccountSerializer
    return BasicAccountSerializer
保存和刪除掛鉤：

mixin類提供了以下方法，並提供了對象保存或刪除行為的輕鬆覆蓋。

perform_create(self, serializer)- CreateModelMixin保存新對象實例時調用。
perform_update(self, serializer)- UpdateModelMixin保存現有對象實例時調用。
perform_destroy(self, instance)- DestroyModelMixin刪除對象實例時調用。
這些掛鉤對於設置請求中隱含的屬性特別有用，但不是請求數據的一部分。例如，您可以根據請求用戶或基於URL關鍵字參數在對像上設置屬性。

def perform_create(self, serializer):
    serializer.save(user=self.request.user)
這些覆蓋點對於添加在保存對象之前或之後發生的行為（例如通過電子郵件發送確認或記錄更新）也特別有用。

def perform_update(self, serializer):
    instance = serializer.save()
    send_email_confirmation(user=self.request.user, modified=instance)
你也可以使用這些鉤子來提供額外的驗證，通過提高ValidationError()。如果您需要在數據庫保存點應用某些驗證邏輯，這可能很有用。例如：

def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
注意：這些方法取代舊式的2.x版pre_save，post_save，pre_delete和post_delete方法，這將不再可用。

其他方法：

您通常不需要覆蓋以下方法，但如果您正在使用編寫自定義視圖，則可能需要調用它們GenericAPIView。

- get_serializer_context(self) - 返回包含應提供給序列化程序的任何額外上下文的字典。默認為包括'request'，'view'和'format'鑰匙。
- get_serializer(self, instance=None, data=None, many=False, partial=False) - 返回一個序列化程序實例。
- get_paginated_response(self, data)- 返回分頁樣式Response對象。
- paginate_queryset(self, queryset)- 如果需要，可以分頁查詢集，返回頁面對象，或者None如果沒有為此視圖配置分頁。
- filter_queryset(self, queryset) - 給定一個查詢集，使用正在使用的過濾後端進行過濾，返回一個新的查詢集。

## 混入
mixin類提供用於提供基本視圖行為的操作。請注意，mixin類提供了操作方法，而不是直接定義處理程序方法，例如.get()和.post()。這允許更靈活的行為組合。

mixin類可以從中導入rest_framework.mixins。

ListModelMixin
提供一種.list(request, *args, **kwargs)實現列出查詢集的方法。

如果填充了查詢集，則返迴200 OK響應，並將查詢集的序列化表示形式作為響應的主體。可選地，可以對響應數據進行分頁。

CreateModelMixin
提供.create(request, *args, **kwargs)實現創建和保存新模型實例的方法。

如果創建了一個對象，則返回一個201 Created響應，該對象的序列化表示形式作為響應的主體。如果表示包含名為的鍵url，則Location響應的標題將填充該值。

如果為創建對象而提供的請求數據無效，400 Bad Request則將返迴響應，並將錯誤詳細信息作為響應的主體。

RetrieveModelMixin
提供一種.retrieve(request, *args, **kwargs)方法，該方法實現在響應中返回現有模型實例。

如果可以檢索對象，則返迴200 OK響應，並將對象的序列化表示作為響應的主體。否則它會返回一個404 Not Found。

UpdateModelMixin
提供.update(request, *args, **kwargs)實現更新和保存現有模型實例的方法。

還提供了.partial_update(request, *args, **kwargs)一種類似於該update方法的方法，除了更新的所有字段都是可選的。這允許支持HTTP PATCH請求。

如果更新了對象，則返迴200 OK響應，並將對象的序列化表示作為響應的主體。

如果為更新對象而提供的請求數據無效，400 Bad Request則將返迴響應，並將錯誤詳細信息作為響應的主體。

DestroyModelMixin
提供一種.destroy(request, *args, **kwargs)實現刪除現有模型實例的方法。

如果刪除了一個對象，則返回一個204 No Content響應，否則返回一個404 Not Found。

具體視圖類
以下類是具體的通用視圖。如果您使用的是通用視圖，這通常是您將要工作的級別，除非您需要大量自定義的行為。

可以從中導入視圖類rest_framework.generics。

CreateAPIView
用於僅創建端點。

提供post方法處理程序。

擴展：GenericAPIView，CreateModelMixin

ListAPIView
用於只讀端點以表示模型實例的集合。

提供get方法處理程序。

擴展：GenericAPIView，ListModelMixin

RetrieveAPIView
用於表示單個模型實例的只讀端點。

提供get方法處理程序。

擴展：GenericAPIView，RetrieveModelMixin

DestroyAPIView
用於單個模型實例的僅刪除端點。

提供delete方法處理程序。

擴展：GenericAPIView，DestroyModelMixin

UpdateAPIView
用於單個模型實例的僅更新端點。

提供put和patch方法處理程序。

擴展：GenericAPIView，UpdateModelMixin

ListCreateAPIView
用於讀寫端點以表示模型實例的集合。

提供get和post方法處理程序。

擴展：GenericAPIView，ListModelMixin，CreateModelMixin

RetrieveUpdateAPIView
用於讀取或更新端點以表示單個模型實例。

提供get，put並且patch方法處理。

擴展：GenericAPIView，RetrieveModelMixin，UpdateModelMixin

RetrieveDestroyAPIView
用於讀取或刪除端點以表示單個模型實例。

提供get和delete方法處理程序。

擴展：GenericAPIView，RetrieveModelMixin，DestroyModelMixin

RetrieveUpdateDestroyAPIView
用於讀寫 - 刪除端點以表示單個模型實例。

提供get，put，patch和delete方法處理。

擴展：GenericAPIView，RetrieveModelMixin，UpdateModelMixin，DestroyModelMixin

自定義通用視圖
通常，您會希望使用現有的通用視圖，但使用一些略微自定義的行為。如果您發現自己在多個位置重複使用某些自定義行為，則可能需要將該行為重構為公共類，然後您可以根據需要將其應用於任何視圖或視圖集。

創建自定義mixins
例如，如果您需要根據URL conf中的多個字段查找對象，則可以創建如下所示的mixin類：

class MultipleFieldLookupMixin(object):
    """
    Apply this mixin to any view or viewset to get multiple field filtering
    based on a `lookup_fields` attribute, instead of the default single field filtering.
    """
    def get_object(self):
        queryset = self.get_queryset()             # Get the base queryset
        queryset = self.filter_queryset(queryset)  # Apply any filter backends
        filter = {}
        for field in self.lookup_fields:
            if self.kwargs[field]: # Ignore empty fields.
                filter[field] = self.kwargs[field]
        obj = get_object_or_404(queryset, **filter)  # Lookup the object
        self.check_object_permissions(self.request, obj)
        return obj
然後，只要您需要應用自定義行為，就可以將此mixin簡單地應用於視圖或視圖集。

class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ('account', 'username')
如果您需要使用自定義行為，則使用自定義mixins是一個不錯的選擇。

創建自定義基類
如果您在多個視圖中使用mixin，則可以更進一步，創建自己的一組基本視圖，然後可以在整個項目中使用。例如：

class BaseRetrieveView(MultipleFieldLookupMixin,
                       generics.RetrieveAPIView):
    pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
                                    generics.RetrieveUpdateDestroyAPIView):
    pass
如果您的自定義行為始終需要在整個項目中的大量視圖中重複，那麼使用自定義基類是一個不錯的選擇。

PUT為創造
在3.0版之前，REST frameworkmixin被PUT視為更新或創建操作，具體取決於對像是否已存在。

允許PUT作為創建操作是有問題的，因為它必然暴露有關對象的存在或不存在的信息。透明地允許重新創建先前刪除的實例並不一定是比簡單地返迴404響應更好的默認行為。

兩種樣式“ PUTas 404”和“ PUTas create”在不同情況下都可以有效，但從版本3.0開始，我們現在使用404行為作為默認值，因為它更簡單，更明顯。

如果需要通用PUT，為創建行為，你可能要包括像這個AllowPUTAsCreateMixin類的混入你的意見。

第三方包
以下第三方包提供了其他通用視圖實現。

Django REST framework批量
在Django的REST的架構，散包實現通用視圖混入以及一些普通混凝土通用視圖允許通過API請求應用批量操作。

Django休息多個模型
Django Rest Multiple Models提供了一個通用視圖（和mixin），用於通過單個API請求發送多個序列化模型和/或查詢集。