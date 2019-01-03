> [官方原文連結](http://www.django-rest-framework.org/api-guide/serializers/)

## Serializers

序列化器允許將諸如查詢集和模型例項之類的複雜資料轉換為原生 Python 資料型別，然後可以將它們輕鬆地呈現為 `JSON`，`XML` 或其他內容型別。序列化器還提供反序列化，在首次驗證傳入資料之後，可以將解析的資料轉換回複雜型別。

REST framework 中的序列化類與 Django 的 `Form` 和 `ModelForm` 類非常相似。我們提供了一個 `Serializer` 類，它提供了一種強大的通用方法來控制響應的輸出，以及一個 `ModelSerializer` 類，它為建立處理模型例項和查詢集的序列化提供了有效的快捷方式。

### 1. 宣告序列化類

首先建立一個簡單的物件用於示例：

``` python
from datetime import datetime

class Comment(object):
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()

comment = Comment(email='leila@example.com', content='foo bar')
```

宣告一個序列化類，使用它來序列化和反序列化與 `Comment` 物件相對應的資料。

宣告一個序列化類看起來非常類似於宣告一個表單：

``` python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

### 2. 序列化物件

現在可以使用 `CommentSerializer` 來序列化評論或評論列表。同樣，使用 `Serializer` 類看起來很像使用 `Form` 類。

``` python
serializer = CommentSerializer(comment)
serializer.data
# {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}
```

此時已經將模型例項轉換為 Python 原生資料型別。為了完成序列化過程，將資料渲染為 `json`。

``` python
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'
```


### 3. 資料反序列化為物件

反序列化是相似的。首先我們將一個流解析為 Python 原生資料型別...

``` python
from django.utils.six import BytesIO
from rest_framework.parsers import JSONParser

stream = BytesIO(json)
data = JSONParser().parse(stream)
```

...然後我們將這些原生資料型別恢復成通過驗證的資料字典。

``` python
serializer = CommentSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}
```




### 4. 儲存例項

如果希望能夠基於驗證的資料返回完整的物件例項，則需要實現 `.create()` 和 `.update()` 方法中的一個或兩個。例如：

``` python
class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()

    def create(self, validated_data):
        return Comment(**validated_data)

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        return instance
```

如果物件例項與 Django 模型相對應，還需要確保這些方法將物件儲存到資料庫。如果 `Comment` 是一個 Django 模型，這些方法可能如下所示：

``` python
    def create(self, validated_data):
        return Comment.objects.create(**validated_data)

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        instance.save()
        return instance
```

現在，當反序列化資料時，我們可以呼叫 `.save()` 根據驗證的資料返回一個物件例項。

``` python
comment = serializer.save()
```

呼叫 `.save()` 將建立一個新例項或更新現有例項，具體取決於在例項化序列化類時是否傳遞了現有例項：

``` python
# .save() will create a new instance.
serializer = CommentSerializer(data=data)

# .save() will update the existing `comment` instance.
serializer = CommentSerializer(comment, data=data)
```

`.create()` 和 `.update()` 方法都是可選的。您可以都不實現，或者實現其中的一個或兩個，具體取決於你的序列化類的用例。

#### 將附加屬性傳遞給 `.save()`

有時你會希望你的檢視程式碼能夠在儲存例項的時候注入額外的資料。這些附加資料可能包含當前使用者，當前時間或其他任何不屬於請求資料的資訊。

``` python
serializer.save(owner=request.user)
```

呼叫 `.create()` 或 `.update()` 時，任何其他關鍵字引數都將包含在 `validated_data` 引數中。

#### 直接覆蓋 `.save()`。

在某些情況下，`.create()` 和 `.update()` 方法名稱可能沒有意義。例如，在 “聯絡人表單” 中，我們可能不會建立新例項，而是傳送電子郵件或其他訊息。

在這些情況下，可以選擇直接覆蓋 `.save()`，因為它更具可讀性和有意義性。

舉個栗子：

``` python
class ContactForm(serializers.Serializer):
    email = serializers.EmailField()
    message = serializers.CharField()

    def save(self):
        email = self.validated_data['email']
        message = self.validated_data['message']
        send_email(from=email, message=message)
