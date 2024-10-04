# Django REST Framework - Authentication & Permissions

ต่อเนื่องจาก tutorial ในสัปดาห์ที่แล้ว app code snippets ของเราไม่มีการป้องกันในส่วนของ Authentication and Authorization เลย ดังนั้นเราจะต้องเพิ่มในส่วนนี้ด้่วย 

โดยเราจะเพิ่มฟีเจอร์เหล่านี้ใน app code snippets ของเรากัน

- เพิ่ม field เก็บ owner ของ snippet
- เฉพาะ user ที่ login (authenticated) จึงจะสามารถสร้าง snippet ได้
- เฉพาะ owner ของ snippet จึงจะสามารถแก้ไขและลบ snippet ของตัวเองได้
- User ที่ไม่ได้ login (unauthenticated) จะสามารถ read code snippet ได้

## Adding information to our model

ก่อนอื่นเรามาเพิ่ม field owner ใน class `Snippet` แก้ไขไฟล์ `snippets/models.py`

```python
from django.db import models

# Create your models here.
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.IntegerField(default=0)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)
    # New fields
    owner = models.ForeignKey('auth.User', related_name='snippets', null=True, on_delete=models.CASCADE)
    highlighted = models.TextField(null=True)

    class Meta:
        ordering = ['created']
```

ทำการลบ database และ สร้างใหม่ จากนั้นทำการ makemigrations และ migrate อีกรอบ

และอย่าลืมไปเพิ่ม 'owner', 'highlighted' ใน `SnippetSerializer`

จากนั้นทำการสร้าง superuser โดยใช้คำสั่ง `python manage.py createsuperuser`

## Adding endpoints for our User models

เรามาสร้าง endpoints สำหรับทำ ดูข้อมูลของ model `User`

```python
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    # เพิ่ม snippets ซึ่งจะแสดง list ของ PK ของ snippets ที่ user นั้นๆ เป็นเจ้าของ
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ['id', 'username', 'snippets']
```

เพิ่ม view สำหรับดูข้อมูล user ใน `snippets/views.py`

```python
from django.contrib.auth.models import User

from snippets.models import Snippet
from snippets.serializers import SnippetSerializer, UserSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

...

class UserList(APIView):
    def get(self, request):
        users = User.objects.all()
        serializer = UserSerializer(users, many=True)
        return Response(serializer.data)


class UserDetail(APIView):
    def get_object(self, pk):
        try:
            return User.objects.get(pk=pk)
        except User.DoesNotExist:
            raise Http404

    def get(self, request, pk):
        user = self.get_object(pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

และ เพิ่ม path ใน `snippets/urls.py` ด้วย

```python
...
    path('users/', views.UserList.as_view()),
    path('users/<int:pk>/', views.UserDetail.as_view()),
