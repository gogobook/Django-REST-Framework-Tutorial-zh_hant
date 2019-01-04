如果你正在做基於REST的Web服務......你應該忽略request.POST。

- Django開發團隊的 Malcom Tredinnick

REST framework的Request類擴展了標準HttpRequest，增加了對REST framework靈活請求解析和請求身份驗證的支持。

## 請求解析
REST framework的Request對象提供靈活的請求解析，允許您以與通常處理表單數據相同的方式處理具有JSON數據或其他媒體類型的請求。

### .data
request.data返回請求正文的已解析內容。這類似於標準request.POST和request.FILES屬性，除了：

* 它包括所有已解析的內容，包括文件和非文件輸入。
* 它支持解析除HTTP POST方法之外的其他內容，這意味著您可以訪問內容PUT和PATCH請求。
* 它支持REST framework的靈活請求解析，而不僅僅支持表單數據。例如，您可以像處理傳入表單數據一樣處理傳入的JSON數據。
有關更多詳細信息，請參閱解析器文檔。

### .query_params
request.query_params是一個更正確命名的同義詞request.GET。

為了清楚您的代碼，我們建議使用request.query_params而不是Django的標準request.GET。這樣做有助於保持代碼庫更加正確和明顯 - 任何HTTP方法類型都可能包含查詢參數，而不僅僅是GET請求。

### .parsers
在APIView類或@api_view裝飾將確保這個屬性被自動設置為列表Parser實例的基礎上，parser_classes對視圖設置或基於該DEFAULT_PARSER_CLASSES設置。

您通常不需要訪問此屬性。

注意：如果客戶端發送格式錯誤的內容，則訪問request.data可能會引發ParseError。默認情況下，REST framework的APIView類或@api_view裝飾器將捕獲錯誤並返迴400 Bad Requestresponse 。

如果客戶端發送的請求具有無法解析的內容類型，UnsupportedMediaType則會引發異常，默認情況下將捕獲該異常並返迴415 Unsupported Media Typeresponse 。

## 內容協商
該請求公開了一些允許您確定內容協商階段結果的屬性。這允許您實現行為，例如為不同的媒體類型選擇不同的序列化方案。

### .accepted_renderer
渲染器實例由內容協商階段選擇的內容。

### .accepted_media_type
表示內容協商階段接受的媒體類型的字符串。

## 認證
REST framework提供靈活的按請求身份驗證，使您能夠：

* 對API的不同部分使用不同的身份驗證策略。
* 支持使用多種身份驗證策略。
* 提供與傳入請求關聯的用戶和令牌信息。

### .user
request.user通常返回實例django.contrib.auth.models.User，但行為取決於所使用的身份驗證策略。

如果請求未經身份驗證，則默認值為request.user實例django.contrib.auth.models.AnonymousUser。

欲了解更多詳細信息，請參閱驗證文檔。

### .auth
request.auth返回任何其他身份驗證上下 確切的行為request.auth取決於所使用的身份驗證策略，但它通常可能是請求經過身份驗證的令牌實例。

如果該請求是未認證的，或者如果沒有附加上下文存在時，默認值request.auth是None。

欲了解更多詳細信息，請參閱驗證文檔。

### .authenticators
在APIView類或@api_view裝飾將確保這個屬性被自動設置為列表Authentication實例的基礎上，authentication_classes對視圖設置或基於該DEFAULT_AUTHENTICATORS設置。

您通常不需要訪問此屬性。

注意：WrappedAttributeError調用.user或.auth屬性時可能會看到引發。這些錯誤源自身份驗證器作為標準AttributeError，但是必須將它們重新引發為不同的異常類型，以防止它們被外部屬性訪問抑制。Python不會識別AttributeError來自身份驗證器的orginates，而是假設請求對像沒有.user或.auth屬性。驗證者需要修復。

## 瀏覽器增強功能
REST framework支持的幾個瀏覽器增強功能，例如基於瀏覽器的PUT，PATCH和DELETE形式。

### .method
request.method返回請求的HTTP方法的大寫字符串表示形式。

基於瀏覽器的PUT，PATCH而DELETE形式是透明的支持。

有關更多信息，請參閱瀏覽器增強功能文檔。

### .content_type
request.content_type，返回表示HTTP請求主體的媒體類型的字符串對象，如果未提供媒體類型，則返回空字符串。

您通常不需要直接訪問請求的內容類型，因為您通常會依賴REST framework的默認請求解析行為。

如果您確實需要訪問請求的內容類型，則應.content_type優先使用該屬性request.META.get('HTTP_CONTENT_TYPE')，因為它為基於瀏覽器的非表單內容提供透明支持。

有關更多信息，請參閱瀏覽器增強功能文檔。

### .stream
request.stream 返回表示請求正文內容的流。

您通常不需要直接訪問請求的內容，因為您通常會依賴REST framework的默認請求解析行為。

## 標準的HttpRequest屬性
由於REST framework 的Request擴展了Django HttpRequest，所有其他標準屬性和方法也可用。例如，request.META和request.session字典可以正常使用。

請注意，由於實現原因，Request類不從HttpRequest類繼承，而是使用組合擴展類。