```

請注意，在上面的情況下，必須直接訪問 `serializer .validated_data` 屬性。

### 5. 驗證

在資料反序列化時，你總是需要在嘗試訪問驗證資料之前呼叫 `is_valid()`，或者儲存物件例項。如果發生任何驗證錯誤，那麼 `.errors` 屬性將包含一個代表錯誤訊息的字典。例如：

``` python
serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}
```

字典中的每個鍵都是欄位名稱，值是與該欄位相對應的錯誤訊息（字串列表）。`non_field_errors` 鍵也可能存在，並會列出任何常規驗證錯誤。可以使用 `NON_FIELD_ERRORS_KEY` （在 settings 檔案中設定）來定製 `non_field_errors` 關鍵字的名稱。

反序列化 item 列表時，錯誤將作為代表每個反序列化 item 的字典列表返回。

#### 資料驗證時丟擲異常

`.is_valid()` 方法帶有一個可選的 `raise_exception` 標誌，如果存在驗證錯誤，將導致它引發 `serializers.ValidationError` 異常。


這些異常由 REST framework 提供的預設異常處理程式自動處理，並且預設情況下將返回 `HTTP 400 Bad Request`。

``` python
# Return a 400 response if the data was invalid.
serializer.is_valid(raise_exception=True)
```

#### 欄位級驗證

你可以通過向 `Serializer` 子類新增 `.validate_<field_name>` 方法來指定自定義欄位級驗證。這些與 Django 表單上的 `.clean_<field_name>` 方法類似。

這些方法只有一個引數，就是需要驗證的欄位值。

您的 `validate_<field_name>` 方法應返回驗證值或引發 `serializers.ValidationError`。

例如：

``` python
from rest_framework import serializers

class BlogPostSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()

    def validate_title(self, value):
        """
        Check that the blog post is about Django.
        """
        if 'django' not in value.lower():
            raise serializers.ValidationError("Blog post is not about Django")
        return value
```

> 注意：如果你的序列化程式中宣告的 `<field_name>` 引數為 `required = False` ，那麼如果未包含該欄位，則不會執行此驗證步驟。

#### 物件級驗證

如果要對多個欄位進行其他的驗證，請將一個名為 `.validate()` 的方法新增到您的 `Serializer` 子類中。這個方法只有一個引數，它是一個欄位值（`field`-`value`）的字典。如果有必要，它應該引發一個 `ValidationError`，或者只是返回驗證的值。例如：

``` python
from rest_framework import serializers

class EventSerializer(serializers.Serializer):
    description = serializers.CharField(max_length=100)
    start = serializers.DateTimeField()
    finish = serializers.DateTimeField()

    def validate(self, data):
        """
        Check that the start is before the stop.
        """
        if data['start'] > data['finish']:
            raise serializers.ValidationError("finish must occur after start")
        return data
```


#### 驗證器

序列化器上的各個欄位可以包含驗證器，方法是在欄位例項上宣告它們，例如：

``` python
def multiple_of_ten(value):
    if value % 10 != 0:
        raise serializers.ValidationError('Not a multiple of ten')

class GameRecord(serializers.Serializer):
    score = IntegerField(validators=[multiple_of_ten])
    ...
```

序列化類還可以包含應用於整個欄位資料集的可重用驗證器。這些驗證器是通過在內部的 `Meta` 類中宣告它們來包含的，如下所示：

``` python
class EventSerializer(serializers.Serializer):
    name = serializers.CharField()
    room_number = serializers.IntegerField(choices=[101, 102, 103, 201])
    date = serializers.DateField()

    class Meta:
        # Each room only has one event per day.
        validators = UniqueTogetherValidator(
            queryset=Event.objects.all(),
            fields=['room_number', 'date']
        )
```

> 看不懂没關係哦，更多关於驗證的内容，以后還会说到。


### 6. 訪問例項和初始資料

將初始物件或查詢集傳遞給序列化類例項時，該物件將作為 `.instance` 提供。如果沒有傳遞初始物件，則 `.instance` 屬性將為 `None`。

將資料傳遞給序列化類例項時，未修改的資料將作為 `.initial_data` 提供。如果 data 關鍵字引數未被傳遞，那麼 `.initial_data` 屬性將不存在。


### 7. 部分更新

預設情況下，序列化程式必須為所有必填欄位傳遞值，否則會引發驗證錯誤。您可以使用 `partial` 引數以允許部分更新。

``` python
# Update `comment` with partial data
serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)
```


### 8. 處理巢狀物件

前面的例子適用於處理只具有簡單資料型別的物件，但有時還需要能夠表示更複雜的物件，其中物件的某些屬性可能不是簡單的資料型別，如字串，日期或整數。

`Serializer` 類本身就是一種 `Field`，可以用來表示一個物件型別巢狀在另一個物件型別中的關係。

``` python
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

如果巢狀物件可以是 `None` 值，則應將 `required = False` 標誌傳遞給巢狀的序列化類。

``` python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)  # May be an anonymous user.
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

同樣，如果巢狀物件是一個列表，則應將 `many = True` 標誌傳遞給巢狀的序列化類。

``` python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)
    edits = EditItemSerializer(many=True)  # A nested list of 'edit' items.
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

