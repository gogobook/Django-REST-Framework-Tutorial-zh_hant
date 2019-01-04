# 設置
> 命名空間是一個好主意 - 讓我們做更多的這些！

> - Python的禪宗

REST框架的配置是在單個Django設置中命名的REST_FRAMEWORK。

例如，您的項目settings.py文件可能包含如下內容：
```py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework.renderers.JSONRenderer',
    ),
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework.parsers.JSONParser',
    )
}
```
### 訪問設置
如果您需要訪問項目中REST框架API設置的值，則應該使用該api_settings對象。例如。
```py
from rest_framework.settings import api_settings

print api_settings.DEFAULT_AUTHENTICATION_CLASSES
```
該api_settings對象將檢查任何用戶定義的設置，否則將回退到默認值。任何使用字符串導入路徑引用類的設置都會自動導入並返回被引用的類，而不是字符串文字。

## API參考
### API策略設置
以下設置控制基本API策略，並應用於每個APIView基於類的視圖或@api_view基於功能的視圖。

#### DEFAULT_RENDERER_CLASSES
渲染器類的列表或元組，用於確定返回Response對象時可能使用的默認渲染器集。

默認：
```py
(
    'rest_framework.renderers.JSONRenderer',
    'rest_framework.renderers.BrowsableAPIRenderer',
)
```
#### DEFAULT_PARSER_CLASSES
解析器類的列表或元組，用於確定訪問request.data屬性時使用的默認解析器集。

默認：
```py
(
    'rest_framework.parsers.JSONParser',
    'rest_framework.parsers.FormParser',
    'rest_framework.parsers.MultiPartParser'
)
```
#### DEFAULT_AUTHENTICATION_CLASSES
身份驗證類的列表或元組，用於確定訪問request.user或request.auth屬性時使用的默認身份驗證集。

默認：
```py
(
    'rest_framework.authentication.SessionAuthentication',
    'rest_framework.authentication.BasicAuthentication'
)
```
#### DEFAULT_PERMISSION_CLASSES
權限類的列表或元組，用於確定在視圖開始時檢查的默認權限集。權限必須由列表中的每個班級授予。

默認：
```py
(
    'rest_framework.permissions.AllowAny',
)
```
#### DEFAULT_THROTTLE_CLASSES
節制類的列表或元組，用於確定在視圖開始時檢查的默認節氣門組。

默認： ()

#### DEFAULT_CONTENT_NEGOTIATION_CLASS
內容協商類，用於確定如何為response 選擇渲染器，並給定傳入請求。

默認： `'rest_framework.negotiation.DefaultContentNegotiation'`

#### DEFAULT_SCHEMA_CLASS
將用於模式生成的視圖檢查器類。

默認： `'rest_framework.schemas.AutoSchema'`

### 通用視圖設置
以下設置控制通用基於類的視圖的行為。

#### DEFAULT_PAGINATION_SERIALIZER_CLASS
此設置已被刪除。

分頁API不使用序列化程序來確定輸出格式，而是需要改寫分頁類上的get_paginated_response方法以指定如何控制輸出格式。

#### DEFAULT_FILTER_BACKENDS
應該用於通用篩選的篩選器後端類列表。如果設置為，None則禁用通用篩選。

#### PAGINATE_BY
此設置已被刪除。

有關設置分頁樣式的進一步指導，請參閱分頁文檔。

#### 頁面大小
用於分頁的默認頁面大小。如果設置為None，則默認情況下禁用分頁。

默認： None

#### PAGINATE_BY_PARAM
此設置已被刪除。

有關設置分頁樣式的進一步指導，請參閱分頁文檔。

#### MAX_PAGINATE_BY
此設置正在等待棄用。

有關設置分頁樣式的進一步指導，請參閱分頁文檔。

### SEARCH_PARAM
查詢參數的名稱，可用於指定所使用的搜索詞SearchFilter。

默認： search

#### ORDERING_PARAM
查詢參數的名稱，可用於指定返回結果的順序OrderingFilter。

默認： ordering

### 版本控制設置
DEFAULT_VERSION
request.version當沒有版本信息存在時應該使用的值。

默認： None

#### ALLOWED_VERSIONS
如果設置，則此值將限製版本控制方案可能返回的版本集合，如果提供的版本不在此集合中，則會引發錯誤。

默認： None

#### VERSION_PARAM
應該用於任何版本控制參數的字符串，例如媒體類型或URL查詢參數。

