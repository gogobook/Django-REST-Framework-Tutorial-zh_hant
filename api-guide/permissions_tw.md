source: permissions.py

# Permissions

> Authentication or identification by itself is not usually sufficient to gain access to information or code.  For that, the entity requesting access must have authorization.
>
> &mdash; [Apple Developer Documentation][cite]

身份驗證或身份驗證通常不足以獲取對信息或代碼的訪問權限。為此，請求訪問的實體必須具有授權。

Together with [authentication] and [throttling], permissions determine whether a request should be granted or denied access.
與身份驗證和限制一起，權限決定是請求應該授予還是拒絕訪問。

Permission checks are always run at the very start of the view, before any other code is allowed to proceed.  Permission checks will typically use the authentication information in the `request.user` and `request.auth` properties to determine if the incoming request should be permitted.
在允許任何其他代碼繼續之前，權限檢查始終在視圖的最開始運行時。權限檢查通常使用request.user和request.auth屬性中的身份驗證信息來確定是否應允許連入請求。

Permissions are used to grant or deny access different classes of users to different parts of the API.
權限被用以決定不同的使用者對不同的api 是否可以存取。

The simplest style of permission would be to allow access to any authenticated user, and deny access to any unauthenticated user. This corresponds the `IsAuthenticated` class in REST framework.
最簡單的權限是認證的使用者可以存取，而未認證的使用者不行，在REST framework中這是`IsAuthenticated`
最簡單的權限類型是允許訪問任何經過身份驗證的用戶，並拒絕訪問任何未經身份驗證的用戶。這對應IsAuthenticated於REST框架中的類。

A slightly less strict style of permission would be to allow full access to authenticated users, but allow read-only access to unauthenticated users. This corresponds to the `IsAuthenticatedOrReadOnly` class in REST framework.
一個較不嚴格的形式是認證使用者具有全部存取權，而非認證使用者具有只讀權
稍微不那麼嚴格的權限樣式是允許對經過身份驗證的用戶進行完全訪問，但允許對未經身份驗證的用戶進行只讀訪問。這對應IsAuthenticatedOrReadOnly於REST框架中的類。

## How permissions are determined
## 權限如何決定
Permissions in REST framework are always defined as a list of permission classes.
在REST framework中 權限總是定義成permission classes的串列

Before running the main body of the view each permission in the list is checked.
在執行視圖的主體之前，串列的每一個permission被檢查。

If any permission check fails an `exceptions.PermissionDenied` or `exceptions.NotAuthenticated` exception will be raised, and the main body of the view will not run.
假如權限檢查失敗，這將造成`exceptions.PermissionDenied` or `exceptions.NotAuthenticated` 例外，並且視圖主體將不會執行。

When the permissions checks fail either a "403 Forbidden" or a "401 Unauthorized" response will be returned, according to the following rules:
同時返回"403 Forbidden" or a "401 Unauthorized" response，依據下列規則。

* The request was successfully authenticated, but permission was denied. *&mdash; An HTTP 403 Forbidden response will be returned.*
* The request was not successfully authenticated, and the highest priority authentication class *does not* use `WWW-Authenticate` headers. *&mdash; An HTTP 403 Forbidden response will be returned.*
* The request was not successfully authenticated, and the highest priority authentication class *does* use `WWW-Authenticate` headers. *&mdash; An HTTP 401 Unauthorized response, with an appropriate `WWW-Authenticate` header will be returned.*

* 請求已成功通過身份驗證，但權限已被拒絕。 - 將返回HTTP 403 Forbidden響應。
* 請求未成功通過身份驗證，並且優先級最高的身份驗證類不使用WWW-Authenticate標頭。 - 將返回HTTP 403 Forbidden響應。
* 請求未成功通過身份驗證，並且優先級最高的身份驗證類確實使用WWW-Authenticate標頭。- WWW-Authenticate將返回帶有適當標頭的HTTP 401 Unauthorized響應。

## Object level permissions
## 物件級權限

