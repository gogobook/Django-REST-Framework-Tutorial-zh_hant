> [官方原文链接](http://www.django-rest-framework.org/api-guide/serializers/)

## Serializers

序列化器允许将诸如查询集和模型實例之類的复杂数据转换为原生 Python 数据類型，然后可以将它们轻松地呈现为 `JSON`，`XML` 或其他内容類型。序列化器还提供反序列化，在首次验证传入数据之后，可以将解析的数据转换回复杂類型。

REST framework 中的序列化類與 Django 的 `Form` 和 `ModelForm` 類非常相似。我们提供了一個 `Serializer` 類，它提供了一种强大的通用方法來控制响应的输出，以及一個 `ModelSerializer` 類，它为創建处理模型實例和查询集的序列化提供了有效的快捷方式。

### 声明序列化類

首先創建一個简單的物件用於示例：

``` python
from datetime import datetime

class Comment(object):
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()

comment = Comment(email='leila@example.com', content='foo bar')
```

声明一個序列化類，使用它來序列化和反序列化與 `Comment` 物件相对应的数据。

声明一個序列化類看起來非常類似於声明一個表單：

``` python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

### 序列化物件

现在可以使用 `CommentSerializer` 來序列化评论或评论列表。同样，使用 `Serializer` 類看起來很像使用 `Form` 類。

``` python
serializer = CommentSerializer(comment)
serializer.data
# {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}
```

此时已经将模型實例转换为 Python 原生数据類型。为了完成序列化过程，将数据渲染为 `json`。

``` python
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'
```


### 反序列化物件

反序列化是相似的。首先我们将一個流解析为 Python 原生数据類型...

``` python
from django.utils.six import BytesIO
from rest_framework.parsers import JSONParser

stream = BytesIO(json)
data = JSONParser().parse(stream)
```

...然后我们将这些原生数据類型恢复成通过验证的数据字典。

``` python
serializer = CommentSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}
```




### 保存實例

如果希望能够基於验证的数据返回完整的物件實例，则需要实现 `.create()` 和 `.update()` 方法中的一個或两個。例如：

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

如果物件實例與 Django 模型相对应，还需要确保这些方法将物件保存到数据库。如果 `Comment` 是一個 Django 模型，这些方法可能如下所示：

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

现在，当反序列化数据时，我们可以调用 `.save()` 根据验证的数据返回一個物件實例。

``` python
comment = serializer.save()
```

调用 `.save()` 将創建一個新實例或更新现有實例，具体取决於在實例化序列化類时是否传递了现有實例：

``` python
# .save() will create a new instance.
serializer = CommentSerializer(data=data)

# .save() will update the existing `comment` instance.
serializer = CommentSerializer(comment, data=data)
```

`.create()` 和 `.update()` 方法都是可选的。您可以都不实现，或者实现其中的一個或两個，具体取决於你的序列化類的用例。

#### 将附加属性传递给 `.save()`

有时你会希望你的视图代码能够在保存實例的时候注入额外的数据。这些附加数据可能包含当前用户，当前时间或其他任何不属於请求数据的信息。

``` python
serializer.save(owner=request.user)
```

调用 `.create()` 或 `.update()` 时，任何其他关键字参数都将包含在 `validated_data` 参数中。

#### 直接覆盖 `.save()`。

在某些情况下，`.create()` 和 `.update()` 方法名称可能没有意义。例如，在 “联系人表單” 中，我们可能不会創建新實例，而是发送电子邮件或其他消息。

在这些情况下，可以选择直接覆盖 `.save()`，因为它更具可读性和有意义性。

举個栗子：

``` python
class ContactForm(serializers.Serializer):
    email = serializers.EmailField()
    message = serializers.CharField()

    def save(self):
        email = self.validated_data['email']
        message = self.validated_data['message']
        send_email(from=email, message=message)
