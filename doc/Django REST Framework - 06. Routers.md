# Django REST Framework - Routers

---

_"Resource routing allows you to quickly declare all of the common routes for a given resourceful controller. Instead of declaring separate routes for your index... a resourceful route declares them in a single line of code."_  

_"리소스 라우팅을 사용하면 주어진 리소스가 많은 컨트롤러에 대한 모든 일반 경로를 빠르게 선언 할 수 있습니다. 인덱스에 대해 별도의 경로를 선언하는 대신... 유용한 루트는 코드 한 줄로 선언합니다."_  

_— Ruby on Rails Documentation_

---

## Routers
`Rails`와 같은 일부 웹 프레임워크는 응용 프로그램의 URL을 들어오는 요청을 처리하는 논리에 매핑하는 방법을 자동으로 결정하는 기능을 제공합니다.  
REST 프레임워크는 Django에 대한 자동 URL라우팅을 지원을 추가하고 뷰 로직을 URL set에 간단하고 빠르게 연관되게 연결하는 방법을 제공합니다.

### Usage
다음은 `SimpleRouter`를 사용하는 간단한 URL 구성의 예입니다.

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)
urlpatterns = router.urls
```
`register()`메서드는 두 가지 필수 인수가 있습니다.  

## API Guide
### SimpleRouter
### DefaultRouter

## Custom Routers
### Customizing dynamic routes
### Example
### Advanced custom routers

## Third Party Packages
### DRF Nested Routers
### ModelRouter (wq.db.rest)
### DRF-extensions

