### 9. 可寫巢狀表示

在處理支援反序列化資料的巢狀表示時，巢狀物件的任何錯誤都將巢狀在巢狀物件的欄位名稱下。

``` python
serializer = CommentSerializer(data={'user': {'email': 'foobar', 'username': 'doe'}, 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'user': {'email': [u'Enter a valid e-mail address.']}, 'created': [u'This field is required.']}
```

同樣，`.validated_data` 屬性將包含巢狀的資料結構。


#### 為巢狀表示書寫 `.create()` 方法

如果你支援可寫巢狀表示，則需要編寫處理儲存多個物件的 `.create()` 或 `.update()` 方法。

以下示例演示如何處理使用巢狀配置檔案物件建立使用者。

``` python
class UserSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer()

    class Meta:
        model = User
        fields = ('username', 'email', 'profile')

    def create(self, validated_data):
        profile_data = validated_data.pop('profile')
        user = User.objects.create(**validated_data)
        Profile.objects.create(user=user, **profile_data)
        return user
```

#### 為巢狀表示書寫 `.update()` 方法

對於更新，您需要仔細考慮如何處理關係更新。例如，如果關係的資料是 `None` 或沒有提供，則應發生以下哪種情況？

* 在資料庫中將關係設定為 `NULL`。
* 刪除關聯的例項。
* 忽略資料並保持原樣。
* 引發驗證錯誤。


以下是我們以前的 `UserSerializer` 類中的 `.update()` 方法的示例。

``` python
    def update(self, instance, validated_data):
        profile_data = validated_data.pop('profile')
        # Unless the application properly enforces that this field is
        # always set, the follow could raise a `DoesNotExist`, which
        # would need to be handled.
        profile = instance.profile

        instance.username = validated_data.get('username', instance.username)
        instance.email = validated_data.get('email', instance.email)
        instance.save()

        profile.is_premium_member = profile_data.get(
            'is_premium_member',
            profile.is_premium_member
        )
        profile.has_support_contract = profile_data.get(
            'has_support_contract',
            profile.has_support_contract
         )
        profile.save()

        return instance
```

因為巢狀建立和更新的行為可能不明確，並且可能需要相關模型之間的複雜依賴關係，所以 REST framework 3 要求你始終明確寫入這些方法。預設的 `ModelSerializer` 的 `.create()`和 `.update()` 方法不包括對可寫巢狀表示的支援。