```

请注意，在上面的情况下，必须直接访问 `serializer .validated_data` 属性。

### 验证

在反序列化数据时，你总是需要在尝试访问验证数据之前调用 `is_valid()`，或者保存物件實例。如果发生任何验证错误，那么 `.errors` 属性将包含一個代表错误消息的字典。例如：

``` python
serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}
```

字典中的每個键都是字段名称，值是與该字段相对应的错误消息（字符串列表）。`non_field_errors` 键也可能存在，并会列出任何常规验证错误。可以使用 `NON_FIELD_ERRORS_KEY` （在 settings 文件中设置）來定制 `non_field_errors` 关键字的名称。

反序列化 item 列表时，错误将作为代表每個反序列化 item 的字典列表返回。

#### 数据验证时抛出异常

`.is_valid()` 方法带有一個可选的 `raise_exception` 标志，如果存在验证错误，将导致它引发 `serializers.ValidationError` 异常。


这些异常由 REST framework 提供的默认异常处理程序自动处理，并且默认情况下将返回 `HTTP 400 Bad Request`。

``` python
# Return a 400 response if the data was invalid.
serializer.is_valid(raise_exception=True)
```

#### 字段级验证

你可以通过向 `Serializer` 子類添加 `.validate_<field_name>` 方法來指定自定义字段级验证。这些與 Django 表單上的 `.clean_<field_name>` 方法類似。

这些方法只有一個参数，就是需要验证的字段值。

您的 `validate_<field_name>` 方法应返回验证值或引发 `serializers.ValidationError`。

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

> 注意：如果你的序列化程序中声明的 `<field_name>` 参数为 `required = False` ，那么如果未包含该字段，则不会执行此验证步骤。

#### 物件级验证

如果要对多個字段进行其他的验证，请将一個名为 `.validate()` 的方法添加到您的 `Serializer` 子類中。这個方法只有一個参数，它是一個字段值（`field`-`value`）的字典。如果有必要，它应该引发一個 `ValidationError`，或者只是返回验证的值。例如：

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


#### 验证器

序列化器上的各個字段可以包含验证器，方法是在字段實例上声明它们，例如：

``` python
def multiple_of_ten(value):
    if value % 10 != 0:
        raise serializers.ValidationError('Not a multiple of ten')

class GameRecord(serializers.Serializer):
    score = IntegerField(validators=[multiple_of_ten])
    ...
```

序列化類还可以包含应用於整個字段数据集的可重用验证器。这些验证器是通过在内部的 `Meta` 類中声明它们來包含的，如下所示：

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

> 看不懂没关系哦，更多关於验证的内容，以后还会说到。


### 访问初始数据和實例

将初始物件或查询集传递给序列化類實例时，该物件将作为 `.instance` 提供。如果没有传递初始物件，则 `.instance` 属性将为 `None`。

将数据传递给序列化類實例时，未修改的数据将作为 `.initial_data` 提供。如果 data 关键字参数未被传递，那么 `.initial_data` 属性将不存在。


### 部分更新

默认情况下，序列化程序必须为所有必填字段传递值，否则会引发验证错误。您可以使用 `partial` 参数以允许部分更新。

``` python
# Update `comment` with partial data
serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)
```


### 处理嵌套物件

前面的例子适用於处理只具有简單数据類型的物件，但有时还需要能够表示更复杂的物件，其中物件的某些属性可能不是简單的数据類型，如字符串，日期或整数。

`Serializer` 類本身就是一种 `Field`，可以用來表示一個物件類型嵌套在另一個物件類型中的关系。

``` python
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

如果嵌套物件可以是 `None` 值，则应将 `required = False` 标志传递给嵌套的序列化類。

``` python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)  # May be an anonymous user.
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

同样，如果嵌套物件是一個列表，则应将 `many = True` 标志传递给嵌套的序列化類。

``` python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)
    edits = EditItemSerializer(many=True)  # A nested list of 'edit' items.
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

### 可写嵌套表示

在处理支持反序列化数据的嵌套表示时，嵌套物件的任何错误都将嵌套在嵌套物件的字段名称下。

``` python
serializer = CommentSerializer(data={'user': {'email': 'foobar', 'username': 'doe'}, 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'user': {'email': [u'Enter a valid e-mail address.']}, 'created': [u'This field is required.']}
```

同样，`.validated_data` 属性将包含嵌套的数据结构。


#### 为嵌套表示书写 `.create()` 方法

如果你支持可写嵌套表示，则需要编写处理保存多個物件的 `.create()` 或 `.update()` 方法。

以下示例演示如何处理使用嵌套配置文件物件創建用户。

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

#### 为嵌套表示书写 `.update()` 方法

