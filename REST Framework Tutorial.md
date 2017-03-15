# REST Framework Tutorial

원문 - [django-rest-framework.org](http://www.django-rest-framework.org/tutorial/1-serialization/)

## 튜토리얼 1: Serialization

### 새 가상환경 만들기

```
pyenv virtualenv env
```
> 가상환경에 대해 더 자세한 내용은 [virtualenv 문서](https://virtualenv.pypa.io/en/latest/index.html)를 참조하세요.

##### 가상환경 설정

```
pip install django  
pip install djangorestframework  
pip install pygments  # 코드 하일라이팅에 사용할 패키지입니다
```

### 새 프로젝트 시작하기

```
django-admin.py startproject tutorial  
cd tutorial 
```

##### 웹 API를 위한 앱 생성

```
python manage.py startapp snippets 
```

settings.py 에 앱 추가

```
# tutorial - settings.py

INSTALLED_APPS = (  
    ...
    'rest_framework',
    'snippets',
)
```
##### URL 설정
tutorial - urls.py

```
url(r'^', include('snippets.urls')),
```
### 모델 만들기
튜토리얼에서 간단히 사용할 Snippet 모델을 만듭니다. 이 모델은 코드 조각들이 저장됩니다.  

```
# snippets/models.py

from django.db import models  
from pygments.lexers import get_all_lexers  
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]  
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])  
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):  
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```
snippet 모델을 초기화할 마이그레이션을 만들어야 하고, 게이터 베이스에 처음으로 싱크도 해야합니다.

```
python manage.py makemigrations snippets  
python manage.py migrate 
```
> 기본 url 설정에서 `snippet`의 url을 include 하기 때문에 snippet에 `urls.py를 생성해 주어야 마이그레이션이 가능합니다.

### 시리얼라이저 클래스 만들기

웹 API를 만들려면 우선 Snippet클래스의 인스턴스를 `json` 같은 형태로 직렬화(serializing)하거나 반직렬화(deserializing)할 수 있어야 합니다.  
Django REST 프레임워크에서는 Django 폼과 비슷한 방식으로 시리얼라이저를 작성합니다. 

```
# snippets/serializers.py

from django.forms import widgets  
from rest_framework import serializers  
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):  
    pk = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        검증한 데이터로 새 `Snippet` 인스턴스를 생성하여 리턴합니다.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        검증한 데이터로 기존 `Snippet` 인스턴스를 업데이트한 후 리턴합니다.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```
시리얼라이저 클래스의 윗부분에서는 직렬화/반직렬화될 필드를 선언했습니다.  
`create()`메서드와 `update()`메서드에서는 `serializer.save()`가 호출되었을 때 인스턴스가 생성 혹은 수정되는 과정을 전부 명시하고 있습니다.

시리얼라이저 클래스는 Django의 `form`클래스와 매우 비슷하고, `required`,`max_length`,`default` 같이 `form`클래스에서 사용하던 값 검증을 위한 옵션도 필도에 지정할 수 있습니다.

이러한 옵션들을 통해 특정상황(HTML로 렌더링한다든지)에 시리얼라이저가 어떻게 작동해야 하는지를 명시할 수 있습니다.  

`{'base_template': 'textarea.html'}` 부분은 Django `form`의 `widget=widgets.Textarea`와 같습니다.  
`ModelSerializer`클래스를 사용하면 이러한 기능을 일일이 구현하지 않아도 되지만 튜토리얼이니까...!

### 시리얼라이저 사용하기git
##### Django shell 띄우기
```
python manage.py shell 
```
필요한 패키지들을 import하고, 코드조각(snippet 클래스의 인스턴스)를 만들어 봅니다.

```
from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer  
from rest_framework.renderers import JSONRenderer  
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')  
snippet.save()

snippet = Snippet(code='print "hello, world"\n')  
snippet.save() 
```
snippet 인스턴스가 만들어졌으니, 이 이슨턴스들 중 하나를 직렬화해봅니다.

```
serializer = SnippetSerializer(snippet)  
serializer.data
{'style': 'friendly', 'code': 'print "hello, world"\n', 'language': 'python', 'pk': 2, 'title': '', 'linenos': False}
```
모델 인스턴스를 파이썬의 데이터 타입으로 변환했습니다. 직렬화 과정을 마무리하려면 이 데이터를 `json`으로 변환해야 합니다.

```
content = JSONRenderer().render(serializer.data)  
content
b'{"pk":2,"title":"","code":"print \\"hello, world\\"\\n","linenos":false,"language":"python","style":"friendly"}'
```
반직렬화도 비슷합니다.  
먼저, 파이썬 데이터 타입을 파싱합니다.

```
from django.utils.six import BytesIO

stream = BytesIO(content)  
data = JSONParser().parse(stream)
```
이 데이터를 인스턴스화합니다.

```
serializer = SnippetSerializer(data=data)  
serializer.is_valid()  
# True
serializer.validated_data  
# OrderedDict([('title', ''), ('code', 'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()  
# <Snippet: Snippet object>
```
뷰를 작성할 뿐만 아니라 시리얼라이저를 사용하는 방식도 `form`을 다루는 방식과 유사합니다.  

모델의 인스턴스뿐만 아니라 쿼리셋도 직렬화할 수 있습니다. 시리얼라이저의 인자에 `many=True`만 추가하면 됩니다.

```
serializer = SnippetSerializer(Snippet.objects.all(), many=True)  
serializer.data
[OrderedDict([('pk', 1), ('title', ''), ('code', 'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('pk', 2), ('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('pk', 3), ('title', ''), ('code', 'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```

### ModelSerializers 사용하기
`SnippetSerializer`클래스는 `snippet`모델의 정보들을 그대로 복사합니다.
Django에서 `form`클래스와 `ModelForm`클래스를 제공하듯이, REST 프레임워크에서도 `Serializer`클래스와 `ModelSerializer` 클래스를 제공합니다.

#### 앞에서 만든 시리얼라이저가 `ModelSerializer`클래스를 사용하도록 리팩터링 합니다.

```
# snippets - serializers.py
# SnippetSerializer 클래스 수정

class SnippetSerializer(serializers.ModelSerializer):  
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```
이렇게 시리얼라이저에 프로퍼티 하나만 정의한 후 시리얼라이저 인스턴스를 출력해보면 모든 필드를 확인할 수 있습니다.

```
# python manage.py shell

>>> from snippets.serializers import SnippetSerializer
>>> serializer = SnippetSerializer()
>>> print(repr(serializer))
SnippetSerializer():  
    id = IntegerField(label='ID', read_only=True)
    title = CharField(allow_blank=True, max_length=100, required=False)
    code = CharField(style={'base_template': 'textarea.html'})
    linenos = BooleanField(required=False)
    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```
`ModelSerializer`클래스는 시리얼라이저 클래스의 단축버전일 뿐입니다.
> - 필드를 자동으로 인식한다.  
> - `create()`메서드와 `update()`메서드가 이미 구현되어 있다.

### 시리얼라이저를 사용하는 Django 뷰 만들기
앞에서 새로 만든 시리얼라이저 클래스를 뷰에서 어떻게 사용할 수 있는지 살펴보겠습니다.  
지금 당장은 REST 프레임워크의 기능을 사용하지 않고, 일반적인 Django 뷰의 형태로 만듭니다.

HttpResponse의 하위 클래스를 만들고, 받은 데이터를 모두 `json`형태로 반환합니다.

```
#snippets/views.py

from django.http import HttpResponse  
from django.views.decorators.csrf import csrf_exempt  
from rest_framework.renderers import JSONRenderer  
from rest_framework.parsers import JSONParser  
from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer

class JSONResponse(HttpResponse):  
    """
    콘텐츠를 JSON으로 변환한 후 HttpResponse 형태로 반환합니다.
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)
```
우리가 만들 API의 최상단에서는 저장된 코드 조각을 모두 보여주며, 새 코드 조각을 만들 수도 있습니다.

```
@csrf_exempt
def snippet_list(request):  
    """
    코드 조각을 모두 보여주거나 새 코드 조각을 만듭니다.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)
```
> JSONResponse 클래스와 동등한 위치에 있어야한다

인증되지 않은 사용자도 이뷰에 POST를 할 수 있도록 `csrf_exempt` 데코레이터를 적어둔 점을 눈여겨 보세요.  
이는 보통의 경우 필요 없을 수도 있고, REST 프레임워크의 뷰가 이보다 더 정밀한 속성들을 제공하기도 하지만, 일단 여기서는 우리가 구현하고 싶은 기능을 `csrf_exempt`가 잘 담당하고 있습니다.

이제 코드 조각 하나를 보여줄 뷰도 필요합니다. 또 이 코드 조각을 업데이트하거나 삭제할 수도 있어야 합니다.

```
@csrf_exempt
def snippet_detail(request, pk):  
    """
    코드 조각 조회, 업데이트, 삭제
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JSONResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data)
        return JSONResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```
> JSONResponse 클래스와 동등한 위치에 있어야한다


마지막으로 이뷰들과 URL을 연결합니다.

```
# snippets - urls.py

from django.conf.urls import url
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```

`json`의 내용이 깨졌거나 뷰가 처리 할 수 없는 메서드 요청인 경우, '500 서버 오류'를 보게 될 것입니다.

### 첫 번째 웹 API 테스트하기
코드 조각을 보여주는 서버를 구동해 봅니다.

셸 종료

```
quit()
```

Django의 개발 서버 작동

```
python manage.py runserver
```
다른 터미널 창에서 버서를 테스트 합시다. 테스트에는 curl이나 httpie를 사용할 수 있습니다. Httpie는 파이썬으로 작성된 사용자 친화적인 http클라이언트입니다.

```
pip install httpie
```

마지막으로 코드 조각 전체를 가져옵니다.

```
# 인터프린터에서

http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK  
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```
id를 지정하여 트정 코드 조각만 가져올 수도 있습니다.

```
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK  
...
{
  "id": 2,
  "title": "",
  "code": "print \"hello, world\"\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```

웹 브라우저에서도 똑같은 json 데이터를 확인 하 수 있습니다.

## 튜토리얼 2: 요청과 응답
REST 프레임워크의 진정한 핵심부를 다룹니다.

### 요청(Request) 객체
REST 프레임워크의 `Request` 객체는 `HttpRequest` 객체를 확장하여 좀더 유연하게 요청을 파싱합니다. `Request` 객체의 핵심부는 `Request.data` 속성입니다. 이 속성은 `Request.POST`와 비슷하지만 웹 API에 좀더 적합합니다.

```
request.POST  # 폼 데이터만 다루며, 'POST' 메서드에서만 사용 가능  
request.data  # 아무 데이터나 다룰 수 있고, 'POST'뿐만 아니라 'PUT'과 'PATCH' 메서드에서도 사용 가능  
```

### 응답(Response) 객체
REST 프레임워크에는 `Response`객체도 존재합니다. 이 객체는 `TemplateResponse`타입이며, 렌더링되지 않은 콘텐츠를 불러와 클라이언트에게 리턴할 콘텐츠 형태로 변환합니다.

```
return Response(data)  # 클라이언트가 요청한 형태로 콘텐트를 렌더링함
```
### 상태 코드
앞에서 만든 뷰에서 숫자 형태의 HTTP 상태 코드를 사용하는 경우, 읽기에도 어렵고 오류가 있더라도 발견하기 어렵습니다.  

REST 프레임워크에서는 각 상태 코드에 대해 좀더 명확한 식별자를 제공합니다.  

예를 들어, `status` 모듈의 `HTTP_400_BAD_REQUEST` 같은 식별자가 있습니다.숫자로 된 식별자를 사용하기보다는 문자 형태의 식별자를 사용하세요.

### API 뷰 감싸기
REST 프레임워크는 API 뷰를 작성할 수 있는 두가지 래퍼를 제공합니다.

1. `@api_view` 데코레이터를 함수 기반 뷰에서 사용할 수 있습니다.
2. `APIView`클래스는 클래스 기반 뷰에서 사용할 수 있습니다.

이 래퍼들은 뷰에서 받은 `Request`에 몇몇 기능을 더하거나, 콘텐츠가 잘 변환되도록 `Response`에 특정 context를 추가합니다.

또한 때에 따라 `405 Method Not Allowed`를 반환하거나, `request.data`가 깨진 경우 `ParseError` 예외를 던지는 등의 일도 수행합니다.

### 이 모든 것을 한 군데 모으기
두 요소를 사용하여 뷰를 작성합니다.

```
# snippets/views.py
# JSONResponse 클래스는 삭제  
# 기존 snippet_list 수정

from rest_framework import status  
from rest_framework.decorators import api_view  
from rest_framework.response import Response  
from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):  
    """
    코드 조각을 모두 보여주거나 새 코드 조각을 만듭니다.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
> 기존 snippet_detail 수정

뷰가 조금 개선되었습니다.
조금 간단해지면서 폼 API와 유사하다는 느낌을 줍니다. 또한 이름 형태의 상태 코드를 사용하여 의미를 명확히 했습니다.

이제 코드 조각 하나를 담당하는 뷰를 수정합니다.

```
@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):  
    """
    코드 조각 조회, 업데이트, 삭제
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

여기서 특정 콘텐츠 형태에 대한 요청이나 응답을 명시적으로 연결하지 않았음을 주목하세요. `request.data`는 `json`요청 뿐만 아니라 `yaml`과 같은 다른 포맷도 다룰 수 있습니다. 응답 객체에 데이터를 담아 리턴하는 것과 비슷하면서도, REST 프레임워크에서는 우리가 원하는 형태로 응답객체를 렌더링 해 줍니다.

### URL의 접미어를 통해 다른 포맷 제공하기

하나의 콘텐츠 형태에 묶여 있지 않다는 응답 객체의 장점을 활용하기 위해, 우리 API에서도 여러 형태의 포맷을 제공해야 합니다. 포맷의 접미어를 URL 형태로 전달받으려면, 다음과 같은 URL을 다룰 수 있어야 합니다.

```
http://example.com/api/item/1.json
```
우선 `format`키워드를 두 가지 뷰에 추가해 봅시다.

```
# snippets/views.py

def snippet_list(request, format=None):  

...

def snippet_detail(request, pk, format=None):
```
`urls.py`를 조금 수정하겠습니다. 기존 URL에 `format_suffix_patterns`라는 패턴을 추가합니다.

```
# snippets/urls.py

from django.conf.urls import patterns, url  
from rest_framework.urlpatterns import format_suffix_patterns  
from snippets import views

urlpatterns = [  
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```
이 외에 더 수정한 부분은 없는데도, 코드는 간명해졌고, 사용자는 자신이 원하는 형태의 포맷을 전달 받을 수 있습니다.

### 어떻게 되었을까?
인터프린터에서 API를 테스트 해봅시다.  
앞에서 했던 것과 비슷하게 작동하지만, 이번에는 잘못된 요청에도 잘 대응합니다.

전체 코드 조각 목록을 받아봅시다.

```
http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK  
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```

이제는 `Accept`헤더를 사용하여 응답 받을 데이터의 포맷도 지정할 수 있습니다.

```
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON  
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML   
```

포맷 접미어를 붙여서 지정 할 수도 있습니다.

```
http http://127.0.0.1:8000/snippets.json  # JSON suffix  
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix  
```

`Content-Type` 헤더를 사용해서 데이터의 포맷을 지정 할 수도 있습니다.

```
# 데이터를 넘기면서 POST 요청
http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

{
  "id": 3,
  "title": "",
  "code": "print 123",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# JSON으로 POST 요청
http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

{
    "id": 4,
    "title": "",
    "code": "print 456",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```
`http://127.0.0.1:8000/snippets/` 브라우저에서도 확인 가능합니다.

### 탐색 가능한 API
API는 클라이언트의 요청에 따라 데이터의 포맷을 결정하여 응답합니다. 따라서 웹 브라우저의 요청에 대해서는 기본적으로 HTML 형태로 응답해주게 됩니다. 이 덕분에 API를 웹 브라우저에서 탐색할 수 있습니다.

브라우저에서 탐색 가능함은 사용성 면에서 굉장히 유용하여, API를 더 쉽게 개발하고 사용하도록 도와줍니다. 또한 다른 개발자들이 API를 파악하고 사용할 때의 진입장벽을 획기적으로 낮춰 줍니다.

## 튜토리얼 3: 클래스 기반 뷰
앞서 함수 기반으로 만들었던 API 뷰를 클래스 기반 뷰로도 만들 수 있습니다. 이는 일반적인 기능을 재사용하게 해주며, 코드중복(DRY)도 막아주기 때문에 굉장히 쓸모 있는 패턴입니다.

### 클래스 기반 뷰로 API 재작성하기
먼저 최상단 뷰를 클래스 기반 뷰로 재작성해 봅시다.

이를 위해 `views.py` 를 약간 리팩터링해야 합니다.

```
# snippets/views.py

from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer  
from django.http import Http404  
from rest_framework.views import APIView  
from rest_framework.response import Response  
from rest_framework import status


class SnippetList(APIView):  
    """
    코드 조각을 모두 보여주거나 새 코드 조각을 만듭니다.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```
이전 코드와 거의 똑같이 보이지만 HTTP 메서드를 분리했기 때문에 좀더 좋습니다. 마찬가지로 `views.py`에서 코드조각 하나를 담당하는 뷰도 수정합니다.

```
class SnippetDetail(APIView):  
    """
    코드 조각 조회, 업데이트, 삭제
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```
아직까진 함수 기반 뷰와 크게 다르지 않습니다.
`urls.py`도 수정해봅시다. 클래스 기반 뷰를 사용해야 하니까요.
 
```
# snippets - urls.py
from django.conf.urls import url  
from rest_framework.urlpatterns import format_suffix_patterns  
from snippets import views

urlpatterns = [  
    url(r'^snippets/$', views.SnippetList.as_view()),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)  
```
끝났습니다. 개발 서버를 실행해 보면 이전과 동일하게 작동하는 모습을 확인 할수 있습니다.

### 믹스인 사용하기
클래스 기반 뷰를 사용하는 큰 이점은 기능들을 손쉽게 조합할 수 있다는 점입니다.

지금까지 사용한 생성/조회/업데이트/삭제 등의 명령은 일반적으로 모델을 사용 할 때의 뷰와 비슷합니다.이러한 보편적인 기능을 REST 프레임워크에서는 믹스인 클래스로 구현해두었습니다.

이제 뷰에 믹스인 클래스를 추가해봅시다.
`views.py`을 수정합니다.

```
# snippet/views.py

from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer  
from rest_framework import mixins  
from rest_framework import generics

class SnippetList(mixins.ListModelMixin,  
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```
`GenericAPIView`와 `ListModelMixin`, `CreateModelMixin`을 사용하여 뷰를 만들었습니다.  
기본 뷰(`GenericAPIView`)는 핵심 기능을 제공하며 믹스인 클래스들은 `.list()`나 `.create()` 기능을 제공합니다. 여기서는 이 기능들을 `get`과 `post` 메서드에 적절히 연결하였습니다.

```
class SnippetDetail(mixins.RetrieveModelMixin,  
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```
비슷하게, 여기서도 `GenericAPIView`는 핵심 기능을 제공하며, 나머지 믹스인들이 `.retrieve()`, `.update()`, `.destroy()` 기능을 제공합니다.

### 제네릭 클래스 기반 뷰 사용하기
믹스인 클래스를 사용하여 뷰의 코드를 꽤 많이 줄였지만, 더 줄일 수 있습니다. REST 프레임워크에서는 믹스인과 연결된 제네릭 뷰를 제공합니다. 이를 사용하면 `views.py`파일이 굉장히 짧아집니다.

```
# snippets/views.py

from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer  
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):  
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):  
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```
정말 간결해졌죠? 별 노력 없이 아주 많은 기능을 구현했는데도 코드는 더 깔끔하고 훨씬 Django다워졌습니다.

## 튜토리얼 4: 인증과 권한
지금까지 우리가 만든 API에서는 누구라도 코드 조각을 편집, 삭제를 할 수 있습니다. 아무런 제한이 없습니다. 여기에 다음과 같은 고급 기능을 추가합니다.

- 코드 조각은 만든 사람과 연관이 있다.
- 인증받은 사용자만 코드 조각을 만들 수 있다.
- 해당 코드 조각을 만든 사람만, 이를 편집, 삭제할 수 있다.
- 인증받지 않은 사용자는 '읽기 전용'으로만 사용 가능하다.

### 모델에 속성 추가하기
`Snippet` 모델을 조금 수정합니다.

필를 두개 추가합니다. 하나(owner)는 코드 조각을 만든 사람을 가리킵니다. 다른 하나는 하이라이트 된 코드를 HTML 형태로 저장하는데 사용됩니다.

```
# snippet/models.py - Snippet

owner = models.ForeignKey('auth.User', related_name='snippets')  
highlighted = models.TextField()  
```
그리고 모델이 저장될 때 하이라이트된 코드를 highlight 필드에 저장해야 합니다. 코드 하이라이팅에는 `pygments` 라이브러리를 사용합니다.

필요한 라이브러리들을 임포트합니다.

```
from pygments.lexers import get_lexer_by_name  
from pygments.formatters.html import HtmlFormatter  
from pygments import highlight
```

모델에 `.save()`메서드를 작성합니다.

```
def save(self, *args, **kwargs):  
    """
    `pygments` 라이브러리를 사용하여 하이라이트된 코드를 만든다.
    """
    lexer = get_lexer_by_name(self.language)
    linenos = self.linenos and 'table' or False
    options = self.title and {'title': self.title} or {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```

코드를 모두 작성하면 데이터베이스를 업데이트 해야합니다.
보통은 데이터베이스 마이그레이션을 작성하지만, 튜토리얼일 뿐이니 데이터를 지우고 새로 만들겠습니다.

```
rm -f tmp.db db.sqlite3  
rm -r snippets/migrations  
python manage.py makemigrations snippets  
python manage.py migrate 
```

API를 테스트하기 위해 사용자 계정을 만듭니다.

```
python manage.py createsuperuser  
```

### 사용자 모델에 엔드포인트 추가하기
사용자를 추가하였으니 사용자를 보여주는 API도 추가합니다.  

`serializers.py` 파일에 새 시리얼라이저를 작성합니다.

```
# snippets/serializers.py(추가)

from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):  
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'snippets')
```
`snippets`는 사용자 모델과 **반대 방향**으로 이어져 있기 때문에 `ModelSerializer`에 기본적으로 추가되지 않습니다. 따라서 명시적으로 필드를 지정해주었습니다.

사용자와 관련된 뷰도 추가해봅시다. 읽기 전용 뷰만 있으면 되니까, 제네릭 클래스 기반 뷰 중에서 `ListAPIView`와 `RetrieveAPIView`를 사용합시다.

```
# snippet/views.py

from django.contrib.auth.models import User
from snippets.serializers import UserSerializer

class UserList(generics.ListAPIView):  
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):  
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
마지막으로 뷰에 URL을 연결합니다.

```
# snippets - urls.py

url(r'^users/$', views.UserList.as_view()),  
url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()), 
```

### 사용자가 만든 코드 조각 연결하기
지금까지 코드 조각을 만들었지만, 해당 코드 조각을 만든 사용자와 아무 관계도 맺지 않았습니다. 사용자는 직렬화된 표현에 나타나지 않았고, 요청하는 측에서 지정하는 속성이었습 뿐입니다.

이를 해결하기 위해 코드 조각 뷰에서 `.perform_create()`메서드를 오버라이딩합니다. 이 메서드는 인스턴스를 저정하는 과정을 조정하며, 따라서 요청이나 요청 URL에서 정보를 가져와 원하는 대로 다룰 수 있습니다.

`SnippetList` 뷰 클래스에 내용을 추가합니다.

```
# snippets/views.py / SnippetList 클래스에 추가

def perform_create(self, serializer):  
    serializer.save(owner=self.request.user)
```
우리가 만든 시리얼라이저의 `create()`메서드는 검증한 요청 데이터에 더하여 `owner`필드도 전달합니다.

### 시리얼라이저 업데이트하기
이제 코드 조각이, 해당 코드 조각을 작성한 사용자와 연결되었습니다. `SnippetSerializer`에도 반영합시다.

```
# serializers.py - SnippetSerializer에 추가

owner = serializers.ReadOnlyField(source='owner.username') 
```
> Meta클래스의 필드 목록에도 `owner`를 추가해야 합니다.

이 필드에는 조금 재미있는 점이 있습니다. `source`인자로는 특정 필드를 지정할 수 있습니다. 여기에는 직렬화된 인스턴스의 속성 뿐만 아니라 위의 코드에서처럼 마침표 표기 방식을 통해 속성을 탐색할 수도 있습니다. 마치 Django의 템플릿 언어와 비슷합니다.

이 필드는 `CharField`나 `BooleanField`와는 달리 타입이 없는 `ReadOnlyField`클래스로 지정했습니다. 타입이 없는 `ReadOnlyField`는 직렬화에 사용되었을 땐 언제나 읽기 전용이므로, 모델의 인스턴스를 업데이트 할 때는 사용할 수 없습니다. `CharField(read_only=True)`도 이와 같은 기능을 수행합니다.

### 뷰에 요청 권한 추가하기
이렇게해서 코드 조각이 사용자와 연결되었습니다. 이제 인증받은 사용자만 코드 조각을 생성/업데이트/삭제 해봅시다.

REST 프레임워크는 특정 뷰에 제한을 걸 수 있는 권한 클래스를 제공하고 있습니다. 그중 한가지인 `IsAuthenticatedOrReadOnly`는 인증 받은 요청에 읽기와 쓰기 권한을 부여하고, 인증받지 않은 요청에 대해서는 읽기 권한만 부여합니다.

뷰 파일에 다음 내용을 추가합니다.

```
# views.py

from rest_framework import permissions
```

`SnippetList` 클래스와 `SnippetDetail` 클래스에 모두 다음 속성을 추가합니다.

```
permission_classes = (permissions.IsAuthenticatedOrReadOnly,)
```

### 탐색 가능한 API에 로그인 추가하기
지금 시점에 브라우저에서 API에 접속해보면 더이상 새 코드 조각을 만들 수 없을겁니다. 이를 해결하려면 사용자 로그인 기능이 필요합니다.

URL 설정 파일인 `urls.py`를 수정하면 탐색 가능한 API에 사용할 로그인 뷰를 추가할 수 있습니다.

```
# snippets/urls.py
from django.conf.urls import include 
```

파일의 끝부분에 다음 내용을 추가합니다. 탐색 가능한 API의 로그인 뷰와 로그아욱 뷰에 사용되는 url패턴입니다.

```
urlpatterns += [  
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
]
```
url 패턴에서 `r'^api-auth/'`부분은 우리가 사용하고 싶은 URL을 나타냅니다. 여기에는 한가지 제약만 따르는데, namespace에 `'rest_framework'`를 지정해야 한다는 점입니다.

다시 브라우저로 API에 접근해보면 오른쪽 상단에 `Login' 링크가 보일 겁니다. 이제 앞에서 만들었던 사용자로 로그인하면 코드조각을 만들 수 있습니다.

코드 조각을 몇개 만들고 '/users/'에 가보세요. 해당 사용자가 만든 코드 조각 목록이 'snippets' 필드에 포함된 것을 확인 할 수 있습니다.

### 객체 수준에서 권한 설정하기
코드 조각은 아무나 볼 수 있어야 하지만, 업데이트,삭제는 해당 코드를 만든 사용자만 할 수 있어야 합니다.

이를 위해 커스텀 권한을 만듭니다.

```
# snippets/permissions.py 생성

from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):  
    """
    객체의 소유자에게만 쓰기를 허용하는 커스텀 권한
    """

    def has_object_permission(self, request, view, obj):
        # 읽기 권한은 모두에게 허용하므로,
        # GET, HEAD, OPTIONS 요청은 항상 허용함
        if request.method in permissions.SAFE_METHODS:
            return True

        # 쓰기 권한은 코드 조각의 소유자에게만 부여함
        return obj.owner == request.user
```
커스텀 권한을 코드 조각 인스턴스에 추가합니다.

```
# views.py -SnippetDetail 클래스에 permission_classes 속성을 추가합니다.

permission_classes = (permissions.IsAuthenticatedOrReadOnly,  
                      IsOwnerOrReadOnly,)
```

`IsOwnerOrReadOnly` 클래스도 임포트합니다.

```
from snippets.permissions import IsOwnerOrReadOnly 
```
다시 브라우저로 돌아가면, 코드 조각에 대한 'DELETE'와 'PUT' 기능은 사용자에게만 나타날겁니다.

### API에 인증 붙이기
API에 권한 설정을 했으므로, 이제는 코드 조각을 수정할 수 있는 인증 절차가 필요합니다.  
지금까지는 [인증 클래스](http://www.django-rest-framework.org/api-guide/authentication/)를 만들지 않고 기본으로 제공되는 `SessionAuthentication`과 `BasicAuthentication`을 사용했습니다.  

웹 브라우저로 API를 사용하는 경우, 로그이니 하면 브라우저의 세션에 인증 정보가 저장됩니다.

프로그램 상에서 API를 사용하는 경우, 인증에 필요한 내용을 명시적으로 전달해야만 합니다.

인증 없이 코드 조각을 생성하려는 경우, 다음과 같은 에러를 보여줍니다.

```
# 인터프린터 오류메세지

http POST http://127.0.0.1:8000/snippets/ code="print 123"

{
    "detail": "Authentication credentials were not provided."
}
```

사용자 계정과 비밀번호를 포함하여 요청하면, 요청은 성공합니다.

```
http -a tom:password POST http://127.0.0.1:8000/snippets/ code="print 789"

{
    "id": 5,
    "owner": "tom",
    "title": "foo",
    "code": "print 789",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```
이렇게 웹 API 위에 권한들이 잘 설정되었고, 사용자의 코드 조각에 대한 엔드 포인트도 완성되었습니다.

## 튜토리얼 5: 관계 & 하이퍼링크 API
우리가 만든 API에서 '관계'는 주 키(Primary key)로 나타나고 있습니다. 이번에는 API의 발견성(discoverability)와 응집력(cohesion)을 향상시키고자 관계를 하이퍼링크로 나타내보겠습니다.

### API의 최상단에 대한 엔드 포인트 만들기
지금까지 '코드조각'과 '사용자'에 대한 엔드 포인트를 만들었지만, API의 시작점은 없었습니다. 이를 만들기 위해 평범한 함수 기반 뷰와 `@api_view` 데코레이터를 사용하겠습니다.

```
# snippets - views.py

from rest_framework.decorators import api_view  
from rest_framework.response import Response  
from rest_framework.reverse import reverse


@api_view(('GET',))
def api_root(request, format=None):  
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```
여기서 URL을 만드는데 `reverse`함수를 사용한 점을 주목하세요.

### 코드 조각의 하이라이트 버전에 대한 엔드 포인트 만들기
API에서 아직까지 만들지 않은 부분은 바로, 코드 조각의 하이라이트 버전을 볼 수 있는 방법입니다.

API의 다른 부분과는 달리 이번엔 JSON 대신 HTML형태로 나타내겠습니다. REST 프레임워크에서 HTML로 렌더링하는 방식은 두 가지 정도가 있는데, 하나는 쳄플릿을 사용하는 것이고, 다른 하나는 미리 렌더링된 HTML을 사용하는 것입니다.

하이라이트된 코드 조각을 보여주려고 할 때 주의해야 할 점은, 우리가 사용 할 만한 제네릭 뷰가 없다는 것입니다. 오브젝트 자체가 아니라, 오브젝트의 속성 하나를 반환할 것이기 때문입니다.

제네릭 뷰 대신 평범한 클래스를 사용하고, `.get()`메서드를 구현하겠습니다.

```
# snippets/views.py

from rest_framework import renderers  
from rest_framework.response import Response

class SnippetHighlight(generics.GenericAPIView):  
    queryset = Snippet.objects.all()
    renderer_classes = (renderers.StaticHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```
새 뷰는 URL설정에서고 연결해야 합니다.
최상단 url을 방금 만든 뷰로 연결합니다.

```
# snippets/urls.py

url(r'^$', views.api_root),  
```
하이라이트된 코드 조각을 술 수 있는 url에 대한 패턴도 추가합니다.

```
url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),  
```

### 하이퍼링크로 API 연결하기
요소들 사이의 관게를 다루는 일은 웹 API 설계에서 또 하나의 도전 과제입니다. 관계를 표현하는 방법은 다양합니다.

- 주 키(Primary key)
- 하이퍼링크
- 관계 요소의 식별 가능한 슬러그(slug) 필드
- 관계 요소의 기본 문자열 표현
- 포함된 관계 요소에 대한 표현
- 이 외에도 사용자화된 표현

REST 프레임워크에서는 모든 방법을 지원합니다. 관계 혹은 역관계에 적용하거나, 제네릭 외부 키(foreign key)처럼 사용자화된 manager에 적용할 수도 있습니다.

여기서는 하이퍼링크 방식을 채택하겠습니다. 이렇게 하려면 기존에 사용했던 `ModelSerializer`를 `HyperlinkedModelSerializer`으로 변경해야 합니다.

`HyperlinkedModelSerializer`는 다음과 같은 점들이 다릅니다.

- `pk` 필드는 기본 요소가 아닙니다.
- `HyperlinkedIdentityField`를 사용하는 `url`필드가 포함되어 있습니다.
- 관계는 `PrimaryKeyRelatedField` 대신 `HyperlinkedRelatedField`를 사용하여 나타냅니다.

기존 시리얼라이저에 하이퍼링크를 추가하시는 쉽습니다.

```
# snippets/serializers.py

class SnippetSerializer(serializers.HyperlinkedModelSerializer):  
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ('url', 'highlight', 'owner',
                  'title', 'code', 'linenos', 'language', 'style')


class UserSerializer(serializers.HyperlinkedModelSerializer):  
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ('url', 'username', 'snippets')
```
이 코드에서는 새롭게 `highlight`필드가 추가되었습니다. 이 필드는 `url`필드와 같은 타입이며, `snippet-detail`url 패턴 대신 `snippet-highlight` url패턴을 가리킵니다.

앞에서 URL의 format접미어로 `.json`을 붙였듯이, `highlight`필드에는 format 접미어로 `'.html'` 을 붙였습니다.

### URL 패턴에 이름 붙이기
하이퍼링크 API를 만들고 싶다면, URL 패턴에 이름을 붙여야 합니다. 어떤 패턴들인지 살펴봅시다.

- API의 최상단은 `user-list`와 `snippet-list`를 가리킵니다.
- 코드조각 시리얼라이저에는 `snippet-highlight`를 가리키는 필드가 존재합니다.
- 사용자 시리얼라이저에는 `snippet-detail`을 가리키는 필드가 존재합니다.
- 코드조각 시리얼라이저와 사용자 시리얼라이저에는 `url`필드가 존재합니다. 이 필드는 기본적으로 `'{모델_이름}-detail'`을 가리키며 따라서 `snippet-detail`과 `user-detail`을 가리킵니다.

이 이름들을 URL 설정에 넣었다면 `snippets/urls.py`은 다음과 같은 것입니다.

```
# snippets/urls.py

from django.conf.urls import url, include

# API endpoints
urlpatterns = format_suffix_patterns([  
    url(r'^$', views.api_root),
    url(r'^snippets/$',
        views.SnippetList.as_view(),
        name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    url(r'^users/$',
        views.UserList.as_view(),
        name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$',
        views.UserDetail.as_view(),
        name='user-detail')
])

# 탐색 가능한 API를 위한 로그인/로그아웃 뷰
urlpatterns += [  
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
]
```

### 페이징 기능 추가하기
사용자나 코드조각의 목록이 꽤 긴 경우들이 있습니다. 결과물을 여러 페이지로 나누어, API 클라이언트 측에서 각 페이지를 하나씩 차례대로 읽어가도록 만들겠습니다.

페이지징 설정의 기본 값을 바꿔 봅니다.

```
# tutorial/settings.py

REST_FRAMEWORK = {  
    'PAGE_SIZE': 10
}
```
REST 프레임워크의 모든 설정은 'REST_FRAMEWORK'라는 딕셔너리에 넣어야합니다. 

필요에 따라 페이징 스타일을 바꿀 수도 있지만 여기서는 기본 스타일을 따르겠습니다.

### 탐색 가능한 API
탐색 가능한 API를 브라우저에서 열어서 링크들을 이리 저리 눌러보면, API의 구석구석을 둘러볼 수 있습니다. 또한 코드조각의 인스턴스의 '하이라이트 버전'을 살펴볼 수고 있습니다.(HTML형태입니다.)

## 튜토리얼 6: 뷰셋 & 라우터
REST 프레임워크는 `ViewSets`이라는 추상 클래스를 제공합니다. 이를 통해 개발자는 API의 상호작용이나 상태별 모델링에 집중할 수 있고, URL 구조는 기본 관례에 따라 자동으로 설정됩니다.

`ViewSet`클래스는 `View`클래스와 거의 비슷하지만, `get`과 `put`메서드는 지원하지 않고 `read`와 `update`메서드를 지원합니다.

`ViewSet`클래스는 따지고 보면, 앞에서 만든 핸들러 메서드가 실제 뷰로 구체화될 때 이를 연결해주기만 합니다. 이때 보통은 `Router`클래스를 사용하여 복잡한 URL설정을 처리합니다.

### 뷰셋을 사용하여 리팩터링하기
지금까지 만든 뷰들을 살펴보면서 뷰셋을 사용해서 리팩터링 해봅시다.

가장 먼저 리팩터링할 뷰는 `UserList`와 `UserDetail` 뷰입니다. `UserViewSet` 하나로 모읍니다. 두 뷰의 코드를 삭제한 다음 아래의 클래스 하나를 입력합니다.

```
# snippets - views.py
# UserList와 UserDetail 삭제

from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):  
    """
    이 뷰셋은 `list`와 `detail` 기능을 자동으로 지원합니다
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
여기서 사용한 `ReadOnlyModelViewSet`클래스는 '읽기 전용' 기능을 자동으로 지원합니다. `queryset`과 `serializer_class` 속성은 여전히 설정을 해야하지만, 두 개의 클래스에 중복으로 설정할 필요는 없어졌습니다.

다음으로 `SnippetList`와 `SnippetDetail`, `SnippetHighlight`뷰를 리팩토링합니다.

```
# SnippetList와 SnippetDetail, SnippetHighlight 삭제

from rest_framework.decorators import detail_route

class SnippetViewSet(viewsets.ModelViewSet):  
    """
    이 뷰셋은 `list`와 `create`, `retrieve`, `update`, 'destroy` 기능을 자동으로 지원합니다

    여기에 `highlight` 기능의 코드만 추가로 작성했습니다
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

    @detail_route(renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
            serializer.save(owner=self.request.user)
```
이번에는 읽기 기능과 쓰기 기능을 모두 지원하는 `ModelViewSet`클래스를 사용했습니다.  

추가한 `highlight`기능에는 여전히 `@detail_router` 데코레이터를 사용했습니다. 이 데코레이터는 `create`나 `update`, `delete`에 해당하지 않는 기능에 대해 사용하면 됩니다.

`@detail_router` 데코레이터를 사용한 기능은 기본적으로 `GET`요청에 응답합니다. `methods` 인자를 설정하면 `POST`요청에도 응답할 수 있습니다.

추가 기능의 URL은 기본적으로 메서드 이름과 같습니다. 이를 변경하고 싶다면 데코레이터에 `url_path`인자를 설정하면 됩니다.

### 뷰셋과 주소를 명시적으로 연결하기
핸들러 메서드는 단지 URL 설정과 연결하는 기능만 담당합니다. 여기서는 먼저 뷰셋의 뷰들을 명시적으로 살펴봅니다.

`urls.py`파일에서 `ViewSet`클래스를 실제 뷰(concrete view)와 연결합니다.

```
# snippets - urls.py

from snippets.views import SnippetViewSet, UserViewSet, api_root  
from rest_framework import renderers

snippet_list = SnippetViewSet.as_view({  
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({  
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({  
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({  
    'get': 'list'
})
user_detail = UserViewSet.as_view({  
    'get': 'retrieve'
})
```
`ViewSet`클래스의 뷰들을HTTP 메서드에 따라 어떻게 실제 뷰와 연결했는지 살펴보세요.

이제 실제 뷰와 URL을 연결합니다.

```
urlpatterns = format_suffix_patterns([  
    url(r'^$', api_root),
    url(r'^snippets/$', snippet_list, name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
    url(r'^users/$', user_list, name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
])
```

### 라우터 사용하기
`View`클래스 대신 `ViewSet`클래스를 사용했기 때문에, 이제는 URL도 설정 할 필요가 없습니다. `Router`클래스를 사용하면 뷰 코드와 뷰, URL이 관례적으로 자동 연결됩니다. 단지 뷰를 라우터에 적절히 등록해주기만 하면 됩니다. 그러면 REST 프레임워크가 알아서 다 합니다.

`urls.py` 파일을 다음과 같이 수정합니다.

```
# snippets/urls.py

from django.conf.urls import url, include  
from snippets import views  
from rest_framework.routers import DefaultRouter

# 라우터를 생성하고 뷰셋을 등록합니다
router = DefaultRouter()  
router.register(r'snippets', views.SnippetViewSet)  
router.register(r'users', views.UserViewSet)

# 이제 API URL을 라우터가 자동으로 인식합니다
# 추가로 탐색 가능한 API를 구현하기 위해 로그인에 사용할 URL은 직접 설정을 했습니다
urlpatterns = [  
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```
라우터에 뷰셋을 등록하는 일은 url 패턴 설정하기와 비슷합니다. 여기서는 두 개를 등록했습니다. 뷰들에 사용할 URL의 접두어와 뷰셋입니다.

`DefaultRouter` 클래스는 API의 최상단 뷰를 자동으로 생성해주므로, `views`모듈에 있는 `api-root` 메서드와 연결했던 URL도 삭제했습니다.

### 뷰? 뷰셋? 장단점 비교하기
뷰셋은 유용한 추상화입니다. API 전반에 걸쳐 일관적인 URL 관례를 구현할 수 있고 작성할 코드 양은 최소한으로 유지할 수 있어서, URL 설정에 낭비될 정성을 API의 상호작용과 표현 자체에 쏟을 수 있습니다.

하지만 이것이 항상 옳다는 뜻은 아닙니다. 클래스 기반 뷰와 함수 뷰에 각각 장단점이 있듯이 말입니다. 뷰셋을 사용하면 명확함이 좀 약해집니다.

정말 적은 양의 코드만으로 pastebin과 같은 웹 API를 구현했습니다 이 API는 웹 브라우저를 완벽히 지원하고, 인증 기능도 있고, 오브젝트로 권한도 설정되며 다양한 형태로 렌더링됩니다.

지금까지 기본 Django 뷰에서 시작하여 기능들을 점진적으로 만드는 설졔 과정을 차근차근 살펴보았습니다.

REST 프레임워크를 개인 공부하기 위해 [raccoony's cave](http://raccoonyy.github.io/)의 튜토리얼을 재정리한 것입니다. 