不過，有第三方軟體包可用，如支援自動可寫巢狀表示的 [DRF Writable Nested](http://www.django-rest-framework.org/api-guide/serializers/#drf-writable-nested)。


#### 在模型管理器類中儲存相關的例項

在序列化類中儲存多個相關例項的另一種方法是編寫自定義模型管理器類。

例如，假設我們希望確保 `User` 例項和 `Profile` 例項始終作為一對建立。我們可能會編寫一個類似下面的自定義管理器類：

``` python
class UserManager(models.Manager):
    ...

    def create(self, username, email, is_premium_member=False, has_support_contract=False):
        user = User(username=username, email=email)
        user.save()
        profile = Profile(
            user=user,
            is_premium_member=is_premium_member,
            has_support_contract=has_support_contract
        )
        profile.save()
        return user
```

此管理器類現在更好地封裝了使用者例項和配置檔案例項始終在同一時間建立。現在可以重新編寫序列化類上的 `.create()`方法，以使用新的管理類方法。

``` python
def create(self, validated_data):
    return User.objects.create(
        username=validated_data['username'],
        email=validated_data['email']
        is_premium_member=validated_data['profile']['is_premium_member']
        has_support_contract=validated_data['profile']['has_support_contract']
    )
```


### 10. 處理多個物件

`Serializer` 類還可以處理序列化或反序列化物件列表。

#### 序列化多個物件

要序列化查詢集或物件列表而不是單個物件例項，在例項化序列化類時，應該傳遞 `many=True` 標誌。然後，您可以傳遞要序列化的查詢集或物件列表。

``` python
queryset = Book.objects.all()
serializer = BookSerializer(queryset, many=True)
serializer.data
# [
#     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
#     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
#     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
# ]
```

#### 資料反序列化為多個物件

反序列化多個物件的預設行為是**支援多個物件建立**，但`不支援多個物件更新`。



### 11. 包含額外的上下文

除了被序列化的物件外，還有一些情況需要為序列化類提供額外的上下文。一種常見的情況是，如果你使用的是包含超連結關係的序列化類，則需要序列化類訪問當前請求，以便它可以正確生成完全限定的URL。

在例項化序列化物件時，你可以通過傳遞上下文引數來提供任意附加上下文。例如：

``` python
serializer = AccountSerializer(account, context={'request': request})
serializer.data
# {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}
```

通過訪問 `self.context` 屬性，可以在任何序列化物件欄位邏輯中使用上下文字典，例如自定義的 `.to_representation()` 方法。

## ModelSerializer

通常你會想要序列化類緊密地對映到 Django 模型定義上。

`ModelSerializer` 類提供了一個快捷方式，可讓你自動建立一個 `Serializer` 類，其中的欄位與模型類欄位對應。

**`ModelSerializer` 類與常規 `Serializer` 類相同，不同之處在於：**  
* 它會根據模型自動生成一組欄位。
* 它會自動為序列化類生成驗證器，例如 unique_together 驗證器。
* 它包含 `.create()` 和 `.update()` 的簡單預設實現。

宣告ModelSerializer如下所示：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

預設情況下，該類中的所有模型類欄位將被對映為相應的序列化類欄位。

任何關係（如模型上的外來鍵）都將對映到 `PrimaryKeyRelatedField` 。除非在序列化關係文件中指定，否則預設不包括反向關係。




### 檢查 `ModelSerializer`

序列化類能夠生成一個表示字串，可以讓你充分檢查其欄位的狀態。在使用 `ModelSerializer` 進行工作時，這是特別有用的，你需要確定它為你自動建立了哪些欄位和驗證器。

為此，使用 `python manage.py shell` 進入 Django shell，然後匯入序列化類，例項化它並列印物件表示形式...

``` python
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print(repr(serializer))
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```


### 指定要包含的欄位

如果你只希望在模型序列化程式中使用預設欄位的子集，則可以使用 `fields` 或 `exclude` 選項來完成此操作，就像使用 `ModelForm` 一樣。強烈建議你顯式使用 `fields` 屬性序列化的所有欄位。這將使你不太可能在模型更改時無意中暴露資料。

舉個栗子：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

你還可以將 `fields` 屬性設定為特殊值 `'__all__'`，以指示應該使用模型中的所有欄位。

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = '__all__'
```


你可以將 `exclude` 屬性設定為從序列化程式中排除的欄位列表。

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        exclude = ('users',)
```

在上面的示例中，如果 `Account` 模型有3個欄位 `account_name`，`users` 和 `created`，則會導致欄位 `account_name` 和 `created` 被序列化。

`fields` 和 `exclude` 屬性中的名稱通常對映到模型類的模型欄位。

或者`fields`選項中的名稱可以對映成屬性或方法。而不會變成模型類中的引數。


從版本 3.3.0 開始，必須提供其中一個屬性 `fields` 或 `exclude`。

### 指定巢狀序列化


預設的 `ModelSerializer` 使用主鍵進行關聯，但你也可以使用 `depth` 選項輕鬆生成巢狀表示（自關聯）：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        depth = 1
```

`depth` 選項應設定為一個整數值，該值指示在還原為平面表示之前應該遍歷的關聯的深度。


如果你想自定義序列化的方式，你需要自己定義欄位。


### 顯式指定欄位

你可以將額外的欄位新增到 `ModelSerializer`，或者通過在類上宣告欄位來覆蓋預設欄位，就像你對 `Serializer` 類所做的那樣。

``` python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)
    groups = serializers.PrimaryKeyRelatedField(many=True)

    class Meta:
        model = Account
```

額外的欄位可以對應於模型上的任何屬性或可呼叫的欄位。

### 指定只讀欄位

你可能希望將多個欄位指定為只讀。不要顯式給每個欄位新增 `read_only = True`屬性，你可以使用快捷方式 Meta 選項 `read_only_fields` 。

該選項應該是欄位名稱的列表或元組，宣告如下：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        read_only_fields = ('account_name',)
```

含有 `editable = False`的模型欄位，`AutoField` 欄位預設設定為只讀，並且不需要新增到 `read_only_fields` 選項。


> **注意**： 有一種特殊情況，只讀欄位是模型級別的 `unique_together` 約束的一部分。在這種情況下，序列化類需要驗證約束該欄位，但也不能由使用者編輯。

> 處理這個問題的正確方法是在序列化類中明確指定欄位，同時提供 `read_only = True` 和 `default = ...` 關鍵字引數。

> 其中一個例子是與當前認證 `User` 的只讀關係，它與另一個識別符號是 `unique_together` 。在這種情況下，你會像這樣宣告使用者欄位：

``` python
user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())
```

> 關於驗證以後還會再說

### 其他關鍵字引數

還有一個快捷方式允許你使用 `extra_kwargs` 選項在欄位上指定任意附加關鍵字引數。與 `read_only_fields` 的情況一樣，這意味著你不需要在序列化類中顯式宣告該欄位。

該選項是一個字典，將欄位名稱對映到關鍵字引數字典。例如：

