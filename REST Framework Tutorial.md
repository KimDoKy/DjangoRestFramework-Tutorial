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
snippets - views.py

두가지 뷰에 `format` 키워드 추가
```
def snippet_list(request, format=None):  
def snippet_detail(request, pk, format=None):
```

snippets - urls.py

```
from django.conf.urls import patterns, url  
from rest_framework.urlpatterns import format_suffix_patterns  
from snippets import views

urlpatterns = [  
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

### 어떻게 되었을까?
인터프린터

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
```
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON  
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML   
```
```
http http://127.0.0.1:8000/snippets.json  # JSON suffix  
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix  
```
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
`http://127.0.0.1:8000/snippets/` 브라우저에서도 확인

### 탐색 가능한 API

## 튜토리얼 3: 클래스 기반 뷰
### 클래스 기반 뷰로 API 재작성하기
snippets - views.py

```
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
 snippets - urls.py
 
```
from django.conf.urls import url  
from rest_framework.urlpatterns import format_suffix_patterns  
from snippets import views

urlpatterns = [  
    url(r'^snippets/$', views.SnippetList.as_view()),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)  
```
### 믹스인 사용하기


snippet - views.py

```
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
### 제네릭 클래스 기반 뷰 사용하기
snippets - views.py

```
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
## 튜토리얼 4: 인증과 권한
### 모델에 속성 추가하기
Snippet - models.py - Snippet

```
owner = models.ForeignKey('auth.User', related_name='snippets')  
highlighted = models.TextField()  
```
```
from pygments.lexers import get_lexer_by_name  
from pygments.formatters.html import HtmlFormatter  
from pygments import highlight
```
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
```
python manage.py createsuperuser  
```
### 사용자 모델에 엔드포인트 추가하기
snippets - serializers.py 소스 추가

```
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):  
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'snippets')
```
snippet - views.py

```
from django.contrib.auth.models import User
from snippets.serializers import UserSerializer

class UserList(generics.ListAPIView):  
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):  
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

snippets - urls.py

```
url(r'^users/$', views.UserList.as_view()),  
url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()), 
```

### 사용자가 만든 코드 조각 연결하기
snippets - views.py / SnippetList 클래스에 추가

```
def perform_create(self, serializer):  
    serializer.save(owner=self.request.user)
```

### 시리얼라이저 업데이트하기
serializers.py의 SnippetSerializer에 추가

```
owner = serializers.ReadOnlyField(source='owner.username') 
```
> Meta클래스의 필드 목록에도 `owner`를 추가해야 합니다.

### 뷰에 요청 권한 추가하기

views.py

```
from rest_framework import permissions
```
SnippetList 클래스와 SnippetDetail 클래스에 모두 다음 속성을 추가합니다.

```
permission_classes = (permissions.IsAuthenticatedOrReadOnly,)
```

### 탐색 가능한 API에 로그인 추가하기

snippets - urls.py

```
from django.conf.urls import include 
```

```
urlpatterns += [  
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
]
```

### 객체 수준에서 권한 설정하기
snippets - permissions.py 생성

```
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
views.py
SnippetDetail 클래스에 permission_classes 속성을 추가합니다.

```
permission_classes = (permissions.IsAuthenticatedOrReadOnly,  
                      IsOwnerOrReadOnly,)
```
```
from snippets.permissions import IsOwnerOrReadOnly 
```
### API에 인증 붙이기

인터프린터 오류메세지
```
http POST http://127.0.0.1:8000/snippets/ code="print 123"

{
    "detail": "Authentication credentials were not provided."
}
```

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
## 튜토리얼 5: 관계 & 하이퍼링크 API

### API의 최상단에 대한 엔드 포인트 만들기
snippets - views.py

```
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

### 코드 조각의 하이라이트 버전에 대한 엔드 포인트 만들기
snippets/views.py

```
from rest_framework import renderers  
from rest_framework.response import Response

class SnippetHighlight(generics.GenericAPIView):  
    queryset = Snippet.objects.all()
    renderer_classes = (renderers.StaticHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```

snippets/urls.py

```
url(r'^$', views.api_root),  
```

```
url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),  
```

### 하이퍼링크로 API 연결하기
snippets/serializers.py

```
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

### URL 패턴에 이름 붙이기

snippets/urls.py

```
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
tutorial/settings.py

```
REST_FRAMEWORK = {  
    'PAGE_SIZE': 10
}
```
### 탐색 가능한 API

## 튜토리얼 6: 뷰셋 & 라우터
### 뷰셋을 사용하여 리팩터링하기
snippets - views.py
UserList와 UserDetail 삭제

```
from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):  
    """
    이 뷰셋은 `list`와 `detail` 기능을 자동으로 지원합니다
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

SnippetList와 SnippetDetail, SnippetHighlight 삭제

```
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

### 뷰셋과 주소를 명시적으로 연결하기
snippets - urls.py

```
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

snippets - urls.py

```
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
### 뷰? 뷰셋? 장단점 비교하기










