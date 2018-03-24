快速看一下使用REST framework 以建立一個簡單的model-backed API。
我們將建立一個read-write API 以存取我們專案的使用者的資訊。
REST framework API 的任何的全域設定被保存在一個單一的設定字典中，其名稱為`REST_FRAMEWORK`，並置於你的`settings.py`中。

```python
REST_FRAMEWORK = {
    # Use Django's standard `django.contrib.auth` permissions,
    # or allow read-only access for unauthenticated users.
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
    ]
}
```
別忘了將`rest_framework`加到`INSTALLED_APPS`中。
現在我們已經準備好了要建立我們的API了，以下是我們的根`urls.py`模組。

```python
from django.conf.urls import url, include
from django.contrib.auth.models import User
from rest_framework import routers, serializers, viewsets

# Serializers define the API representation.
class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ('url', 'username', 'email', 'is_staff')

# ViewSets define the view behavior.
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

# Routers provide an easy way of automatically determining the URL conf.
router = routers.DefaultRouter()
router.register(r'users', UserViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

你現在可以用你的瀏覽器打開API `http://127.0.0.1:8000`， 並且檢視你的新'使用者'API ，假如你使用登錄控制，你將能在系統中建立與刪除使用者。

## Quickstart

Can't wait to get started? The [quickstart guide][quickstart] is the fastest way to get up and running, and building APIs with REST framework.

## Tutorial

The tutorial will walk you through the building blocks that make up REST framework.   It'll take a little while to get through, but it'll give you a comprehensive understanding of how everything fits together, and is highly recommended reading.

* [1 - Serialization][tut-1]
* [2 - Requests & Responses][tut-2]
* [3 - Class-based views][tut-3]
* [4 - Authentication & permissions][tut-4]
* [5 - Relationships & hyperlinked APIs][tut-5]
* [6 - Viewsets & routers][tut-6]
* [7 - Schemas & client libraries][tut-7]

There is a live example API of the finished tutorial API for testing purposes, [available here][sandbox].

## API Guide

The API guide is your complete reference manual to all the functionality provided by REST framework.

* [Requests][request]
* [Responses][response]
* [Views][views]
* [Generic views][generic-views]
* [Viewsets][viewsets]
* [Routers][routers]
* [Parsers][parsers]
* [Renderers][renderers]
* [Serializers][serializers]
* [Serializer fields][fields]
* [Serializer relations][relations]
* [Validators][validators]
* [Authentication][authentication]
* [Permissions][permissions]
* [Throttling][throttling]
* [Filtering][filtering]
* [Pagination][pagination]
* [Versioning][versioning]
* [Content negotiation][contentnegotiation]
* [Metadata][metadata]
* [Schemas][schemas]
* [Format suffixes][formatsuffixes]
* [Returning URLs][reverse]
* [Exceptions][exceptions]
* [Status codes][status]
* [Testing][testing]
* [Settings][settings]

## Topics

General guides to using REST framework.

* [Documenting your API][documenting-your-api]
* [API Clients][api-clients]
* [Internationalization][internationalization]
* [AJAX, CSRF & CORS][ajax-csrf-cors]
* [HTML & Forms][html-and-forms]
* [Browser enhancements][browser-enhancements]
* [The Browsable API][browsableapi]
* [REST, Hypermedia & HATEOAS][rest-hypermedia-hateoas]
* [Third Party Packages][third-party-packages]
* [Tutorials and Resources][tutorials-and-resources]
* [Contributing to REST framework][contributing]
* [Project management][project-management]
* [3.0 Announcement][3.0-announcement]
* [3.1 Announcement][3.1-announcement]
* [3.2 Announcement][3.2-announcement]
* [3.3 Announcement][3.3-announcement]
* [3.4 Announcement][3.4-announcement]
* [3.5 Announcement][3.5-announcement]
* [3.6 Announcement][3.6-announcement]
* [3.7 Announcement][3.7-announcement]
* [Kickstarter Announcement][kickstarter-announcement]
* [Mozilla Grant][mozilla-grant]
* [Funding][funding]
* [Release Notes][release-notes]
* [Jobs][jobs]

## Development

See the [Contribution guidelines][contributing] for information on how to clone
the repository, run the test suite and contribute changes back to REST
Framework.

## Support

For support please see the [REST framework discussion group][group], try the  `#restframework` channel on `irc.freenode.net`, search [the IRC archives][botbot], or raise a  question on [Stack Overflow][stack-overflow], making sure to include the ['django-rest-framework'][django-rest-framework-tag] tag.

For priority support please sign up for a [professional or premium sponsorship plan](https://fund.django-rest-framework.org/topics/funding/).

For updates on REST framework development, you may also want to follow [the author][twitter] on Twitter.

<a style="padding-top: 10px" href="https://twitter.com/_tomchristie" class="twitter-follow-button" data-show-count="false">Follow @_tomchristie</a>
<script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0];if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src="//platform.twitter.com/widgets.js";fjs.parentNode.insertBefore(js,fjs);}}(document,"script","twitter-wjs");</script>