REST framework permissions also support object-level permissioning.  Object level permissions are used to determine if a user should be allowed to act on a particular object, which will typically be a model instance.
REST framework permissions也支援物件級權限，
REST框架權限還支持對象級權限。對象級權限用於確定是否應允許用戶對特定對象執行操作，該對象通常是模型實例。

Object level permissions are run by REST framework's generic views when `.get_object()` is called.
當`.get_object()`被呼叫時，由REST framework's generic views執行物件級權限
`.get_object()`調用時，REST框架的通用視圖將運行對象級權限。與視圖級別權限一樣，exceptions.PermissionDenied如果不允許用戶對給定對象執行操作，則會引發異常。

As with view level permissions, an `exceptions.PermissionDenied` exception will be raised if the user is not allowed to act on the given object.

If you're writing your own views and want to enforce object level permissions, or if you override the `get_object` method on a generic view, then you'll need to explicitly call the `.check_object_permissions(request, obj)` method on the view at the point at which you've retrieved the object.

This will either raise a `PermissionDenied` or `NotAuthenticated` exception, or simply return if the view has the appropriate permissions.

For example:

    def get_object(self):
        obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
        self.check_object_permissions(self.request, obj)
        return obj

#### Limitations of object level permissions
#### 物件權限的限制
For performance reasons the generic views will not automatically apply object level permissions to each instance in a queryset when returning a list of objects.
出於性能原因，在返回對象列表時，通用視圖不會自動將對象級權限應用於查詢集中的每個實例。

Often when you're using object level permissions you'll also want to [filter the queryset][filtering] appropriately, to ensure that users only have visibility onto instances that they are permitted to view.
通常，當您使用對象級權限時，您還需要適當地過濾查詢集，以確保用戶只能看到允許他們查看的實例。

## Setting the permission policy

The default permission policy may be set globally, using the `DEFAULT_PERMISSION_CLASSES` setting.  For example.

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': (
            'rest_framework.permissions.IsAuthenticated',
        )
    }

If not specified, this setting defaults to allowing unrestricted access:

    'DEFAULT_PERMISSION_CLASSES': (
       'rest_framework.permissions.AllowAny',
    )

You can also set the authentication policy on a per-view, or per-viewset basis,
using the `APIView` class-based views.
您還可以使用APIView基於類的視圖在每個視圖或每個視圖集的基礎上設置身份驗證策略。

    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        permission_classes = (IsAuthenticated,)

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

