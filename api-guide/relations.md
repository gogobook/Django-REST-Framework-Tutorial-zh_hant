
> [官方原文連結](http://www.django-rest-framework.org/api-guide/relations/)  

# Serializer 關係

關係字段用於表示模型關係。 它們可以應用於 ForeignKey，ManyToManyField 和 OneToOneField 關係，反向關係以及 GenericForeignKey 等自定義關係。

>**注意**： 關係字段在 relations.py 中聲明，但按照慣例，你應該從 serializers 模組導入它們，使用 from rest_framework import serializers 引入，並像 serializers.<FieldName> 這樣引用字段。

#### 檢查關係。

在使用 `ModelSerializer` 類時，將自動為你生成序列化字段和關係字段。檢查這些自動生成的字段可以學習如何定製關係的格式。

為此，使用 python manage.py shell 打開 Django shell，然後導入序列化類，實例化它並打印物件表示形式...
```python
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print repr(serializer)  # Or `print(repr(serializer))` in Python 3.x.
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```
## API 參考

為瞭解釋各種類型的關係字段，我們將為我們的示例使用幾個簡單的模型。我們的模型將使用音樂專輯為例子，以及每張專輯中列出的曲目。
```python
class Album(models.Model):
    album_name = models.CharField(max_length=100)
    artist = models.CharField(max_length=100)

class Track(models.Model):
    album = models.ForeignKey(Album, related_name='tracks', on_delete=models.CASCADE)
    order = models.IntegerField()
    title = models.CharField(max_length=100)
    duration = models.IntegerField()

    class Meta:
        unique_together = ('album', 'order')
        ordering = ['order']

    def __unicode__(self):
        return '%d: %s' % (self.order, self.title)
```
### StringRelatedField

`StringRelatedField` 用於使用 `__unicode__` 方法表示關係。

例如，下面的序列化類。
```python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.StringRelatedField(many=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```
將序列化為以下形式。
```python
{
    'album_name': 'Things We Lost In The Fire',
    'artist': 'Low',
    'tracks': [
        '1: Sunflower',
        '2: Whitetail',
        '3: Dinosaur Act',
        ...
    ]
}
```
該字段是只讀的。

**參數**:

* `many` - 如果是一對多的關係，就將此參數設置為 True.

### PrimaryKeyRelatedField

`PrimaryKeyRelatedField` 用於使用其主鍵表示關係。

例如，以下序列化類：
```python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.PrimaryKeyRelatedField(many=True, read_only=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```
將序列化為這樣的表示：
```python
{
    'album_name': 'Undun',
    'artist': 'The Roots',
    'tracks': [
        89,
        90,
        91,
        ...
    ]
}
```
默認情況下，該字段是可讀寫的，但您可以使用 read_only 標誌更改此行為。

**參數**:

* `queryset`   - 驗證字段輸入時用於模型實例查詢的查詢集。必須顯式地設置查詢集，或設置 read_only=True。
* `many`       - 如果應用於一對多關係，則應將此參數設置為 True。
* `allow_null` - 如果設置為 True，那麼該字段將接受 None 值或可為空的關係的空字符串。默認為 False。
* `pk_field`   - 設置為一個字段來控制主鍵值的序列化/反序列化。例如， pk_field=UUIDField(format='hex') 會將 UUID 主鍵序列化為其緊湊的十六進製表示形式。

### HyperlinkedRelatedField

`HyperlinkedRelatedField` 用於使用超鏈接來表示關係。

例如，以下序列化類：
```python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.HyperlinkedRelatedField(
        many=True,
        read_only=True,
        view_name='track-detail'
    )

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```
將序列化為這樣的表示：
```python
{
    'album_name': 'Graceland',
    'artist': 'Paul Simon',
    'tracks': [
        'http://www.example.com/api/tracks/45/',
        'http://www.example.com/api/tracks/46/',
        'http://www.example.com/api/tracks/47/',
        ...
    ]
}
```
默認情況下，該字段是可讀寫的，但您可以使用 `read_only` 標誌更改此行為。

注意：該字段是為映射到接受單個 URL 關鍵字參數的 URL 的物件而設計的，如使用 lookup_field 和 lookup_url_kwarg 參數設置的物件。

這適用於包含單個主鍵或 slug 參數作為 URL 一部分的 URL。

如果需要更複雜的超鏈接表示，你需要自定義該字段，稍後會詳解。

**參數**：

* `view_name`        - 用作關係目標的視圖名稱。如果你使用的是標準路由器類，則這將是一個格式為 <modelname>-detail 的字符串。必填.
* `queryset`         - 驗證字段輸入時用於模型實例查詢的查詢集。必須顯式地設置查詢集，或設置 read_only=True。
* `many`             - 如果應用於一對多關係，則應將此參數設置為 True。
* `allow_null`       - 如果設置為 True，那麼該字段將接受 None 值或可為空的關係的空字符串。默認為 False。
* `lookup_field`     - 用於查找的目標字段。對應於引用視圖上的 URL 關鍵字參數。默認是 'pk'.
* `lookup_url_kwarg` - 與查找字段對應的 URL conf 中定義的關鍵字參數的名稱。默認使用與 lookup_field 相同的值。
* `format`           - 如果使用格式後綴，則超鏈接字段將使用與目標相同的格式後綴，除非使用 format 參數進行覆蓋。

### SlugRelatedField

`SlugRelatedField` 用於使用目標上的字段來表示關係。

例如，以下序列化類：
```python
class AlbumSerializer(serializers.ModelSerializer):
    tracks = serializers.SlugRelatedField(
        many=True,
        read_only=True,
        slug_field='title'
     )

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```
將序列化為這樣的表示：
```python
{
    'album_name': 'Dear John',
    'artist': 'Loney Dear',
    'tracks': [
        'Airport Surroundings',
        'Everything Turns to You',
        'I Was Only Going Out',
        ...
    ]
}
```
默認情況下，該字段是可讀寫的，但您可以使用 read_only 標誌更改此行為。

將 SlugRelatedField 用作讀寫字段時，通常需要確保 slug 字段與 unique=True 的模型字段相對應。

參數：

* `slug_field` - 用來表示目標的字段。這應該是唯一標識給定實例的字段。例如， username。必填
* `queryset`   - 驗證字段輸入時用於模型實例查詢的查詢集。必須顯式地設置查詢集，或設置 read_only=True。
* `many`       - 如果應用於一對多關係，則應將此參數設置為 True。
* `allow_null` - 如果設置為 True，那麼該字段將接受 None 值或可為空的關係的空字符串。默認為 False。

### HyperlinkedIdentityField

此字段可以作為身份關係應用，例如 HyperlinkedModelSerializer 上的 'url' 字段。它也可以用於物件的屬性。例如，以下序列化類：
```python
class AlbumSerializer(serializers.HyperlinkedModelSerializer):
    track_listing = serializers.HyperlinkedIdentityField(view_name='track-list')

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'track_listing')
```
將序列化為這樣的表示：
```python
{
    'album_name': 'The Eraser',
    'artist': 'Thom Yorke',
    'track_listing': 'http://www.example.com/api/track_list/12/',
}
```
該字段始終為只讀。

參數：

* `view_name`        - 用作關係目標的視圖名稱。如果你使用的是標準路由器類，則這將是一個格式為 <modelname>-detail 的字符串。必填。
* `lookup_field`     - 用於查找的目標字段。對應於引用視圖上的 URL 關鍵字參數。默認是 'pk'。
* `lookup_url_kwarg` - 與查找字段對應的 URL conf 中定義的關鍵字參數的名稱。默認使用與 lookup_field 相同的值。
* `format`           - 如果使用格式後綴，則超鏈接字段將使用與目標相同的格式後綴，除非使用 format 參數進行覆蓋。

### 嵌套關係

嵌套關係可以通過使用`序列化類`作為字段來表達。

如果該字段用於表示一對多關係，則應將 many=True 標誌添加到序列化字段。
舉個栗子

例如，以下序列化類：
```python
class TrackSerializer(serializers.ModelSerializer):
    class Meta:
        model = Track
        fields = ('order', 'title', 'duration')

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackSerializer(many=True, read_only=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```
將序列化為這樣的嵌套表示：
```python
>>> album = Album.objects.create(album_name="The Grey Album", artist='Danger Mouse')
>>> Track.objects.create(album=album, order=1, title='Public Service Announcement', duration=245)
<Track: Track object>
>>> Track.objects.create(album=album, order=2, title='What More Can I Say', duration=264)
<Track: Track object>
>>> Track.objects.create(album=album, order=3, title='Encore', duration=159)
<Track: Track object>
>>> serializer = AlbumSerializer(instance=album)
>>> serializer.data
{
    'album_name': 'The Grey Album',
    'artist': 'Danger Mouse',
    'tracks': [
        {'order': 1, 'title': 'Public Service Announcement', 'duration': 245},
        {'order': 2, 'title': 'What More Can I Say', 'duration': 264},
        {'order': 3, 'title': 'Encore', 'duration': 159},
        ...
    ],
}
```
### 可寫嵌套序列化類

默認情況下，嵌套序列化類是只讀的。如果要支持對嵌套序列化字段的寫操作，則需要創建 create() 和/或 update() 方法，以明確指定應如何保存子關係。
```python
class TrackSerializer(serializers.ModelSerializer):
    class Meta:
        model = Track
        fields = ('order', 'title', 'duration')

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackSerializer(many=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')

    def create(self, validated_data):
        tracks_data = validated_data.pop('tracks')
        album = Album.objects.create(**validated_data)
        for track_data in tracks_data:
            Track.objects.create(album=album, **track_data)
        return album

>>> data = {
    'album_name': 'The Grey Album',
    'artist': 'Danger Mouse',
    'tracks': [
        {'order': 1, 'title': 'Public Service Announcement', 'duration': 245},
        {'order': 2, 'title': 'What More Can I Say', 'duration': 264},
        {'order': 3, 'title': 'Encore', 'duration': 159},
    ],
}
>>> serializer = AlbumSerializer(data=data)
>>> serializer.is_valid()
True
>>> serializer.save()
<Album: Album object>
```
### 自定義關係字段

在極少數情況下，現有的關係類型都不符合您需要的表示形式，你可以實現一個完全自定義的關係字段，該字段準確描述應該如何從模型實例生成輸出表示。

要實現自定義關係字段，您應該重寫 RelatedField，並實現 .to_representation(self, value) 方法。此方法將字段的目標作為 value 參數，並返回應用於序列化目標的表示。value 參數通常是一個模型實例。

如果要實現讀寫關係字段，則還必須實現 .to_internal_value(self, data) 方法。

要提供基於 context 的動態查詢集，還可以覆蓋 .get_queryset(self)，而不是在類上指定 .queryset 或初始化該字段。
舉個栗子

例如，我們可以定義一個關係字段，使用它的順序，標題和持續時間將音軌序列化為自定義字符串表示。
```python
import time

class TrackListingField(serializers.RelatedField):
    def to_representation(self, value):
        duration = time.strftime('%M:%S', time.gmtime(value.duration))
        return 'Track %d: %s (%s)' % (value.order, value.name, duration)

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackListingField(many=True)

    class Meta:
        model = Album
        fields = ('album_name', 'artist', 'tracks')
```
將序列化為這樣的表示：
```python
{
    'album_name': 'Sometimes I Wish We Were an Eagle',
    'artist': 'Bill Callahan',
    'tracks': [
        'Track 1: Jim Cain (04:39)',
        'Track 2: Eid Ma Clack Shaw (04:19)',
        'Track 3: The Wind and the Dove (04:34)',
        ...
    ]
}
```
### 自定義超鏈接字段

在某些情況下，您可能需要自定義超鏈接字段的行為，以表示需要多個查詢字段的 URL。

您可以通過繼承 HyperlinkedRelatedField 來實現此目的。有兩個可以被覆蓋的方法：

get_url(self, obj, view_name, request, format)

get_url 方法用於將物件實例映射到其 URL 表示。

如果 view_name 和 lookup_field 屬性未配置為正確匹配 URL conf，可能會引發 NoReverseMatch 。

get_object(self, queryset, view_name, view_args, view_kwargs)

如果您想支持可寫的超鏈接字段，那麼您還需要重寫 get_object，以便將傳入的 URL 映射回它們表示的物件。對於只讀超鏈接字段，不需要重寫此方法。

此方法的返回值應該是與匹配的 URL conf 參數對應的物件。

可能會引發 ObjectDoesNotExist 異常。
舉個栗子

假設我們有一個帶有兩個關鍵字參數的 customer 物件的 URL，如下所示：

/api/<organization_slug>/customers/<customer_pk>/

這沒辦法用僅接受單個查找字段的默認實現來表示。

在這種情況下，我們需要繼承 HyperlinkedRelatedField 並重寫其中的方法來獲得我們想要的行為：
```python
from rest_framework import serializers
from rest_framework.reverse import reverse

class CustomerHyperlink(serializers.HyperlinkedRelatedField):
    # We define these as class attributes, so we don't need to pass them as arguments.
    view_name = 'customer-detail'
    queryset = Customer.objects.all()

    def get_url(self, obj, view_name, request, format):
        url_kwargs = {
            'organization_slug': obj.organization.slug,
            'customer_pk': obj.pk
        }
        return reverse(view_name, kwargs=url_kwargs, request=request, format=format)

    def get_object(self, view_name, view_args, view_kwargs):
        lookup_kwargs = {
           'organization__slug': view_kwargs['organization_slug'],
           'pk': view_kwargs['customer_pk']
        }
        return self.get_queryset().get(**lookup_kwargs)
```
請注意，如果您想將此風格與通用視圖一起使用，那麼您還需要覆蓋視圖上的 .get_object 以獲得正確的查找行為。

一般來說，我們建議儘可能在 API 表示方式下使用平面風格，但嵌套 URL 風格在適度使用時也是合理的。
## 進一步說明
### queryset 參數

queryset 參數隻對可寫關係字段是必需的，在這種情況下，它用於執行模型實例查找，該查找從基本用戶輸入映射到模型實例。

在 2.x 版本中，如果正在使用 ModelSerializer 類，則序列化類有時會自動確定 queryset 參數。

此行為現​​在替換為始終為可寫關係字段使用顯式 queryset 參數。

這樣做可以減少 ModelSerializer 提供的隱藏 「魔術」 數量（指 ModelSerializer 在內部幫我們完成的工作），使字段的行為更加清晰，並確保在使用 ModelSerializer 快捷方式（高度封裝過，使用簡單）或使用完全顯式的 Serializer 類之間轉換是微不足道的。
### 自定義 HTML 顯示

模型內置的 `__str__` 方法用來生成用於填充 choices 屬性的物件的字符串表示形式。這些 choices 用於在可瀏覽的 API 中填充選擇的 HTML input。

要為這些 input 提供自定義表示，請重寫 RelatedField 子類的 display_value() 方法。這個方法將接收一個模型物件，並且應該返回一個適合表示它的字符串。例如：
```python
class TrackPrimaryKeyRelatedField(serializers.PrimaryKeyRelatedField):
    def display_value(self, instance):
        return 'Track: %s' % (instance.title)
```
### Select field cutoffs

在渲染可瀏覽的 API 關係字段時，默認只顯示最多 1000 個可選 item。如果存在更多項目，則會顯示 "More than 1000 items…" 的 disabled 選項。

此行為旨在防止由於顯示大量關係而導致模板無法在可接受的時間範圍內完成渲染。

有兩個關鍵字參數可用於控制此行為：

    html_cutoff - 設置 HTML select 下拉菜單中顯示的選項的最大數量。設置為 None 可以禁用任何限制。默認為 1000。
    html_cutoff_text - 設置一個文本字符串，在 HTML select 下拉菜單超出最大顯示數量時顯示。默認是 "More than {count} items…"

你還可以在 settings 中用 HTML_SELECT_CUTOFF 和 HTML_SELECT_CUTOFF_TEXT 來全局控制這些設置。

在強制執行 cutoff 的情況下，您可能希望改為在 HTML 表單中使用簡單的 input 字段。你可以使用 style 關鍵字參數來做到這一點。例如：
```python
assigned_to = serializers.SlugRelatedField(
   queryset=User.objects.all(),
   slug_field='username',
   style={'base_template': 'input.html'}
)
```
### 反向關係

請注意，反向關係不會自動包含在 ModelSerializer 和 HyperlinkedModelSerializer 類中。要包含反向關係，您必須明確將其添加到字段列表中。例如：
```python
class AlbumSerializer(serializers.ModelSerializer):
    class Meta:
        fields = ('tracks', ...)
```
通常需要確保已經在關係上設置了適當的 related_name 參數，可以將其用作字段名稱。例如：
```python
class Track(models.Model):
    album = models.ForeignKey(Album, related_name='tracks', on_delete=models.CASCADE)
    ...
```
如果你還沒有為反向關係設置相關名稱，則需要在 fields 參數中使用自動生成的相關名稱。例如：
```python
class AlbumSerializer(serializers.ModelSerializer):
    class Meta:
        fields = ('track_set', ...)
```
通用關係

如果要序列化通用外鍵，則需要自定義字段，以明確確定如何序列化關係。

例如，給定一個以下模型的標籤，該標籤與其他任意模型具有通用關係：
```python
class TaggedItem(models.Model):
    """
    Tags arbitrary model instances using a generic relation.

    See: https://docs.djangoproject.com/en/stable/ref/contrib/contenttypes/
    """
    tag_name = models.SlugField()
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    tagged_object = GenericForeignKey('content_type', 'object_id')

    def __unicode__(self):
        return self.tag_name
```
以下兩種模式可以用相關的標籤：
```python
class Bookmark(models.Model):
    """
    A bookmark consists of a URL, and 0 or more descriptive tags.
    """
    url = models.URLField()
    tags = GenericRelation(TaggedItem)

```

```python
class Note(models.Model):
    """
    A note consists of some text, and 0 or more descriptive tags.
    """
    text = models.CharField(max_length=1000)
    tags = GenericRelation(TaggedItem)
```
我們可以定義一個可用於序列化標籤實例的自定義字段，並使用每個實例的類型來確定它應該如何序列化。

```python
class TaggedObjectRelatedField(serializers.RelatedField):
    """
    A custom field to use for the `tagged_object` generic relationship.
    """

    def to_representation(self, value):
        """
        Serialize tagged objects to a simple textual representation.
        """
        if isinstance(value, Bookmark):
            return 'Bookmark: ' + value.url
        elif isinstance(value, Note):
            return 'Note: ' + value.text
        raise Exception('Unexpected type of tagged object')
```
如果你需要的關係具有嵌套表示，則可以在 .to_representation() 方法中使用所需的序列化類：
```python
    def to_representation(self, value):
        """
        Serialize bookmark instances using a bookmark serializer,
        and note instances using a note serializer.
        """
        if isinstance(value, Bookmark):
            serializer = BookmarkSerializer(value)
        elif isinstance(value, Note):
            serializer = NoteSerializer(value)
        else:
            raise Exception('Unexpected type of tagged object')

        return serializer.data
```
請注意，使用 GenericRelation 字段表示的反向通用鍵可以使用常規關係字段類型進行序列化，因為關係中目標的類型總是已知的。
具有 Through 模型的 ManyToManyFields

默認情況下，將指定帶有 through 模型的 ManyToManyField 的關係字段設置為只讀。

如果你要明確指定一個指向具有 through 模型的 ManyToManyField 的關係字段，請確保將 read_only 設置為 True 。