对於更新，您需要仔细考虑如何处理关系更新。例如，如果关系的数据是 `None` 或没有提供，则应发生以下哪种情况？

* 在数据库中将关系设置为 `NULL`。
* 删除关联的實例。
* 忽略数据并保持原样。
* 引发验证错误。


以下是我们以前的 `UserSerializer` 類中的 `.update()` 方法的示例。

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

因为嵌套創建和更新的行为可能不明确，并且可能需要相关模型之间的复杂依赖关系，所以 REST framework 3 要求你始终明确写入这些方法。默认的 `ModelSerializer` 的 `.create()`和 `.update()` 方法不包括对可写嵌套表示的支持。

不过，有第三方软件包可用，如支持自动可写嵌套表示的 [DRF Writable Nested](http://www.django-rest-framework.org/api-guide/serializers/#drf-writable-nested)。


#### 在模型管理器類中保存相关的實例

在序列化類中保存多個相关實例的另一种方法是编写自定义模型管理器類。

例如，假设我们希望确保 `User` 實例和 `Profile` 實例始终作为一对創建。我们可能会编写一個類似下面的自定义管理器類：

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

此管理器類现在更好地封装了用户實例和配置文件實例始终在同一时间創建。现在可以重新编写序列化類上的 `.create()`方法，以使用新的管理類方法。

``` python
def create(self, validated_data):
    return User.objects.create(
        username=validated_data['username'],
        email=validated_data['email']
        is_premium_member=validated_data['profile']['is_premium_member']
        has_support_contract=validated_data['profile']['has_support_contract']
    )
```


### 处理多個物件

`Serializer` 類还可以处理序列化或反序列化物件列表。

#### 序列化多個物件

要序列化查询集或物件列表而不是單個物件實例，在實例化序列化類时，应该传递 `many=True` 标志。然后，您可以传递要序列化的查询集或物件列表。

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

#### 反序列化多個物件

反序列化多個物件的默认行为是支持多個物件創建，但不支持多個物件更新。



### 包含额外的上下文

除了被序列化的物件外，还有一些情况需要为序列化類提供额外的上下文。一种常见的情况是，如果你使用的是包含超链接关系的序列化類，则需要序列化類访问当前请求，以便它可以正确生成完全限定的URL。

在實例化序列化物件时，你可以通过传递上下文参数來提供任意附加上下文。例如：

``` python
serializer = AccountSerializer(account, context={'request': request})
serializer.data
# {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}
```

通过访问 `self.context` 属性，可以在任何序列化物件字段逻辑中使用上下文字典，例如自定义的 `.to_representation()` 方法。

## ModelSerializer

通常你会想要序列化類紧密地映射到 Django 模型定义上。

`ModelSerializer` 類提供了一個快捷方式，可让你自动創建一個 `Serializer` 類，其中的字段與模型類字段对应。

**`ModelSerializer` 類與常规 `Serializer` 類相同，不同之处在於：**  
* 它会根据模型自动生成一组字段。
* 它会自动为序列化類生成验证器，例如 unique_together 验证器。
* 它包含 `.create()` 和 `.update()` 的简單默认实现。

声明ModelSerializer如下所示：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

默认情况下，该類中的所有模型類字段将被映射为相应的序列化類字段。

任何关系（如模型上的外键）都将映射到 `PrimaryKeyRelatedField` 。除非在序列化关系文档中指定，否则默认不包括反向关系。




### 检查 `ModelSerializer`

序列化類能够生成一個表示字符串，可以让你充分检查其字段的状态。在使用 `ModelSerializer` 进行工作时，这是特别有用的，你需要确定它为你自动創建了哪些字段和验证器。

为此，使用 `python manage.py shell` 进入 Django shell，然后导入序列化類，實例化它并打印物件表示形式...

``` python
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print(repr(serializer))
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```


### 指定要包含的字段

如果你只希望在模型序列化程序中使用默认字段的子集，则可以使用 `fields` 或 `exclude` 选项來完成此操作，就像使用 `ModelForm` 一样。强烈建议你显式使用 `fields` 属性序列化的所有字段。这将使你不太可能在模型更改时无意中暴露数据。

举個栗子：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

你还可以将 `fields` 属性设置为特殊值 `'__all__'`，以指示应该使用模型中的所有字段。

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = '__all__'
```


你可以将 `exclude` 属性设置为从序列化程序中排除的字段列表。

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        exclude = ('users',)
```

在上面的示例中，如果 `Account` 模型有3個字段 `account_name`，`users` 和 `created`，则会导致字段 `account_name` 和 `created` 被序列化。

`fields` 和 `exclude` 属性中的名称通常映射到模型類的模型字段。

或者`fields`选项中的名称可以映射成属性或方法。而不会变成模型類中的参数。


从版本 3.3.0 开始，必须提供其中一個属性 `fields` 或 `exclude`。

### 指定嵌套序列化


默认的 `ModelSerializer` 使用主键进行关联，但你也可以使用 `depth` 选项轻松生成嵌套表示（自关联）：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        depth = 1
```

`depth` 选项应设置为一個整数值，该值指示在还原为平面表示之前应该遍历的关联的深度。


如果你想自定义序列化的方式，你需要自己定义字段。


### 显式指定字段

你可以将额外的字段添加到 `ModelSerializer`，或者通过在類上声明字段來覆盖默认字段，就像你对 `Serializer` 類所做的那样。

``` python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)
    groups = serializers.PrimaryKeyRelatedField(many=True)

    class Meta:
        model = Account