...
```

## Associating Snippets with Users

เราต้องการว่าในจังหวะที่สร้าง snippet เราจะทำการระบุไปด้วยว่า owner เป็น `request.user`

แก้ไขในไฟล์ `snippets/.py` ใน `SnippetList.post()`

```python
...
    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(owner=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
...
```

ตอนนี้เราเตรียมการพร้อมแล้วเรามาเรียนรู้การทำ Authentication ของ REST Framework กัน

# Authentication in REST framework

[Doc](https://www.django-rest-framework.org/api-guide/authentication/)

เราสามารถกำหนด authentication scheme ได้มากกว่า 1 authentication scheme โดยกำหนดเป็น list ดังในตัวอย่าง ซึ่ง REST framework ก็จะพยายามทำการ authenticate แต่ละ class ใน list ที่กำหนด โดยถ้าสำเร็จก็จะกำหนดค่า `request.user` และ `request.auth` เพื่อเอาไปใช้งานใน view ต่อไป

สำหรับ `APIView` ใช้งานดังนี้

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    authentication_classes = [SessionAuthentication, BasicAuthentication]
    permission_classes = [IsAuthenticated]

    def get(self, request, format=None):
        content = {
            'user': str(request.user),
            'auth': str(request.auth),
        }
        return Response(content)
```

และสำหรับ function-based view ใช้งานดังนี้

```python
from rest_framework.decorators import api_view, authentication_classes, permission_classes

@api_view(['GET'])
@authentication_classes([SessionAuthentication, BasicAuthentication])
@permission_classes([IsAuthenticated])
def example_view(request, format=None):
    content = {
        'user': str(request.user),
        'auth': str(request.auth),
    }
    return Response(content)
```

นอกจากนั้นเรายังสามารถตั้งค่า default authentication classes ได้ใน `settings.py` ใน `settings.py` ที่ setting `REST_FRAMEWORK`

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ]
}
```

โดย authentication class ที่จะนำมาแนะนำในวันนี้คือ `TokenAuthentication`

## TokenAuthentication

TokenAuthentication นั้นจะเป็นการทำ token-based HTTP authentication scheme แบบง่ายๆ ซึ่งจะเหมาะสำหรับการใช้งานกับ website ในลักษณะ client-side rendering ที่มีการแยก front-end และ back-end

การใช้งานเริ่มต้นด้วยการเพิ่ม 'rest_framework.authtoken' ไปใน `INSTALL_APPS` ใน `settings.py`

```python
INSTALLED_APPS = [
    ...
    'rest_framework.authtoken'
]
```

โดยจะต้องทำการ run คำสั่ง `python manage.py migrate` ด้วย เพื่อสร้างตารางสำหรับเก็บ token

สำหรับการ authenticate ทางฝั่ง client จะต้องแนบ key `Authorization` มาใน HTTP header ด้วย ยกตัวอย่างเช่น

```text
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

โดยตัว token จะต้องนำหน้าด้วย keyword "Token" แต่ถ้าอยากเปลี่ยนเป็น keyword อื่นเช่น "Bearer" ก็สามารถทำได้โดยการ subclass TokenAuthentication และแก้ไข class variable `keyword`

โดยถ้า authenticate สำเร็จ ใน view เราจะได้ credentials ดังนี้

- `request.user` ซึ่งจะเป็น Django User instance
- `request.auth` ซึ่งจะเป็น rest_framework.authtoken.models.Token instance

### Generating Tokens

สามารถทำได้หลายทางดังนี้

#### By exposing an api endpoint

เราสามารถเพิ่ม path เข้าไปในไฟล์ `urls.py` ให้ชี้ไปที่ view `obtain_auth_token` ซึ่งเป็น view built-in มาให้ใน Django REST framework

```python
from rest_framework.authtoken import views
urlpatterns += [
    path('api-token-auth/', views.obtain_auth_token)
]
```

โดยทาง client จะส่ง username และ password มากับ body ของ POST request โดยเมื่อ authenticate สำเร็จ view `obtain_auth_token` ก็จะ response token ของ user นั้นๆ กลับไปให้เป็น JSON

```json
{ "token" : "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b" }
```

#### With Django Admin

เราสามารถเข้าไปกด "Add Token" ในหน้าเวบ admin ของ Django ได้เลย ดังภาพ

![admin](images/token-admin.png)

#### Using Django manage.py command

`python manage.py drf_create_token <username>`


## Let's continue our tutorial

เรามาลองใช้ TokenAuthentication กับ API ของ app Snippet ของเรากัน

1. เริ่มต้นด้วยการเพิ่ม 'rest_framework.authtoken' ใน `INSTALL_APPS`
2. Run command `python manage.py migrate`
3. ทำการเพิ่ม endpoint สำหรับ generate token

```python
# week14_tutorial/settings.py
from django.contrib import admin
from django.urls import path, include
from rest_framework.authtoken import views

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api-token-auth/", views.obtain_auth_token),
    path('', include('snippets.urls')),
]
```