``` python
class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('email', 'username', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```



### 關係欄位

序列化模型例項時，可以選擇多種不同的方式來表示關係。`ModelSerializer` 的預設表示是使用相關例項的主鍵。

其他表示形式包括使用超連結序列化，序列化完整巢狀表示形式或使用自定義表示形式序列化。


### 自定義欄位對映

ModelSerializer 類還公開了一個可以覆蓋的 API，以便在例項化序列化物件時更改序列化物件的欄位。


通常，如果 `ModelSerializer` 沒辦法生成預設需要的欄位，那麼你應該將它們明確地新增到類中，或者直接使用常規的`Serializer` 類。但是，在某些情況下，你可能需要建立一個新的基類，以定義如何為給定模型建立序列化物件的欄位。


**`.serializer_field_mapping`**

Django 模型類到 REST framework 序列化類的對映。你可以重寫此對映來更改應該用於每個模型類的預設序列化類。


**`.serializer_related_field`**

該屬性應該是序列化器欄位類，預設情況下用於關係欄位。

對於 `ModelSerializer`，它預設為 `PrimaryKeyRelatedField`。

對於 `HyperlinkedModelSerializer`，它預設為 `serializers.HyperlinkedRelatedField`。

**`serializer_url_field`**

序列化器欄位類，應該用於序列化類中的任何 `url` 欄位。

預設是 `serializers.HyperlinkedIdentityField` 。


**`serializer_choice_field`**

序列化器欄位類，應該用於序列化程式中的任何選擇欄位。

預設是 `serializers.ChoiceField`。


####  field_class 和  field_kwargs API

呼叫以下方法來確定應該自動包含在序列化程式中的每個欄位的類和關鍵字引數。這些方法都應返回兩個元組 `(field_class, field_kwargs)`。



**`.build_standard_field(self, field_name, model_field)`**

呼叫以生成對映到標準模型欄位的序列化器欄位。

預設實現基於 `serializer_field_mapping` 屬性返回序列化類。

**`.build_relational_field(self, field_name, relation_info)`**

呼叫以生成對映到關係模型欄位的序列化器欄位。

預設實現基於 `serializer_relational_field` 屬性返回一個序列化類。 

`relation_info` 引數是一個命名元組，它包含 `model_field`，`related_model`，`to_many`和 `has_through_model` 屬性。

**`.build_nested_field(self, field_name, relation_info, nested_depth)`**

當 `depth` 選項已設定時，呼叫以生成對映到關係模型欄位的序列化程式欄位。

預設實現動態建立一個基於 `ModelSerializer` 或 `HyperlinkedModelSerializer` 的巢狀序列化類。

`nested_depth` 將是 `depth` 選項的值減 1。

`relation_info` 引數是一個命名元組，它包含 `model_field`，`related_model`，`to_many`和 `has_through_model` 屬性。


**`.build_property_field(self, field_name, model_class)`**

呼叫以生成對映到模型類上的屬性或零引數方法的序列化器欄位。

預設實現返回一個 `ReadOnlyField` 類。


**`.build_url_field(self, field_name, model_class)`**

被呼叫來為序列化器自己的 `url` 欄位生成一個序列化器欄位。

預設實現返回一個 `HyperlinkedIdentityField` 類。


**`.build_unknown_field(self, field_name, model_class)`**

當欄位名稱未對映到任何模型欄位或模型屬性時呼叫。預設實現會引發錯誤。但是子類可以自定義這種行為。



## HyperlinkedModelSerializer

`HyperlinkedModelSerializer` 類與 `ModelSerializer` 類相似，只不過它使用超連結來表示關係而不是主鍵。

預設情況下，序列化器將包含一個 `url` 欄位而不是主鍵欄位。

url 欄位將使用 `HyperlinkedIdentityField` 序列化器欄位來表示，並且模型上的任何關係都將使用 `HyperlinkedRelatedField` 序列化器欄位來表示。

你可以通過將主鍵新增到 `fields` 選項來明確包含主鍵，例如：

``` python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ('url', 'id', 'account_name', 'users', 'created')
```


### 絕對和相對 URL

在例項化 `HyperlinkedModelSerializer` 時，必須在序列化上下文中包含當前請求，例如：

``` python
serializer = AccountSerializer(queryset, context={'request': request})
```

這樣做將確保超連結可以包含適當的主機名，以便生成完全限定的 URL，例如：

``` python
http://api.example.com/accounts/1/
```

而不是相對的 URL，例如：

``` python
/accounts/1/
```

如果你確實想要使用相對 URL，則應該在序列化上下文中顯式傳遞 `{'request'：None}`。


### 如何確定超連結檢視

需要確定哪些檢視應該用於超連結到模型例項。