```

额外的字段可以对应於模型上的任何属性或可调用的字段。

### 指定只读字段

你可能希望将多個字段指定为只读。不要显式给每個字段添加 `read_only = True`属性，你可以使用快捷方式 Meta 选项 `read_only_fields` 。

该选项应该是字段名称的列表或元组，声明如下：

``` python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        read_only_fields = ('account_name',)
```

含有 `editable = False`的模型字段，`AutoField` 字段默认设置为只读，并且不需要添加到 `read_only_fields` 选项。


> **注意**： 有一种特殊情况，只读字段是模型级别的 `unique_together` 约束的一部分。在这种情况下，序列化類需要验证约束该字段，但也不能由用户编辑。

> 处理这個问题的正确方法是在序列化類中明确指定字段，同时提供 `read_only = True` 和 `default = ...` 关键字参数。

> 其中一個例子是與当前认证 `User` 的只读关系，它與另一個标识符是 `unique_together` 。在这种情况下，你会像这样声明用户字段：

``` python
user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())
```

> 关於验证以后还会再说

### 其他关键字参数

还有一個快捷方式允许你使用 `extra_kwargs` 选项在字段上指定任意附加关键字参数。與 `read_only_fields` 的情况一样，这意味着你不需要在序列化類中显式声明该字段。

该选项是一個字典，将字段名称映射到关键字参数字典。例如：

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



### 关系字段

序列化模型實例时，可以选择多种不同的方式來表示关系。`ModelSerializer` 的默认表示是使用相关實例的主键。

其他表示形式包括使用超链接序列化，序列化完整嵌套表示形式或使用自定义表示形式序列化。


### 自定义字段映射

ModelSerializer 類还公开了一個可以覆盖的 API，以便在實例化序列化物件时更改序列化物件的字段。


通常，如果 `ModelSerializer` 没办法生成默认需要的字段，那么你应该将它们明确地添加到類中，或者直接使用常规的`Serializer` 類。但是，在某些情况下，你可能需要創建一個新的基類，以定义如何为给定模型創建序列化物件的字段。


**`.serializer_field_mapping`**

Django 模型類到 REST framework 序列化類的映射。你可以重写此映射來更改应该用於每個模型類的默认序列化類。


**`.serializer_related_field`**

该属性应该是序列化器字段類，默认情况下用於关系字段。

对於 `ModelSerializer`，它默认为 `PrimaryKeyRelatedField`。

对於 `HyperlinkedModelSerializer`，它默认为 `serializers.HyperlinkedRelatedField`。

**`serializer_url_field`**

序列化器字段類，应该用於序列化類中的任何 `url` 字段。

默认是 `serializers.HyperlinkedIdentityField` 。


**`serializer_choice_field`**

序列化器字段類，应该用於序列化程序中的任何选择字段。

默认是 `serializers.ChoiceField`。


####  field_class 和  field_kwargs API

调用以下方法來确定应该自动包含在序列化程序中的每個字段的類和关键字参数。这些方法都应返回两個元组 `(field_class, field_kwargs)`。



**`.build_standard_field(self, field_name, model_field)`**

调用以生成映射到标准模型字段的序列化器字段。

默认实现基於 `serializer_field_mapping` 属性返回序列化類。

**`.build_relational_field(self, field_name, relation_info)`**

调用以生成映射到关系模型字段的序列化器字段。

默认实现基於 `serializer_relational_field` 属性返回一個序列化類。 

`relation_info` 参数是一個命名元组，它包含 `model_field`，`related_model`，`to_many`和 `has_through_model` 属性。

**`.build_nested_field(self, field_name, relation_info, nested_depth)`**

当 `depth` 选项已设置时，调用以生成映射到关系模型字段的序列化程序字段。

默认实现动态創建一個基於 `ModelSerializer` 或 `HyperlinkedModelSerializer` 的嵌套序列化類。

`nested_depth` 将是 `depth` 选项的值减 1。

`relation_info` 参数是一個命名元组，它包含 `model_field`，`related_model`，`to_many`和 `has_through_model` 属性。


**`.build_property_field(self, field_name, model_class)`**

调用以生成映射到模型類上的属性或零参数方法的序列化器字段。

默认实现返回一個 `ReadOnlyField` 類。


**`.build_url_field(self, field_name, model_class)`**

被调用來为序列化器自己的 `url` 字段生成一個序列化器字段。

默认实现返回一個 `HyperlinkedIdentityField` 類。


**`.build_unknown_field(self, field_name, model_class)`**

当字段名称未映射到任何模型字段或模型属性时调用。默认实现会引发错误。但是子類可以自定义这种行为。



## HyperlinkedModelSerializer

`HyperlinkedModelSerializer` 類與 `ModelSerializer` 類相似，只不过它使用超链接來表示关系而不是主键。

默认情况下，序列化器将包含一個 `url` 字段而不是主键字段。

url 字段将使用 `HyperlinkedIdentityField` 序列化器字段來表示，并且模型上的任何关系都将使用 `HyperlinkedRelatedField` 序列化器字段來表示。

你可以通过将主键添加到 `fields` 选项來明确包含主键，例如：

``` python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ('url', 'id', 'account_name', 'users', 'created')
```


### 绝对和相对 URL

在實例化 `HyperlinkedModelSerializer` 时，必须在序列化上下文中包含当前请求，例如：

``` python
serializer = AccountSerializer(queryset, context={'request': request})
```

这样做将确保超链接可以包含适当的主机名，以便生成完全限定的 URL，例如：

``` python
http://api.example.com/accounts/1/
```

而不是相对的 URL，例如：

``` python
/accounts/1/
```

如果你确实想要使用相对 URL，则应该在序列化上下文中显式传递 `{'request'：None}`。


### 如何确定超链接视图

需要确定哪些视图应该用於超链接到模型實例。

默认情况下，超链接预期对应於與样式 `'{model_name}-detail'` 匹配的视图名称，并通过 `pk` 关键字参数查找實例。

您可以使用 `extra_kwargs` 设置中的 `view_name` 和 `lookup_field` 选项覆盖 URL 字段视图名称和查找字段，如下所示：

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


或者，可以显式设置序列化類中的字段。例如：

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

> 提示：正确地匹配超链接和 URL conf 有时可能有点困难。打印 `HyperlinkedModelSerializer` 實例的 `repr` 是一种特别有用的方法，可以准确检查这些关系预期映射的 view name 和 lookup field。


### 更改 URL 字段名称

URL 字段的名称默认为 'url'。可以使用 `URL_FIELD_NAME`  （在 settings 文件）全局覆盖此设置。

## ListSerializer

`ListSerializer` 類提供了一次序列化和验证多個物件的行为。你通常不需要直接使用 `ListSerializer`，而应该在實例化序列化類时简單地传递 `many=True`。

当一個序列化類被實例化并且 `many = True` 被传递时，一個 `ListSerializer` 實例将被創建。序列化類成为父级 `ListSerializer` 的子级

以下参数也可以传递给 `ListSerializer` 字段或传递了 `many = True` 的序列化類：

### `allow_empty`

默认情况下为 `True`，但如果要禁止将空列表作为有效输入，则可将其设置为 `False`。


### 自定义 `ListSerializer` 行为

有几种情况可能需要自定义 `ListSerializer` 行为。例如：  
* 希望提供对列表的特定验证，例如检查一個元素是否與列表中的另一個元素不冲突。
* 想要自定义多個物件的創建或更新行为。


对於这些情况，可以通过使用序列化類的 `Meta` 類中的 `list_serializer_class` 选项來修改传递了 `many=True` 时使用的類。

``` python
class CustomListSerializer(serializers.ListSerializer):
    ...

class CustomSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = CustomListSerializer
```


#### 自定义多個物件的創建

創建多個物件的默认实现是简單地为列表中的每個 item 调用 `.create()`。如果要自定义此行为，则需要在传递 `many=True`时自定义 `ListSerializer` 類上的 `.create()`方法。

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


#### 自定义多個物件的更新

默认情况下，`ListSerializer` 類不支持多物件更新。这是因为插入和删除预期的行为是不明确的。

为了支持多物件更新，你需要重写 update 方法。在编写你的多物件更新代码时，一定要记住以下几点：  
* 如何确定应该为数据列表中的每個 item 更新哪個實例？
* 插入应该如何处理？它们是无效的，还是創建新物件？
* 应该如何处理删除？它们是否暗示了物件删除，或者删除了一段关系？它们应该被忽略，还是无效？
* 如何处理排序？改变两個 item 的位置是否意味着状态的改变或者被忽略？



你需要为實例序列化類添加一個显式 `id` 字段。默认的隐式生成的 `id` 字段被标记为 `read_only`。这会导致它在更新时被删除。一旦你明确声明它，它将在列表序列化類的更新方法中可用。

下面是你如何选择实现多物件更新的示例：

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


#### 自定义 ListSerializer 初始化

当具有 `many=True`的序列化類實例化时，我们需要确定哪些参数和关键字参数应该传递给子级 `Serializer`類和父级 `ListSerializer` 類的 `.__ init __()` 方法。


默认的实现是将所有参数传递给两個類，除了 `validators` 和任何自定义关键字参数，这两個参数都假定用於子序列化類。

有时你可能需要明确指定在传递 `many=True` 时如何實例化子類和父類。您可以使用 `many_init` 類方法來完成此操作。


``` python
    @classmethod
    def many_init(cls, *args, **kwargs):
        # Instantiate the child serializer.
        kwargs['child'] = cls()
        # Instantiate the parent list serializer.
        return CustomListSerializer(*args, **kwargs)