默認： 'version'

### 認證設置
以下設置控制未經身份驗證的請求的行為。

#### UNAUTHENTICATED_USER
應該用於初始化request.user未經身份驗證的請求的類。（如果刪除認證完全，例如，通過去除django.contrib.auth從 INSTALLED_APPS設置UNAUTHENTICATED_USER到None。）

默認： django.contrib.auth.models.AnonymousUser

#### UNAUTHENTICATED_TOKEN
應該用於初始化request.auth未經身份驗證的請求的類。

默認： None

測試設置
以下設置控制APIRequestFactory和APIClient的行為

#### TEST_REQUEST_DEFAULT_FORMAT
進行測試請求時應使用的默認格式。

這應該與設置中其中一個渲染器類的格式相匹配TEST_REQUEST_RENDERER_CLASSES。

默認： 'multipart'

#### TEST_REQUEST_RENDERER_CLASSES
構建測試請求時支持的渲染器類。

構建測試請求時可以使用任何這些渲染器類的格式，例如： client.post('/users', {'username': 'jamie'}, format='json')

默認：
```py
(
    'rest_framework.renderers.MultiPartRenderer',
    'rest_framework.renderers.JSONRenderer'
)
```
### 架構生成控件
#### SCHEMA_COERCE_PATH_PK
如果設置，則'pk'在生成模式路徑參數時，會將URL conf中的標識符映射到實際字段名稱上。通常情況是這樣'id'。由於“主鍵”是實現細節，因此這提供了更適合的表示，而“標識符”是更一般的概念。

默認： True

#### SCHEMA_COERCE_METHOD_NAMES
如果設置，則用於將內部視圖方法名稱映射到模式生成中使用的外部操作名稱。這使我們能夠生成比代碼庫中內部使用的名稱更適合外部表示的名稱。

默認： {'retrieve': 'read', 'destroy': 'delete'}

### 內容類型控制
#### URL_FORMAT_OVERRIDE
Accept通過format=…在請求URL中使用查詢參數，可用於覆蓋默認內容協商標頭行為的URL參數的名稱。

例如： `http://example.com/organizations/?format=csv`

如果這個設置的值是None那麼URL格式覆蓋將被禁用。

默認： `'format'`

#### FORMAT_SUFFIX_KWARG
URL conf中可用於提供格式後綴的參數名稱。此設置適用於使用format_suffix_patterns後綴URL模式。

例如： `http://example.com/organizations.csv/`

默認： `'format'`

### 日期和時間格式
以下設置用於控制如何分析和渲染日期和時間表示。

#### DATETIME_FORMAT
格式字符串，默認情況下用於呈現DateTimeField序列化程序字段的輸出。如果None，則DateTimeField序列化器字段將返回Python datetime對象，並且日期時間編碼將由渲染器確定。

可能是任何一種None，'iso-8601'或Python strftime格式的字符串。

默認： 'iso-8601'

#### DATETIME_INPUT_FORMATS
默認情況下用於解析DateTimeField序列化器字段輸入的格式字符串列表。

可能是包含字符串'iso-8601'或Python strftime格式字符串的列表。

默認： ['iso-8601']

#### 日期格式
格式字符串，默認情況下用於呈現DateField序列化程序字段的輸出。如果None，則DateField序列化程序字段將返回Python date對象，並且日期編碼將由渲染程序確定。

可能是任何一種None，'iso-8601'或Python strftime格式的字符串。

默認： 'iso-8601'

#### DATE_INPUT_FORMATS
默認情況下用於解析DateField序列化器字段輸入的格式字符串列表。

可能是包含字符串'iso-8601'或Python strftime格式字符串的列表。

默認： ['iso-8601']

#### 時間格式
格式字符串，默認情況下用於呈現TimeField序列化程序字段的輸出。如果None，則TimeField序列化程序字段將返回Python time對象，並且時間編碼將由渲染程序確定。

可能是任何一種None，'iso-8601'或Python strftime格式的字符串。

默認： 'iso-8601'

#### TIME_INPUT_FORMATS
默認情況下用於解析TimeField序列化器字段輸入的格式字符串列表。

可能是包含字符串'iso-8601'或Python strftime格式字符串的列表。

默認： ['iso-8601']

## 編碼
#### UNICODE_JSON
設置True為時，JSONresponse 將允許在response 中使用unicode字符。例如：