預設情況下，超連結預期對應於與樣式 `'{model_name}-detail'` 匹配的檢視名稱，並通過 `pk` 關鍵字引數查詢例項。

您可以使用 `extra_kwargs` 設定中的 `view_name` 和 `lookup_field` 選項覆蓋 URL 欄位檢視名稱和查詢欄位，如下所示：

``` python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ('account_url', 'account_name', 'users', 'created')
        extra_kwargs = {
            'url': {'view_name': 'accounts', 'lookup_field': 'account_name'},
            'users': {'lookup_field': 'username'}
        }
```


或者，可以顯式設定序列化類中的欄位。例如：

``` python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='accounts',
        lookup_field='slug'
    )
    users = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        lookup_field='username',
        many=True,
        read_only=True
    )

    class Meta:
        model = Account
        fields = ('url', 'account_name', 'users', 'created')
```

> 提示：正確地匹配超連結和 URL conf 有時可能有點困難。列印 `HyperlinkedModelSerializer` 例項的 `repr` 是一種特別有用的方法，可以準確檢查這些關係預期對映的 view name 和 lookup field。


### 更改 URL 欄位名稱

URL 欄位的名稱預設為 'url'。可以使用 `URL_FIELD_NAME`  （在 settings 檔案）全域性覆蓋此設定。

## ListSerializer

`ListSerializer` 類提供了一次序列化和驗證多個物件的行為。你通常不需要直接使用 `ListSerializer`，而應該在例項化序列化類時簡單地傳遞 `many=True`。

當一個序列化類被例項化並且 `many = True` 被傳遞時，一個 `ListSerializer` 例項將被建立。序列化類成為父級 `ListSerializer` 的子級

以下引數也可以傳遞給 `ListSerializer` 欄位或傳遞了 `many = True` 的序列化類：

### `allow_empty`

預設情況下為 `True`，但如果要禁止將空列表作為有效輸入，則可將其設定為 `False`。


### 自定義 `ListSerializer` 行為

有幾種情況可能需要自定義 `ListSerializer` 行為。例如：  
* 希望提供對列表的特定驗證，例如檢查一個元素是否與列表中的另一個元素不衝突。
* 想要自定義多個物件的建立或更新行為。


對於這些情況，可以通過使用序列化類的 `Meta` 類中的 `list_serializer_class` 選項來修改傳遞了 `many=True` 時使用的類。

``` python
class CustomListSerializer(serializers.ListSerializer):
    ...

class CustomSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = CustomListSerializer
```


#### 自定義多個物件的建立

建立多個物件的預設實現是簡單地為列表中的每個 item 呼叫 `.create()`。如果要自定義此行為，則需要在傳遞 `many=True`時自定義 `ListSerializer` 類上的 `.create()`方法。

``` python
class BookListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        books = [Book(**item) for item in validated_data]
        return Book.objects.bulk_create(books)

class BookSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = BookListSerializer
```


#### 自定義多個物件的更新

預設情況下，`ListSerializer` 類不支援多物件更新。這是因為插入和刪除預期的行為是不明確的。

為了支援多物件更新，你需要重寫 update 方法。在編寫你的多物件更新程式碼時，一定要記住以下幾點：  
* 如何確定應該為資料列表中的每個 item 更新哪個例項？
* 插入應該如何處理？它們是無效的，還是建立新物件？
* 應該如何處理刪除？它們是否暗示了物件刪除，或者刪除了一段關係？它們應該被忽略，還是無效？
* 如何處理排序？改變兩個 item 的位置是否意味著狀態的改變或者被忽略？



你需要為例項序列化類新增一個顯式 `id` 欄位。預設的隱式生成的 `id` 欄位被標記為 `read_only`。這會導致它在更新時被刪除。一旦你明確宣告它，它將在列表序列化類的更新方法中可用。

下面是你如何選擇實現多物件更新的示例：

``` python
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        # Maps for id->instance and id->data item.
        book_mapping = {book.id: book for book in instance}
        data_mapping = {item['id']: item for item in validated_data}

        # Perform creations and updates.
        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
            else:
                ret.append(self.child.update(book, data))

        # Perform deletions.
        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    # We need to identify elements in the list using their primary key,
    # so use a writable field here, rather than the default which would be read-only.
    id = serializers.IntegerField()
    ...

    class Meta:
        list_serializer_class = BookListSerializer
```


#### 自定義 ListSerializer 初始化

當具有 `many=True`的序列化類例項化時，我們需要確定哪些引數和關鍵字引數應該傳遞給子級 `Serializer`類和父級 `ListSerializer` 類的 `.__ init __()` 方法。


