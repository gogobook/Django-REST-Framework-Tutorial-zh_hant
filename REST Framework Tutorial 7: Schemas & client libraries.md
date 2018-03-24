# Tutorial 7: Schemas & client libraries

schema 是一個機器可讀的文件，描述了可用的API endpoints, URLs,及支援怎樣的操作。

對於自動產出型的文件而言，schemas是非常有用的，而且可做為與API 互動的dynamic client libraries的驅動。

## Core API
REST framework使用Core API以提供schema支援。

Core API是一種文件規格用以描述APIs，它被用以提供可用的endpoints和可能的互動的中間表現格式，這是一種API exposes，它可以是server端或client端。

當用在server端，Core API 支援繪出成大range的schema或hypermedia formats.

當在client端，Core API允許動態驅動client libraries，而可與任何exposes a supported schema or hypermedia format 的API互動。

## 添加一個schema
REST framework支援不限於明確定義的schema views，也包含自動產出的schema，由於我們正使用viewsets與routers，我們可以簡單地使用自動schma產出。

你將需要安裝`coreapi`

```sh
    pip install coreapi
```
用添加一個`schema_title`參數到router 實體化上來為我們的APIs包含一個schema

```python
    router = DefaultRouter(schema_title='Pastebin API')
```
假如你在browser上檢視API root endpoint，你現在應該可以看到做為一個選項的corejson呈現型
![Schema format](http://www.django-rest-framework.org/img/corejson-format.png)
我們也可以從command line request the schema，利用指定在Accept header的content type.

```sh
$ http http://127.0.0.1:8000/ Accept:application/vnd.coreapi+json
HTTP/1.0 200 OK
Allow: GET, HEAD, OPTIONS
Content-Type: application/vnd.coreapi+json

{
    "_meta": {
        "title": "Pastebin API"
    },
    "_type": "document",
    ...
}
```
預設的輸出格式是使用Core JSON 編碼。
其他的schema格式，如Open API也有支援。

## Using a command line client
## Authenticating our client
## Reviewing our work
## Onwards and upwards