## Security

If you believe you've found something in Django REST framework which has security implications, please **do not raise the issue in a public forum**.

Send a description of the issue via email to [rest-framework-security@googlegroups.com][security-mail].  The project maintainers will then work with you to resolve any issues where required, prior to any public disclosure.

## License

Copyright (c) 2011-2017, Tom Christie
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

Redistributions of source code must retain the above copyright notice, this
list of conditions and the following disclaimer.
Redistributions in binary form must reproduce the above copyright notice, this
list of conditions and the following disclaimer in the documentation and/or
other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[mozilla]: https://www.mozilla.org/en-US/about/
[redhat]: https://www.redhat.com/
[heroku]: https://www.heroku.com/
[eventbrite]: https://www.eventbrite.co.uk/about/
[coreapi]: https://pypi.python.org/pypi/coreapi/
[markdown]: https://pypi.python.org/pypi/Markdown/
[django-filter]: https://pypi.python.org/pypi/django-filter
[django-crispy-forms]: https://github.com/maraujop/django-crispy-forms
[django-guardian]: https://github.com/django-guardian/django-guardian
[index]: .
[oauth1-section]: api-guide/authentication/#django-rest-framework-oauth
[oauth2-section]: api-guide/authentication/#django-oauth-toolkit
[serializer-section]: api-guide/serializers#serializers
[modelserializer-section]: api-guide/serializers#modelserializer
[functionview-section]: api-guide/views#function-based-views
[sandbox]: https://restframework.herokuapp.com/
[sponsors]: https://fund.django-rest-framework.org/topics/funding/#our-sponsors

[quickstart]: tutorial/quickstart.md
[tut-1]: tutorial/1-serialization.md
[tut-2]: tutorial/2-requests-and-responses.md
[tut-3]: tutorial/3-class-based-views.md
[tut-4]: tutorial/4-authentication-and-permissions.md
[tut-5]: tutorial/5-relationships-and-hyperlinked-apis.md
[tut-6]: tutorial/6-viewsets-and-routers.md
[tut-7]: tutorial/7-schemas-and-client-libraries.md

[request]: api-guide/requests.md
[response]: api-guide/responses.md
[views]: api-guide/views.md
[generic-views]: api-guide/generic-views.md
[viewsets]: api-guide/viewsets.md
[routers]: api-guide/routers.md
[parsers]: api-guide/parsers.md
[renderers]: api-guide/renderers.md
[serializers]: api-guide/serializers.md
[fields]: api-guide/fields.md
[relations]: api-guide/relations.md
[validators]: api-guide/validators.md
[authentication]: api-guide/authentication.md
[permissions]: api-guide/permissions.md
[throttling]: api-guide/throttling.md
[filtering]: api-guide/filtering.md
[pagination]: api-guide/pagination.md
[versioning]: api-guide/versioning.md
[contentnegotiation]: api-guide/content-negotiation.md
[metadata]: api-guide/metadata.md
[schemas]: api-guide/schemas.md
[formatsuffixes]: api-guide/format-suffixes.md
[reverse]: api-guide/reverse.md
[exceptions]: api-guide/exceptions.md
[status]: api-guide/status-codes.md
[testing]: api-guide/testing.md
[settings]: api-guide/settings.md

[documenting-your-api]: topics/documenting-your-api.md
[api-clients]: topics/api-clients.md
[internationalization]: topics/internationalization.md
[ajax-csrf-cors]: topics/ajax-csrf-cors.md
[html-and-forms]: topics/html-and-forms.md
[browser-enhancements]: topics/browser-enhancements.md
[browsableapi]: topics/browsable-api.md
[rest-hypermedia-hateoas]: topics/rest-hypermedia-hateoas.md
[contributing]: topics/contributing.md
[project-management]: topics/project-management.md
[third-party-packages]: topics/third-party-packages.md
[tutorials-and-resources]: topics/tutorials-and-resources.md
[3.0-announcement]: topics/3.0-announcement.md
[3.1-announcement]: topics/3.1-announcement.md
[3.2-announcement]: topics/3.2-announcement.md
[3.3-announcement]: topics/3.3-announcement.md
[3.4-announcement]: topics/3.4-announcement.md
[3.5-announcement]: topics/3.5-announcement.md
[3.6-announcement]: topics/3.6-announcement.md
[3.7-announcement]: topics/3.7-announcement.md
[kickstarter-announcement]: topics/kickstarter-announcement.md
[mozilla-grant]: topics/mozilla-grant.md
[funding]: topics/funding.md
[release-notes]: topics/release-notes.md
[jobs]: topics/jobs.md

[group]: https://groups.google.com/forum/?fromgroups#!forum/django-rest-framework
[botbot]: https://botbot.me/freenode/restframework/
[stack-overflow]: https://stackoverflow.com/
[django-rest-framework-tag]: https://stackoverflow.com/questions/tagged/django-rest-framework
[security-mail]: mailto:rest-framework-security@googlegroups.com
[twitter]: https://twitter.com/_tomchristie