```


## BaseSerializer

`BaseSerializer` 類可以用來方便地支持其他序列化和反序列化风格。

这個類实现了與 `Serializer` 類相同的基本 API：  
* `.data` - 返回传出的原始表示。
* `.is_valid()` - 反序列化并验证传入的数据。
* `.validated_data` - 返回验证的传入数据。
* `.errors` - 在验证期间返回错误。
* `.save()` - 将验证的数据保存到物件實例中。

有四种方法可以被覆盖，这取决於你希望序列化類支持的功能：  
* `.to_representation()` - 重写此操作以支持序列化，用於读取操作。
* `.to_internal_value()` - 重写此操作以支持反序列化，以用於写入操作。
* `.create() 和 .update()` - 覆盖其中一個或两個以支持保存實例。

因为这個類提供了與 `Serializer` 類相同的接口，所以你可以像现有的常规 `Serializer`或 `ModelSerializer` 一样，将它與基於類的通用视图一起使用。

唯一不同的是，`BaseSerializer` 類不会在可浏览的 API 中生成 HTML 表單。这是因为它们返回的数据不包含所有的字段信息，这些字段信息允许将每個字段渲染为合适的 HTML 输入。


### Read-only `BaseSerializer` 類

要使用 `BaseSerializer` 類实现只读序列化類，我们只需重写 `.to_representation()` 方法。让我们來看一個使用简單的 Django 模型的示例：

``` python
class HighScore(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    player_name = models.CharField(max_length=10)
    score = models.IntegerField()
```

創建用於将 `HighScore` 實例转换为基本数据類型的只读序列化類非常简單。

``` python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }
```

我们现在可以使用这個類來序列化單個 `HighScore` 實例：

``` python
@api_view(['GET'])
def high_score(request, pk):
    instance = HighScore.objects.get(pk=pk)
    serializer = HighScoreSerializer(instance)
    return Response(serializer.data)
