# JWT (JSON Web Tokens)

## What is JSON Web Token?

> JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with HMAC algorithm) or a public/private key pair using RSA.

ข้อดีของ JWT คือ
1. **Compact**: ข้อมูลที่ส่งถูกทำให้มีขนาดเล็กสำหรับส่งผ่าน URL หรือ POST parameters และ ส่งได้เร็ว
2. **Self-contained**: ใน payload ที่ถูกสร้างเป็น JWT นั้นมีข้อมูลเกี่ยวกับ user ครบถ้วน เพื่อที่จะได้ไม่ต้อง query ข้อมูลจาก DB

## Which is the JSON Web Token structure?

JWT ประกอบด้วย 3 ส่วนซึ่งแบ่งด้วย dot (.) ได้แก่

- Header
- Payload
- Signature

ซึ่งจะหน้าตาเป็นดังนี้ `xxxxxx`.`yyyyyy`.`zzzzzz`

### Header

ใน header มักจะประกอบด้วย 2 ส่วนได้่แก่ ประเภทของ token ซึ่งก็คือ "JWT" และ hashing algorithm ซึ่งอาจจะเป็น HMAC, SHA256 หรือ RSA

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Header จะถูกเข้ารหัสด้วย **Base64Url**

### Payload

ในส่วนที่ 2 จะเป็นส่วนของ "claims" ซึ่งหมายถึงส่วนที่เป็นข้อมูล user และ metadata ที่เกี่ยวข้อง

Claims มีทั้งหมด 3 ประเภทได้แก่

- **Resersed claims**: ซึ่งเป็นชื่อ properties ที่ถูกสงวนไว้ ยกตัวอย่างเช่น **iss** (issuer), **exp** (expiration time), **sub** (subject), **aud** (audience) - *ข้อมูลพวกนี้ไม่จำเป็น (not mandatory)*
- **Public claims**: ซึ่งเป็นชื่อ properties ที่ถูกกำหนดไว้ใน list [IANA JSON Web Token Registry](https://www.iana.org/assignments/jwt/jwt.xhtml#claims)
- **Private claims**: ซึ่งเป็นชื่อ properties ที่กำหนดเอง

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

Payload จะถูกเข้ารหัสด้วย **Base64Url**

> Do note that for signed tokens this information, though protected against **tampering**, is readable by anyone. Do not put secret information in the payload or header elements of a JWT unless it is encrypted.

### Signature

ในการสร้างส่วน `Signature` เราจะต้องนำส่วน encoded header, encoded payload, secret, algorithm ที่กำหนดใน header มารวมกันแล้วทำการ sign ดังตัวอย่าง

```javascript
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

หน้าที่สำคัญของ `Signature` คือใช้ในการ verify ว่าข้อมูลไม่ได้ถูกแก้ไขในระหว่างทาง

หน้าตาของ JWT token ที่นำทั้ง 3 ส่วนมารวมกันแล้วเป็นดังรูป

![jwt](images/jwt.png)

สามารถลองเล่น JWT ได้ที่ [jwt.io debugger](https://jwt.io/#debugger-io)

โดยปกติเราจะส่ง JWT แนบไปกับ HTTP header ที่ key `Authorization` ดังตัวอย่าง

```text
Authorization: Bearer <token>
```

JWT authentication เป็น stateless authentication mechanism ซึ่งจะเหมาะกับการใช้งาน APIs ระหว่าง frontend และ backend ดังภาพด้านล่าง

![sequence](images/jwt-sequence.png)

## JWT with Django REST framework

สำหรับการใช้งาน JWT กับ REST framework เราจะใช้งาน library `Simple JWT` ซึ่งมีขั้นตอนการใช้งานดังนี้

1. เริ่มต้นด้วยการติดตั้งด้วยคำสั่ง `pip install djangorestframework-simplejwt`
2. เพิ่มการตั้งค่าใน `settings.py`

```python
REST_FRAMEWORK = {
    ...
    'DEFAULT_AUTHENTICATION_CLASSES': (
        ...
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
    ...
}
```

3. ไปเพิ่ม path ในไฟล์ `urls.py` ตัวหลัก

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    ...
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

4. ทดลองใช้งานโดยใช้ Postman ยิง POST request ไปที่ `http://127.0.0.1:8000/api/token/` จะได้รับ token กลับมา

```json
{
    "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTcyNzk3NTMwMywiaWF0IjoxNzI3ODg4OTAzLCJqdGkiOiI3NjJiODgwZTJmZmI0YTcxOGY4Y2U1ODA4YWI2MzgzNCIsInVzZXJfaWQiOjJ9.2qJTmGQdrB5-BLMj6gGbpE_PW_N3RrprggQfamj3zXk",
    "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzI3ODg5MjAzLCJpYXQiOjE3Mjc4ODg5MDMsImp0aSI6IjFiNjY0N2E3OWU3MjQzMjc5NWNiMWMzMGQ2NTFiNzQyIiwidXNlcl9pZCI6Mn0.ECZPXlaOF4zwE_1r435nyexXMrsxvlTZy89EYzeVVms"
}
```

5. แก้ไขใน view เปลี่ยนไปใช้ `rest_framework_simplejwt.authentication.JWTAuthentication` แทน `rest_framework.authentication.TokenAuthentication`

```python
...
class SnippetList(APIView):
    authentication_classes = [JWTAuthentication]
    permission_classes = [IsAuthenticatedOrReadOnly]
...
class SnippetDetail(APIView):
    authentication_classes = [JWTAuthentication]
    permission_classes = [IsAuthenticatedOrReadOnly]
...
```

6. ทดสอบโดยใช้ Postman อีกครั้ง คราวนี้แนบ JWT แทนที่ token เดิม (ใช้ตัว "access")

## Customizing token claims

ถ้าเราอยากจะเพิ่ม claims เข้าไปใน payload ของ JWT สามารถทำได้โดยการ extend class `TokenObtainPairSerializer`

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # Add custom claims
        token['first_name'] = user.first_name
        # ...

        return token
```

และไปเพิ่ม setting `SIMPLE_JWT` ใน `settings.py`

```python
# Django project settings.py
...

SIMPLE_JWT = {
  # It will work instead of the default serializer(TokenObtainPairSerializer).
  "TOKEN_OBTAIN_SERIALIZER": "my_app.serializers.MyTokenObtainPairSerializer",
  # ...
}
```