Or, if you're using the `@api_view` decorator with function based views.
或者，如果您正在使用@api_view具有基於功能的視圖的裝飾器。

    from rest_framework.decorators import api_view, permission_classes
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response

    @api_view(['GET'])
    @permission_classes((IsAuthenticated, ))
    def example_view(request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

__Note:__ when you set new permission classes through class attribute or decorators you're telling the view to ignore the default list set over the __settings.py__ file.
注意：當您通過類屬性或裝飾器設置新的權限類時，您正在告訴視圖忽略settings.py文件中的默認列表集。

Provided they inherit from `rest_framework.permissions.BasePermission`, permissions can be composed using standard Python bitwise operators. For example, `IsAuthenticatedOrReadOnly` could be written:
如果它們繼承自`rest_framework.permissions.BasePermission`，則可以使用標準Python按位運算符組合權限。例如，IsAuthenticatedOrReadOnly可以寫成：

    from rest_framework.permissions import BasePermission, IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ReadOnly(BasePermission):
        def has_permission(self, request, view):
            return request.method in SAFE_METHODS

    class ExampleView(APIView):
        permission_classes = (IsAuthenticated|ReadOnly)

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

__Note:__ it only supports & -and- and | -or-.

---

# API Reference

## AllowAny

The `AllowAny` permission class will allow unrestricted access, **regardless of if the request was authenticated or unauthenticated**.

This permission is not strictly required, since you can achieve the same result by using an empty list or tuple for the permissions setting, but you may find it useful to specify this class because it makes the intention explicit.

## IsAuthenticated

The `IsAuthenticated` permission class will deny permission to any unauthenticated user, and allow permission otherwise.

This permission is suitable if you want your API to only be accessible to registered users.

## IsAdminUser

The `IsAdminUser` permission class will deny permission to any user, unless `user.is_staff` is `True` in which case permission will be allowed.

This permission is suitable if you want your API to only be accessible to a subset of trusted administrators.

## IsAuthenticatedOrReadOnly

The `IsAuthenticatedOrReadOnly` will allow authenticated users to perform any request.  Requests for unauthorised users will only be permitted if the request method is one of the "safe" methods; `GET`, `HEAD` or `OPTIONS`.

This permission is suitable if you want to your API to allow read permissions to anonymous users, and only allow write permissions to authenticated users.

## DjangoModelPermissions

This permission class ties into Django's standard `django.contrib.auth` [model permissions][contribauth].  This permission must only be applied to views that have a `.queryset` property set. Authorization will only be granted if the user *is authenticated* and has the *relevant model permissions* assigned.
此權限類與Django的標準django.contrib.auth 模型權限相關聯。此權限只能應用於具有.queryset屬性集的視圖。只有在用戶通過身份驗證且分配了相關的模型權限後，才會授予授權。

* `POST` requests require the user to have the `add` permission on the model.
* `PUT` and `PATCH` requests require the user to have the `change` permission on the model.
* `DELETE` requests require the user to have the `delete` permission on the model.

* POST請求要求用戶擁有add模型的權限。
* PUT和PATCH請求要求用戶擁有change模型的權限。
* DELETE請求要求用戶擁有delete模型的權限。

The default behaviour can also be overridden to support custom model permissions.  For example, you might want to include a `view` model permission for `GET` requests.
還可以重寫默認行為以支持自定義模型權限。例如，您可能希望包含請求的view模型權限GET。

To use custom model permissions, override `DjangoModelPermissions` and set the `.perms_map` property.  Refer to the source code for details.
要使用自定義模型權限，請覆蓋`DjangoModelPermissions`並設置.perms_map屬性。有關詳細信息，請參閱源代碼。


#### Using with views that do not include a `queryset` attribute.

If you're using this permission with a view that uses an overridden `get_queryset()` method there may not be a `queryset` attribute on the view. In this case we suggest also marking the view with a sentinel queryset, so that this class can determine the required permissions. For example:
如果您對使用重寫`get_queryset()`方法的視圖使用此權限，則視圖上可能沒有queryset屬性。在這種情況下，我們建議使用sentinel查詢集標記視圖，以便此類可以確定所需的權限。例如：

    queryset = User.objects.none()  # Required for DjangoModelPermissions

## DjangoModelPermissionsOrAnonReadOnly

Similar to `DjangoModelPermissions`, but also allows unauthenticated users to have read-only access to the API.

## DjangoObjectPermissions

This permission class ties into Django's standard [object permissions framework][objectpermissions] that allows per-object permissions on models.  In order to use this permission class, you'll also need to add a permission backend that supports object-level permissions, such as [django-guardian][guardian].
此權限類與Django的標準對象權限框架相關聯，該框架允許模型上的每個對象權限。要使用此權限類，您還需要添加支持對象級權限的權限後端，例如django-guardian。

As with `DjangoModelPermissions`, this permission must only be applied to views that have a `.queryset` property or `.get_queryset()` method. Authorization will only be granted if the user *is authenticated* and has the *relevant per-object permissions* and *relevant model permissions* assigned.
與DjangoModelPermissions此一樣，此權限只能應用於具有.queryset屬性或.get_queryset()方法的視圖。只有在用戶通過身份驗證且具有相關的每個對象權限和相關的模型權限時，才會授予授權。

* `POST` requests require the user to have the `add` permission on the model instance.
* `PUT` and `PATCH` requests require the user to have the `change` permission on the model instance.
* `DELETE` requests require the user to have the `delete` permission on the model instance.

* POST請求要求用戶擁有add模型實例的權限。
* PUT和PATCH請求要求用戶擁有change模型實例的權限。
* DELETE請求要求用戶擁有delete模型實例的權限。

Note that `DjangoObjectPermissions` **does not** require the `django-guardian` package, and should support other object-level backends equally well.
請注意，DjangoObjectPermissions 不要求django-guardian包，並且應該支持其他對象級後端同樣出色。

As with `DjangoModelPermissions` you can use custom model permissions by overriding `DjangoObjectPermissions` and setting the `.perms_map` property.  Refer to the source code for details.
與`DjangoModelPermissions`您一樣，您可以通過覆蓋`DjangoObjectPermissions`和設置`.perms_map`屬性來使用自定義模型權限。有關詳細信息，請參閱源代碼。
---

**Note**: If you need object level `view` permissions for `GET`, `HEAD` and `OPTIONS` requests and are using django-guardian for your object-level permissions backend, you'll want to consider using the `DjangoObjectPermissionsFilter` class provided by the [`djangorestframework-guardian` package][django-rest-framework-guardian]. It ensures that list endpoints only return results including objects for which the user has appropriate view permissions.
**注意**：如果你需要的對象級別view的權限GET，HEAD並OPTIONS請求和正在使用Django的監護人為對象級權限的後端，你要考慮使用`DjangoObjectPermissionsFilter`由提供的類`djangorestframework-guardian`包。它確保列表端點僅返回包括用戶具有適當查看權限的對象的結果。
---

# Custom permissions

To implement a custom permission, override `BasePermission` and implement either, or both, of the following methods:
要實現自定義權限，請覆蓋`BasePermission`並實現以下方法之一或兩者：

* `.has_permission(self, request, view)`
* `.has_object_permission(self, request, view, obj)`

The methods should return `True` if the request should be granted access, and `False` otherwise.
True如果應該授予請求訪問權限，則應返回方法，False否則返回。

If you need to test if a request is a read operation or a write operation, you should check the request method against the constant `SAFE_METHODS`, which is a tuple containing `'GET'`, `'OPTIONS'` and `'HEAD'`.  For example:
如果您需要測試請求是讀取操作還是寫入操作，則應該針對常量檢查請求方法，該常量SAFE_METHODS是包含的元組'GET'，'OPTIONS'和'HEAD'。例如：

    if request.method in permissions.SAFE_METHODS:
        # Check permissions for read-only request
    else:
        # Check permissions for write request

---

**Note**: The instance-level `has_object_permission` method will only be called if the view-level `has_permission` checks have already passed. Also note that in order for the instance-level checks to run, the view code should explicitly call `.check_object_permissions(request, obj)`. If you are using the generic views then this will be handled for you by default. (Function-based views will need to check object permissions explicitly, raising `PermissionDenied` on failure.)
**注意**：`has_object_permission`只有在`has_permission`已經通過視圖級別檢查時才會調用實例級方法。另請注意，為了運行實例級檢查，視圖代碼應顯式調用`.check_object_permissions(request, obj)`。如果您使用的是通用視圖，則默認情況下將為您處理。（基於函數的視圖需要顯式檢查對象權限，否則會PermissionDenied導致失敗。）
---

Custom permissions will raise a `PermissionDenied` exception if the test fails. To change the error message associated with the exception, implement a `message` attribute directly on your custom permission. Otherwise the `default_detail` attribute from `PermissionDenied` will be used.
`PermissionDenied`如果測試失敗，自定義權限將引發異常。要更改與異常關聯的錯誤消息，請message直接在自定義權限上實現屬性。否則將使用`default_detail`屬性`from PermissionDenied`。

    from rest_framework import permissions

    class CustomerAccessPermission(permissions.BasePermission):
        message = 'Adding customers not allowed.'

        def has_permission(self, request, view):
             ...

## Examples

The following is an example of a permission class that checks the incoming request's IP address against a blacklist, and denies the request if the IP has been blacklisted.
以下是根據黑名單檢查傳入請求的IP地址的權限類的示例，如果IP已被列入黑名單，則拒絕該請求。

    from rest_framework import permissions

    class BlacklistPermission(permissions.BasePermission):
        """
        Global permission check for blacklisted IPs.
        """

        def has_permission(self, request, view):
            ip_addr = request.META['REMOTE_ADDR']
            blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
            return not blacklisted

As well as global permissions, that are run against all incoming requests, you can also create object-level permissions, that are only run against operations that affect a particular object instance.  For example:
除了針對所有傳入請求運行的全局權限之外，您還可以創建對象級權限，這些權限僅針對影響特定對象實例的操作運行。例如：

    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        Object-level permission to only allow owners of an object to edit it.
        Assumes the model instance has an `owner` attribute.
        """

        def has_object_permission(self, request, view, obj):
            # Read permissions are allowed to any request,
            # so we'll always allow GET, HEAD or OPTIONS requests.
            if request.method in permissions.SAFE_METHODS:
                return True

            # Instance must have an attribute named `owner`.
            return obj.owner == request.user

Note that the generic views will check the appropriate object level permissions, but if you're writing your own custom views, you'll need to make sure you check the object level permission checks yourself.  You can do so by calling `self.check_object_permissions(request, obj)` from the view once you have the object instance.  This call will raise an appropriate `APIException` if any object-level permission checks fail, and will otherwise simply return.
請注意，通用視圖將檢查相應的對象級別權限，但如果您正在編寫自己的自定義視圖，則需要確保自己檢查對象級別權限檢查。您可以通過self.check_object_permissions(request, obj)在擁有對象實例後從視圖中調用來完成此操作。APIException如果任何對象級權限檢查失敗，此調用將引發適當的調用，否則將返回。

Also note that the generic views will only check the object-level permissions for views that retrieve a single model instance.  If you require object-level filtering of list views, you'll need to filter the queryset separately.  See the [filtering documentation][filtering] for more details.
另請注意，通用視圖僅檢查檢索單個模型實例的視圖的對象級權限。如果需要對列表視圖進行對象級別過濾，則需要單獨過濾查詢集。有關詳細信息，請參閱過濾文檔。

---

# Third party packages

The following third party packages are also available.

## Composed Permissions

The [Composed Permissions][composed-permissions] package provides a simple way to define complex and multi-depth (with logic operators) permission objects, using small and reusable components.

## REST Condition

The [REST Condition][rest-condition] package is another extension for building complex permissions in a simple and convenient way.  The extension allows you to combine permissions with logical operators.

## DRY Rest Permissions

The [DRY Rest Permissions][dry-rest-permissions] package provides the ability to define different permissions for individual default and custom actions. This package is made for apps with permissions that are derived from relationships defined in the app's data model. It also supports permission checks being returned to a client app through the API's serializer. Additionally it supports adding permissions to the default and custom list actions to restrict the data they retrieve per user.

## Django Rest Framework Roles

The [Django Rest Framework Roles][django-rest-framework-roles] package makes it easier to parameterize your API over multiple types of users.

## Django Rest Framework API Key

The [Django Rest Framework API Key][django-rest-framework-api-key] package allows you to ensure that every request made to the server requires an API key header. You can generate one from the django admin interface.

## Django Rest Framework Role Filters

The [Django Rest Framework Role Filters][django-rest-framework-role-filters] package provides simple filtering over multiple types of roles.

[cite]: https://developer.apple.com/library/mac/#documentation/security/Conceptual/AuthenticationAndAuthorizationGuide/Authorization/Authorization.html
[authentication]: authentication.md
[throttling]: throttling.md
[filtering]: filtering.md
[contribauth]: https://docs.djangoproject.com/en/stable/topics/auth/customizing/#custom-permissions
[objectpermissions]: https://docs.djangoproject.com/en/stable/topics/auth/customizing/#handling-object-permissions
[guardian]: https://github.com/lukaszb/django-guardian
[filtering]: filtering.md
[composed-permissions]: https://github.com/niwibe/djangorestframework-composed-permissions
[rest-condition]: https://github.com/caxap/rest_condition
[dry-rest-permissions]: https://github.com/Helioscene/dry-rest-permissions
[django-rest-framework-roles]: https://github.com/computer-lab/django-rest-framework-roles
[django-rest-framework-api-key]: https://github.com/manosim/django-rest-framework-api-key
[django-rest-framework-role-filters]: https://github.com/allisson/django-rest-framework-role-filters
[django-rest-framework-guardian]: https://github.com/rpkilby/django-rest-framework-guardian
