
1. 建立序列化類
    1. 直接使用父類serializer.ModelSerializer
    1. class Meta:
1. 熟悉序列化\反序列化的操作
1. 通過Serializer編寫Django View，並使用DRF的CBV(使用基於泛型類的view，或更簡單地使用viewset)，同時urls.py或DRF router要設置。
1. 使用預設有包裝器功的request.data與Response(data)以及format(後綴)
1. 設置認證與權限。


In [16]: tokens = Token.objects.all()

In [17]: tokens[0].key
Out[17]: '845bc555d179deb6d5c573c107dc5f28951154b7'

In [18]: tokens[1].key
Out[18]: 'a158139e6ff467c5ecc8a2b6fa1916f868323459'

In [19]: tokens[2].key
Out[19]: 'd9cb0e5dbe5020b9dbb86ca11baab665c3586964'