```

或者用它來序列化多個實例：

``` python
@api_view(['GET'])
def all_high_scores(request):
    queryset = HighScore.objects.order_by('-score')
    serializer = HighScoreSerializer(queryset, many=True)
    return Response(serializer.data)
```

### Read-write BaseSerializer 類

要創建一個可读写的序列化類，我们首先需要实现一個 `.to_internal_value()` 方法。此方法返回将用於构造物件實例的验证值，并且如果提供的数据格式不正确，则可能引发 `ValidationError` 。

一旦实现 `.to_internal_value()`，基本验证 API 将在序列化器中可用，并且你将能够使用 `.is_valid()`，`.validated_data` 和 `.errors`。

如果你还想支持 `.save()`，则还需要实现 `.create()` 和 `.update()`方法中的一個或两個。


以下是我们之前的 `HighScoreSerializer` 的一個完整示例，该示例已更新为支持读取和写入操作。


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

### 創建新的基類

如果你希望实现新的泛型序列化類來处理特定的序列化风格，或者與可选的存储后端进行集成，那么 `BaseSerializer` 類也很有用。

以下類是可以处理将任意物件强制转换为基本表示形式的泛型序列化類的示例。


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

## Serializer 使用进阶

### 重写序列化和反序列化行为

如果你需要更改序列化類的序列化或反序列化行为，可以通过覆盖 `.to_representation()` 或 `.to_internal_value()` 方法來实现。

以下原因可能需要重写这两個方法...  

* 为新的序列化基類添加新行为。
* 稍微修改现有類的行为。
* 提高经常访问的 API 端点的序列化性能，以便返回大量数据。


这些方法的签名如下：

**`.to_representation(self, obj)`**

接受需要序列化的物件實例，并返回一個原始表示。通常这意味着返回一個内置 Python 数据類型的结构。可以处理的确切類型取决於您为 API 配置的渲染類。

可能会被重写以便修改表示风格。例如：

``` python
def to_representation(self, instance):
    """Convert `username` to lowercase."""
    ret = super().to_representation(instance)
    ret['username'] = ret['username'].lower()
    return ret
```

**`.to_internal_value(self, data)`**

将未验证的传入数据作为输入，并应返回将作为 `serializer.validated_data` 提供的验证数据。如果在序列化類上调用了 `.save()` ，则返回值也将传递给 `.create()` 或 `.update()` 方法。


如果验证失败，则该方法会引发 `serializers.ValidationError(errors)`。`errors` 参数应该是一個由字段名称（或 `settings.NON_FIELD_ERRORS_KEY`）映射到错误消息列表的字典。如果不需要改变反序列化行为，而是想提供物件级验证，则建议改为覆盖 `.validate()`方法。


传递给此方法的 `data` 参数通常是 `request.data` 的值，因此它提供的数据類型将取决於你为 API 配置的解析器類。



### 继承序列化類

與 Django 表單類似，你可以通过继承來扩展和重用序列化類。这使你可以在父類上声明一组通用的字段或方法，然后可以在多個序列化類中使用它们。例如，

``` python
class MyBaseSerializer(Serializer):
    my_field = serializers.CharField()

    def validate_my_field(self):
        ...

class MySerializer(MyBaseSerializer):
    ...
```

與 Django 的 `Model` 和 `ModelForm` 類一样，序列化類中的内部 `Meta` 類不会从其父類的内部 `Meta` 類中隐式继承。如果你想让 `Meta` 類继承父類，必须明确的指出。例如：

``` python
class AccountSerializer(MyBaseSerializer):
    class Meta(MyBaseSerializer.Meta):
        model = Account
```

通常我们建议不要在内部的 `Meta` 類中使用继承，而是显式声明所有选项。

### 动态修改字段

一旦序列化類初始化完毕，就可以使用 `.fields` 属性访问在序列化類中设置的字段字典。通过访问和修改这個属性可以达到动态地修改序列化類的目的。

直接修改 `fields` 参数允许你做一些有趣的事情，比如在运行时改变序列化字段的参数，而不是在声明序列化類的时候。

举個栗子：

例如，如果你希望能够设置序列化類在初始化时应使用哪些字段，你可以創建这样一個序列化類，如下所示：

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

这将允许你执行以下操作：

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