預設的實現是將所有引數傳遞給兩個類，除了 `validators` 和任何自定義關鍵字引數，這兩個引數都假定用於子序列化類。

有時你可能需要明確指定在傳遞 `many=True` 時如何例項化子類和父類。您可以使用 `many_init` 類方法來完成此操作。


``` python
    @classmethod
    def many_init(cls, *args, **kwargs):
        # Instantiate the child serializer.
        kwargs['child'] = cls()
        # Instantiate the parent list serializer.
        return CustomListSerializer(*args, **kwargs)
```


## BaseSerializer

`BaseSerializer` 類可以用來方便地支援其他序列化和反序列化風格。

這個類實現了與 `Serializer` 類相同的基本 API：  
* `.data` - 返回傳出的原始表示。
* `.is_valid()` - 反序列化並驗證傳入的資料。
* `.validated_data` - 返回驗證的傳入資料。
* `.errors` - 在驗證期間返回錯誤。
* `.save()` - 將驗證的資料儲存到物件例項中。

有四種方法可以被覆蓋，這取決於你希望序列化類支援的功能：  
* `.to_representation()` - 重寫此操作以支援序列化，用於讀取操作。
* `.to_internal_value()` - 重寫此操作以支援反序列化，以用於寫入操作。
* `.create() 和 .update()` - 覆蓋其中一個或兩個以支援儲存例項。

因為這個類提供了與 `Serializer` 類相同的介面，所以你可以像現有的常規 `Serializer`或 `ModelSerializer` 一樣，將它與基於類的通用檢視一起使用。

唯一不同的是，`BaseSerializer` 類不會在可瀏覽的 API 中生成 HTML 表單。這是因為它們返回的資料不包含所有的欄位資訊，這些欄位資訊允許將每個欄位渲染為合適的 HTML 輸入。


### Read-only `BaseSerializer` 類

要使用 `BaseSerializer` 類實現只讀序列化類，我們只需重寫 `.to_representation()` 方法。讓我們來看一個使用簡單的 Django 模型的示例：

``` python
class HighScore(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    player_name = models.CharField(max_length=10)
    score = models.IntegerField()
```

建立用於將 `HighScore` 例項轉換為基本資料型別的只讀序列化類非常簡單。

``` python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }
```

我們現在可以使用這個類來序列化單個 `HighScore` 例項：

``` python
@api_view(['GET'])
def high_score(request, pk):
    instance = HighScore.objects.get(pk=pk)
    serializer = HighScoreSerializer(instance)
    return Response(serializer.data)
```

或者用它來序列化多個例項：

``` python
@api_view(['GET'])
def all_high_scores(request):
    queryset = HighScore.objects.order_by('-score')
    serializer = HighScoreSerializer(queryset, many=True)
    return Response(serializer.data)
```

### Read-write BaseSerializer 類

要建立一個可讀寫的序列化類，我們首先需要實現一個 `.to_internal_value()` 方法。此方法返回將用於構造物件例項的驗證值，並且如果提供的資料格式不正確，則可能引發 `ValidationError` 。

一旦實現 `.to_internal_value()`，基本驗證 API 將在序列化器中可用，並且你將能夠使用 `.is_valid()`，`.validated_data` 和 `.errors`。

如果你還想支援 `.save()`，則還需要實現 `.create()` 和 `.update()`方法中的一個或兩個。


以下是我們之前的 `HighScoreSerializer` 的一個完整示例，該示例已更新為支援讀取和寫入操作。


``` python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_internal_value(self, data):
        score = data.get('score')
        player_name = data.get('player_name')

        # Perform the data validation.
        if not score:
            raise ValidationError({
                'score': 'This field is required.'
            })
        if not player_name:
            raise ValidationError({
                'player_name': 'This field is required.'
            })
        if len(player_name) > 10:
            raise ValidationError({
                'player_name': 'May not be more than 10 characters.'
            })

        # Return the validated values. This will be available as
        # the `.validated_data` property.
        return {
            'score': int(score),
            'player_name': player_name
        }

    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }

    def create(self, validated_data):
        return HighScore.objects.create(**validated_data)
```

### 建立新的基類

如果你希望實現新的泛型序列化類來處理特定的序列化風格，或者與可選的儲存後端進行整合，那麼 `BaseSerializer` 類也很有用。

以下類是可以處理將任意物件強制轉換為基本表示形式的泛型序列化類的示例。


