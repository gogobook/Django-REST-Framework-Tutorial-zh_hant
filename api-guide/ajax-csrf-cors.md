# Working with AJAX, CSRF & CORS
> 在你自己的網站上仔細查看可能的CSRF / XSRF漏洞。它們是最糟糕的漏洞——很容易被攻擊者利用，但對於軟件開發人員來說，這並不容易理解，至少在你被攻擊之前是如此。
> — Jeff Atwood

## Javascript客戶端
如果您正在構建JavaScript客戶端以與您的Web API接口，則需要考慮客戶端是否可以使用網站其餘部分使用的相同認證策略，並確定是否需要使用CSRF令牌或CORS標頭。

在與它們交互的API相同的上下文中創建的AJAX請求通常會使用SessionAuthentication。這確保了一旦用戶已經登錄，任何AJAX請求都可以使用用於網站其他部分的基於會話的同一認證進行認證。

在與他們通信的API不同的站點上發出的AJAX請求通常需要使用基於非會話的身份驗證方案，例如TokenAuthentication。

## CSRF保護
跨站點請求偽造防護是一種防範特定類型攻擊的機制，可能會在用戶未登出網站並發生持續有效會話時發生。在這種情況下，惡意網站可能能夠在登錄會話的上下文中針對目標網站執行操作。

為了防範這些類型的攻擊，您需要做兩件事：

確保“安全”HTTP操作，如GET，HEAD而OPTIONS不能用於改變任何服務器端狀態。
確保所有“不安全”的HTTP操作，如POST，PUT，PATCH和DELETE，總是需要一個有效的CSRF令牌。
如果你使用SessionAuthentication你需要包括有效的CSRF令牌任何POST，PUT，PATCH或DELETE操作。

為了製作AJAX請求，您需要在HTTP標頭中包含CSRF標記，如Django文檔中所述。

## CORS
跨源資源共享是一種允許客戶端與託管在不同域上的API進行交互的機制。CORS的工作原理是要求服務器包含一組特定的標頭，這些標頭允許瀏覽器確定是否允許跨域請求以及何時允許跨域請求。

在REST框架中處理CORS的最好方法是在中間件中添加所需的響應標頭。這確保CORS被透明地支持，而不必改變視圖中的任何行為。

Otto Yiu維護django-cors-headers軟件包，該軟件包可以正確使用REST框架API。