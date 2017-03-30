# Django REST Framework - Settings

---

_"Namespaces are one honking great idea - let's do more of those!"_  

_"네임 스페이스는 훌륭한 아이디어를 제공합니다. let's do more of those!"_  
 
_— The Zen of Python_

---

## Settings
REST 프레임워크의 모든 설정은 `REST_FRAMEWORK`라는 단일 Django 설정에 네임 스페이스를 설정합니다.  
예를 들어 프로젝트의 `settings.py`파일에는 다음과 같은 내용이 포함될 수 있습니다.

```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework.renderers.JSONRenderer',
    ),
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework.parsers.JSONParser',
    )
}
```

### Accessing settings
프로젝트에서 REST 프레임워크의 API 설정값에 액서스해야하는 경우 `api_settings`객체를 사용해야합니다. 예를 들면.

```python
from rest_framework.settings import api_settings

print api_settings.DEFAULT_AUTHENTICATION_CLASSES
```
`api_settings`객체는 사용자가 정의한 설정을 확인하고 그렇지 않으면 기본값을 fall back합니다. 클래스를 참조하기 위해 string import path를 사용하여 모든 설정은 문자열 리터럴 대신 참조 된 클래스를 자동으로 가져오고 반환합니다.

---

## API Reference

### API policy settings
다음 설정은 기본 API 정책을 제어하며 모든 `APIView` CBV 또는 @api_view FBV에 적용됩니다.

#### DEFAULT_RENDERER_CLASSES
`Response` 객체를 반환할 때 사용할 수 있는 renderer의 기본 set을 결정하는 rederer클래스의 list 또는 tuple입니다.  

Default:

```python
(
    'rest_framework.renderers.JSONRenderer',
    'rest_framework.renderers.BrowsableAPIRenderer',
)
```
#### DEFAULT_PARSER_CLASSES
`request.data`속성에 액서스 할 때 사용되는 parser의 기본 set을 결정하는 parser 클래스의 list 또는 tuple입니다.

Default:

```python
(
    'rest_framework.parsers.JSONParser',
    'rest_framework.parsers.FormParser',
    'rest_framework.parsers.MultiPartParser'
)
```

#### DEFAULT_AUTHENTICATION_CLASSES
`request.user` 또는 `request.auth`등록 정보에 액서스할 때 사용되는 인증자의 기본 set을 판별하는 authentication 클래스의 list 또는 tuple입니다.  

Default:

```python
(
    'rest_framework.authentication.SessionAuthentication',
    'rest_framework.authentication.BasicAuthentication'
)
```

#### DEFAULT_PERMISSION_CLASSES
view의 시작에 체크 된 권한의 기본 set을 결정하는 permission 클래스의 list 또는 tuple입니다. permission은 list의 모든 클래스에서 부여해야합니 다.  

Default:

```python
(
    'rest_framework.permissions.AllowAny',
)
```

#### DEFAULT_THROTTLE_CLASSES
view의 시작에서 점검되는 기본 throttle set을 결정하는 throttle 클래스의 list 또는 tuple입니다.  

Default: `()`

#### DEFAULT_CONTENT_NEGOTIATION_CLASS
들어오는 request에 따라 rederer가 response에 대해 선택되는 방법을 결정하는 content negotiation 클래스 입니다.  

Default: `'rest_framework.negotiation.DefaultContentNegotiation'`

---

### Generic view settings
다음 설정은 generic CBV의 동작을 제어합니다.  

#### `DEFAULT_PAGINATION_SERIALIZER_CLASS`

---

**이 설정은 제거되었습니다.**  

pagination API는 출력 형식을 결정하기 위해 serializer를 사용하지 않으므로 대신 출력 형식 제어 방법을 지정하기 위해 pagination 클래스의 'get_paginated_response` 메서드를 대체해야합니다.

---

#### `DEFAULT_FILTER_BACKENDS`
generic 필터링에 사용해야 하는 filter backend 클래스 list입니다. `None`으로 설정하면 generic 필터링이 비활성화됩니다.

#### `PAGINATE_BY`

---

**이 설정은 제거 되었습니다.**  

pagination 스타일 설정에 대한 자세한 지침은 [setting the pagination style](http://www.django-rest-framework.org/api-guide/pagination/#modifying-the-pagination-style)를 참조하세요.

---
#### `PAGE_SIZE`
pagination에 사용할 기본 페이지 크기입니다. `None`으로 설정하면 기본적으로 pagination이 비활성화됩니다.  

Default: `None`  

#### `PAGINATE_BY_PARAM`

---
**이 설정은 제거 되었습니다.**  

pagination 스타일 설정에 대한 자세한 지침은 [setting the pagination style](http://www.django-rest-framework.org/api-guide/pagination/#modifying-the-pagination-style)를 참조하세요.

---
#### `MAX_PAGINATE_BY`

---
**이 설정은 지원 중단 예정입니다.**  

pagination 스타일 설정에 대한 자세한 지침은 [setting the pagination style](http://www.django-rest-framework.org/api-guide/pagination/#modifying-the-pagination-style)를 참조하세요.

---
#### SEARCH_PARAM
`SearchFilter`가 사용하는 검색어를 지정하는데 사용 할 수 있는 검색어 매개 변수의 이름입니다.  

Default: `search`  

#### ORDERING_PARAM
`OrderingFilter`에 의해 반환 된 결과의 순서를 지정하는데 사용할 수 있는 쿼리 매개 변수의 이름입니다.  

Default: `ordering`

---

### Versioning settings

#### DEFAULT_VERSION
버전 정보가 없는 경우 `request.version`에 사용해야 하는 값입니다.  

Default: `None`

#### ALLOWED_VERSIONS
이 값을 설정하면 버전 체계에 의해 반환 될 수 있는 버전 set이 제한되며, 제공된 버전이 이 set에 포함되어 있지 않으면 오류가 발생합니다.  

Default: `none`  

#### VERSION_PARAM
미디어 타입 또는 URL 쿼리 매개변수와 같이 모든 버젼 지정 매개변수에 사용해야하는 문자열입니다.  

Default: `version`

---

### Authentication settings
다음 설정은 인증되지 않은 요청의 동작을 제어합니다.  

#### UNAUTHENTICATED_USER
인증되지 않은 요청에 대해 `request.user`를 초기화하는데 사용해야하는 클래스입니다.  

Default: `django.contrib.auth.models.AnonymousUser`  

#### UNAUTHENTICATED_TOKEN
인증되지 않은 요청에 대해 `request.auth`를 초기화하는데 사용해야 하는 클래스입니다.  

default: `None`

--

### Test settings
다음 설정은 APIRequestFactory 및 APIClient의 동작을 제어합니다.  

#### TEST_REQUEST_DEFAULT_FORMAT
테스트 요청을 할때 가용해야하는 기본 형식입니다.  
이 값은 `TEST_REQUEST_RENDERER_CLASSES`설정의 renderer 클래스 중 하나의 형식과 일치해야합니다.  

Default: `'multipart'`

### Schema generation controls
### Content type controls
### Date and time formatting
### Encodings
### View names and descriptions
### HTML Select Field cutoffs
### Miscellaneous settings





