``` python
class ObjectSerializer(serializers.BaseSerializer):
    """
    A read-only serializer that coerces arbitrary complex objects
    into primitive representations.
    """
    def to_representation(self, obj):
        for attribute_name in dir(obj):
            attribute = getattr(obj, attribute_name)
            if attribute_name('_'):
                # Ignore private attributes.
                pass
            elif hasattr(attribute, '__call__'):
                # Ignore methods and other callables.
                pass
            elif isinstance(attribute, (str, int, bool, float, type(None))):
                # Primitive types can be passed through unmodified.
                output[attribute_name] = attribute
            elif isinstance(attribute, list):
                # Recursively deal with items in lists.
                output[attribute_name] = [
                    self.to_representation(item) for item in attribute
                ]
            elif isinstance(attribute, dict):
                # Recursively deal with items in dictionaries.
                output[attribute_name] = {
                    str(key): self.to_representation(value)
                    for key, value in attribute.items()
                }
            else:
                # Force anything else to its string representation.
                output[attribute_name] = str(attribute)
```

## Serializer 使用進階

### 重寫序列化和反序列化行為

如果你需要更改序列化類的序列化或反序列化行為，可以通過覆蓋 `.to_representation()` 或 `.to_internal_value()` 方法來實現。

以下原因可能需要重寫這兩個方法...  

* 為新的序列化基類新增新行為。
* 稍微修改現有類的行為。
* 提高經常訪問的 API 端點的序列化效能，以便返回大量資料。


這些方法的簽名如下：

**`.to_representation(self, obj)`**

接受需要序列化的物件例項，並返回一個原始表示。通常這意味著返回一個內建 Python 資料型別的結構。可以處理的確切型別取決於您為 API 配置的渲染類。

可能會被重寫以便修改表示風格。例如：

``` python
def to_representation(self, instance):
    """Convert `username` to lowercase."""
    ret = super().to_representation(instance)
    ret['username'] = ret['username'].lower()
    return ret
```

**`.to_internal_value(self, data)`**

將未驗證的傳入資料作為輸入，並應返回將作為 `serializer.validated_data` 提供的驗證資料。如果在序列化類上呼叫了 `.save()` ，則返回值也將傳遞給 `.create()` 或 `.update()` 方法。


如果驗證失敗，則該方法會引發 `serializers.ValidationError(errors)`。`errors` 引數應該是一個由欄位名稱（或 `settings.NON_FIELD_ERRORS_KEY`）對映到錯誤訊息列表的字典。如果不需要改變反序列化行為，而是想提供物件級驗證，則建議改為覆蓋 `.validate()`方法。


傳遞給此方法的 `data` 引數通常是 `request.data` 的值，因此它提供的資料型別將取決於你為 API 配置的解析器類。



### 繼承序列化類

與 Django 表單類似，你可以通過繼承來擴充套件和重用序列化類。這使你可以在父類上宣告一組通用的欄位或方法，然後可以在多個序列化類中使用它們。例如，

``` python
class MyBaseSerializer(Serializer):
    my_field = serializers.CharField()

    def validate_my_field(self):
        ...

class MySerializer(MyBaseSerializer):
    ...
```

與 Django 的 `Model` 和 `ModelForm` 類一樣，序列化類中的內部 `Meta` 類不會從其父類的內部 `Meta` 類中隱式繼承。如果你想讓 `Meta` 類繼承父類，必須明確的指出。例如：

``` python
class AccountSerializer(MyBaseSerializer):
    class Meta(MyBaseSerializer.Meta):
        model = Account
```

通常我們建議不要在內部的 `Meta` 類中使用繼承，而是顯式宣告所有選項。

### 動態修改欄位

一旦序列化類初始化完畢，就可以使用 `.fields` 屬性訪問在序列化類中設定的欄位字典。通過訪問和修改這個屬性可以達到動態地修改序列化類的目的。

直接修改 `fields` 引數允許你做一些有趣的事情，比如在執行時改變序列化欄位的引數，而不是在宣告序列化類的時候。

舉個栗子：

例如，如果你希望能夠設定序列化類在初始化時應使用哪些欄位，你可以建立這樣一個序列化類，如下所示：

``` python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes an additional `fields` argument that
    controls which fields should be displayed.
    """

    def __init__(self, *args, **kwargs):
        # Don't pass the 'fields' arg up to the superclass
        fields = kwargs.pop('fields', None)

        # Instantiate the superclass normally
        super(DynamicFieldsModelSerializer, self).__init__(*args, **kwargs)

        if fields is not None:
            # Drop any fields that are not specified in the `fields` argument.
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
```

這將允許你執行以下操作：

``` python
>>> class UserSerializer(DynamicFieldsModelSerializer):
>>>     class Meta:
>>>         model = User
>>>         fields = ('id', 'username', 'email')
>>>
>>> print UserSerializer(user)
{'id': 2, 'username': 'jonwatts', 'email': 'jon@example.com'}
>>>
>>> print UserSerializer(user, fields=('id', 'email'))
{'id': 2, 'email': 'jon@example.com'}
```