4. Login เข้า Django admin เพื่อ สร้าง user มาลองใช้ generate token
5. ลองใช้ Postman ยิงไปที่ path `http://127.0.0.1:8000/api-token-auth/` เพื่อ generate token
6. ปรับแก้ไขใน view เพื่อให้ตรงตามเงื่อนไข

    - เฉพาะ user ที่ login (authenticated) จึงจะสามารถสร้าง snippet ได้
    - เฉพาะ owner ของ snippet จึงจะสามารถแก้ไขและลบ snippet ของตัวเองได้
    - User ที่ไม่ได้ login (unauthenticated) จะสามารถ read code snippet ได้

7. เริ่มด้วยการเพิ่ม decorator `@authentication_classes`

```python
...
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from rest_framework.authentication import TokenAuthentication
...
class MyTokenAuthentication(TokenAuthentication):
    keyword = "Bearer"

class SnippetList(APIView):
    authentication_classes = [MyTokenAuthentication]
    permission_classes = [IsAuthenticatedOrReadOnly]

    """
    List all snippets, or create a new snippet.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(owner=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
...
```

8. ลองทดสอบดูด้วย Postman
9. จากนั้นไปเพิ่มการตรวจสอบความเป็นเจ้าของในจังหวะ แก้ไข และ ลบ snippet

```python
...
class SnippetDetail(APIView):
    authentication_classes = [MyTokenAuthentication]
    permission_classes = [IsAuthenticatedOrReadOnly]
    """
    Retrieve, update or delete a snippet instance.
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk):
        snippet = self.get_object(pk)

        if request.user != snippet.owner:
            return Response("Only owner can edit.", status=status.HTTP_403_FORBIDDEN)

        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk):
        snippet = self.get_object(pk)

        if request.user != snippet.owner:
            return Response("Only owner can delete.", status=status.HTTP_403_FORBIDDEN)
    
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

10. ลองทดสอบดูด้วย Postman

## Custom permissions

สำหรับการทำ custom permission เราสามารถ override method `has_permission` และ `has_object_permission` ของ `BasePermission`

- `.has_permission(self, request, view)`
- `.has_object_permission(self, request, view, obj)`

โดยการ return True หมายถึงผ่านการตรวจสอบ permission และ False หมายถึงไม่่ผ่าน

เพิ่มเติมจากใน tutorial เราต้องการตรวจสอบ permission เพิ่มเติมดังนี้

- method GET จะต้องมี permission `snippets.view_snippet`
- method POST จะต้องมี permission `snippets.add_snippet`
- method PUT จะต้องมี permission `snippets.change_snippet`
- method DELETE จะต้องมี permission `snippets.delete_snippet`

เราสามารถเขียน custom permission class ได้ดังตัวอย่าง

```python
# snippets/permissions.py

from rest_framework import permissions

class SnippetPermission(permissions.BasePermission):
    """
    Object-level permission to only allow owners of an object to edit and delete it.
    """

    def has_permission(self, request, view):
        # SAFE_METHODS = GET, HEAD or OPTIONS
        if request.method in permissions.SAFE_METHODS:
            return request.user.has_perm("snippets.view_snippet")
        elif request.method == "POST":
            return request.user.has_perm("snippets.add_snippet")
        elif request.method == "PUT":
            return request.user.has_perm("snippets.change_snippet")
        elif request.method == "DELETE":
            return request.user.has_perm("snippets.delete_snippet")
        return False
    
    def has_obj_permission(self, request, view, obj):
        return obj.created_by == request.user
```

เพิ่ม `SnippetPermission` เข้าไปใน view

```python
...
class SnippetList(APIView):
    authentication_classes = [MyTokenAuthentication]
    permission_classes = [IsAuthenticated, SnippetPermission]
...
class SnippetDetail(APIView):
    authentication_classes = [MyTokenAuthentication]
    permission_classes = [IsAuthenticated, SnippetPermission]
...
```

**Note**: list ของ permission_classes จะ support & (and), | (or) and ~ (not) ดังตัวอย่าง `permission_classes = [IsAuthenticated|ReadOnly]` 
