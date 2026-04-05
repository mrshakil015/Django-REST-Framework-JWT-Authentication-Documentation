# Django REST Framework JWT Authentication Documentation
This guide explains how to implement JWT Authentication in a Django REST Framework project using a custom user model.

# 1. Install Required Packages

```python
pip install djangorestframework
pip install djangorestframework-simplejwt
pip install django-cors-headers
```

## 2. Configure `settings.py`

### Add Installed Apps

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'corsheaders',
    'rest_framework_simplejwt',
]
```

### Custom User Model

Tells Django to use your custom user model instead of the default one.

```python
AUTH_USER_MODEL = 'users_auth.User'
```

### DRF Authentication Setup

Defines JWT as the default authentication method for all APIs.

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}
```

### JWT Configuration

Controls token lifetime and authentication header format.

```python
from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(days=1),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

### CORS Configuration (Frontend Integration)

Allows your frontend (React, etc.) to communicate with backend APIs.

```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:5173",
    "http://127.0.0.1:5500"
]

CORS_ALLOW_CREDENTIALS = True

CORS_ALLOW_HEADERS = [
    'content-type',
    'authorization',
    'x-requested-with',
    'accept',
]

CORS_ALLOW_METHODS = [
    'GET',
    'POST',
    'PUT',
    'PATCH',
    'DELETE',
    'OPTIONS',
]
```

Add CORS Middleware: `corsheaders.middleware.CorsMiddleware`

```python
MIDDLEWARE = [
	  ......
    'corsheaders.middleware.CorsMiddleware', #----------- This line
    'django.middleware.common.CommonMiddleware',
		............
]
```

# 3. Custom User Model

Extends Django’s default user to include roles (manager/staff).

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    USER_TYPE_CHOICES = (
        ('manager', 'Manager'),
        ('staff', 'Staff'),
    )

    user_type = models.CharField(max_length=10, choices=USER_TYPE_CHOICES)

    def __str__(self):
        return self.username
```

# 4. Serializer

### Register Serializer

Validates user input and creates a new user.

```python
from rest_framework import serializers
from .models import User

class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)
    confirm_password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'confirm_password', 'user_type']

    def validate(self, data):
        if data['password'] != data['confirm_password']:
            raise serializers.ValidationError("Passwords do not match")
        return data

    def create(self, validated_data):
        validated_data.pop('confirm_password')
        user = User.objects.create_user(**validated_data)
        return user
```

### Login Serializer

Authenticates user credentials before login.

```python
from rest_framework import serializers
from django.contrib.auth import authenticate

class LoginSerializer(serializers.Serializer):
    username = serializers.CharField()
    password = serializers.CharField(write_only=True)

    def validate(self, data):
        user = authenticate(
            username=data['username'],
            password=data['password']
        )

        if not user:
            raise serializers.ValidationError("Invalid credentials")

        data['user'] = user
        return data
```

# 5. Views

### Register View

Handles user registration via API.

```python
from rest_framework import generics, status
from rest_framework.response import Response
from .serializers import RegisterSerializer

class RegisterView(generics.CreateAPIView):
    serializer_class = RegisterSerializer
    permission_classes = []

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response({
			        "success": True,
							"message": "User registered successfully"
		        },
            status=status.HTTP_201_CREATED
        )
```

### Login View

Authenticates user and returns JWT tokens.

```python
from rest_framework import generics, status
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken
from .serializers import LoginSerializer

class LoginView(generics.GenericAPIView):
    serializer_class = LoginSerializer
    permission_classes = []

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        user = serializer.validated_data['user']
        refresh = RefreshToken.for_user(user)

        return Response({
            "access": str(refresh.access_token),
            "refresh": str(refresh),
            "user": {
                "username": user.username,
                "email": user.email,
                "user_type": user.user_type
            }
        }, status=status.HTTP_200_OK)
```

# 6. URL Configuration

Defines authentication-related API endpoints.

```python
from django.urls import path
from .views import RegisterView, LoginView
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
    path('login/', LoginView.as_view(), name='login'),

    path('token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

**Main `urls.py`**

Includes app routes into the main project.

```python
from django.urls import path, include

urlpatterns = [
    path('api/', include('users_auth.urls')),
]
```

## 7. API Testing

### Register API

Creates a new user account apply `POST` method

```json
/api/register/
```

Request Body

```json
{
  "username":"shakil",
  "email":"shakil@example.com",
  "password":"12345678",
  "confirm_password":"12345678",
  "user_type":"manager"
}
```

Return Response

```json
{
  "success":true,
  "message":"User registered successfully"
}
```

### Login API

Returns JWT tokens after successful authentication.

```json
/api/login/
```

Request Body

```json
{
  "username":"shakil",
  "password":"12345678"
}
```

Return Response

```json
{
  "access":"your_access_token",
  "refresh":"your_refresh_token",
  "user": {
    "username":"shakil",
    "email":"shakil@example.com",
    "user_type":"manager"
  }
}
```

## 8. How Authentication Works

How JWT is used in requests.

- Login generates **Access Token** and **Refresh Token**
- Access token is sent with every request
- Refresh token is used to get a new access token

```
Authorization: Bearer your_access_token
```