{"unicode black star":"★"}
設置False為時，JSONresponse 將轉義非ascii字符，如下所示：

{"unicode black star":"\u2605"}
兩種樣式都符合RFC 4627，並且在語法上有效JSON。在檢查APIresponse 時，unicode樣式更受用戶歡迎。

默認： True

#### COMPACT_JSON
當設置為True，JSONresponse 將返回簡潔表示，用後無間距':'和','字符。例如：

`{"is_admin":false,"email":"jane@example"}`
設置False為時，JSONresponse 將返回稍微更詳細的表示，如下所示：

`{"is_admin": false, "email": "jane@example"}`
默認樣式是返回縮小的response ，符合Heroku的API設計準則。

默認： True

#### STRICT_JSON
當設置為True，JSON渲染和解析只會遵守語法有效的JSON，提高了擴展浮點值異常（nan，inf，-inf）由Python的接受json模塊。這是推薦的設置，因為通常不支持這些值。例如，Javascript JSON.Parse或PostgreSQL的JSON數據類型都不接受這些值。

設置False為時，JSON呈現和解析將是寬容的。但是，這些值仍然無效，需要在代碼中專門處理。

默認： True

#### COERCE_DECIMAL_TO_STRING
在不支持原生十進制類型的API表示形式中返回小數對象時，通常最好將該值作為字符串返回。這樣可以避免二進制浮點實現所帶來的精度損失。

設置True為時，序列化程序DecimalField類將返回字符串而不是Decimal對象。設置False為時，序列化程序將返回Decimal對象，默認的JSON編碼器將返回為浮點數。

默認： True

### 查看名稱和說明
以下設置用於生成視圖名稱和描述，如對OPTIONS請求的response 中所使用的，以及在可瀏覽的API中使用的。

#### VIEW_NAME_FUNCTION
表示生成視圖名稱時應使用的函數的字符串。

這應該是一個具有以下簽名的函數：

`view_name(cls, suffix=None)`
- cls：視圖類。通常，名稱函數會在生成描述性名稱時通過訪問來檢查類的名稱`cls.__name__`。
- suffix：區分視圖中各個視圖時使用的可選後綴。
默認： `'rest_framework.views.get_view_name'`

### VIEW_DESCRIPTION_FUNCTION
表示生成視圖描述時應使用的函數的字符串。

此設置可以更改為支持除默認降價以外的標記樣式。例如，您可以使用它支持rst在可瀏覽的API中輸出的視圖文檔字符串中的標記。

這應該是一個具有以下簽名的函數：

`view_description(cls, html=False)`
- cls：視圖類。通常，描述函數會在生成描述時通過訪問來檢查類的文檔字符串cls.__doc__
- html：一個布爾值，指示是否需要HTML輸出。 True當在可瀏覽的API中False使用時，以及用於生成OPTIONSresponse 時。
默認： 'rest_framework.views.get_view_description'

### HTML選擇字段截止
用於在可瀏覽API中呈現關係字段的選擇字段截斷的全局設置。

#### HTML_SELECT_CUTOFF
html_cutoff價值的全局設置。必須是整數。

默認值：1000

#### HTML_SELECT_CUTOFF_TEXT
代表全局設置的字符串html_cutoff_text。

默認： "More than {count} items..."

## 雜項設置
#### EXCEPTION_HANDLER
一個字符串，表示在返回任何給定異常的response 時應使用的函數。如果函數返回None，將會引發500錯誤。

此設置可以更改為支持默認{"detail": "Failure..."}response 以外的錯誤response 。例如，您可以使用它來提供APIresponse `{"errors": [{"message": "Failure...", "code": ""} ...]}`。

這應該是一個具有以下簽名的函數：

`exception_handler(exc, context)`
- exc：例外。
默認： `'rest_framework.views.exception_handler'`

#### NON_FIELD_ERRORS_KEY
表示應該用於序列化程序錯誤的關鍵字的字符串，該字符串不涉及特定字段，而是一般錯誤。

默認： 'non_field_errors'

#### URL_FIELD_NAME
一個字符串，表示應該用於生成的URL字段的鍵HyperlinkedModelSerializer。

默認： 'url'

NUM_PROXIES
一個0或更大的整數，可用於指定API在後面運行的應用程序代理的數量。這允許限制更準確地識別客戶端IP地址。如果設置為，None則較不嚴格的IP匹配將由節氣門等級使用。

默認： None