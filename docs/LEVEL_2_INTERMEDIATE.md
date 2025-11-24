# Level 2: Intermediate - Authentication, Permissions & Querysets

## Goal

Build secure APIs with robust authentication, permissions, filtering, searching, ordering, and pagination. By the end of this level, you'll be able to create production-ready, secure REST APIs.

## Note on Examples

This guide uses **Task API** as the primary example throughout explanation sections for consistency with Level 1. Complete step-by-step implementations are provided for both **Task API** (user-owned, private data) and **Book API** (can be public or user-owned) to show different use cases:

- **Task API examples**: Used in explanations (permissions, filtering, searching, etc.) - represents private, user-owned data
- **Book API examples**: Used in complete step-by-step section - represents content that can be public or user-owned
- **Both examples**: Help you understand different scenarios and patterns

You can apply all concepts to any model (Task, Book, Product, Post, etc.).

## Table of Contents

1. [Django Authentication System](#django-authentication-system)
2. [DRF Authentication Classes](#drf-authentication-classes)
3. [JWT Authentication Setup](#jwt-authentication-setup)
4. [Permissions](#permissions)
5. [Filtering](#filtering)
6. [Searching](#searching)
7. [Ordering](#ordering)
8. [Pagination](#pagination)
9. [Step-by-Step: Secure Book API](#step-by-step-secure-book-api)
10. [Step-by-Step: Secure Task API](#step-by-step-secure-task-api)
11. [UserProfile API](#userprofile-api)
12. [Throttling](#throttling)
13. [Exception Handling](#exception-handling)
14. [Testing Authenticated APIs](#testing-authenticated-apis)
15. [Exercises](#exercises)
16. [Common Errors and Solutions](#common-errors-and-solutions)
17. [Add-ons](#add-ons)

## Django Authentication System

### Why Authentication?

**The Problem**: Without authentication, anyone can access your API and modify data. This is a security risk!

**Real-world analogy**: 
- **No authentication** = House with no locks (anyone can enter)
- **With authentication** = House with keys (only authorized people enter)

**Why it matters**:
- **Security**: Protect user data
- **Personalization**: Show users only their data
- **Accountability**: Track who did what
- **Access control**: Limit what users can do

### Understanding Django Users

Django comes with a built-in User model that handles user accounts:

**What is a User model?**
- Represents a person using your application
- Stores login credentials (username, password)
- Tracks user information (email, name)
- Manages permissions and access levels

**Why use Django's User model?**
- **Ready-made**: No need to build from scratch
- **Secure**: Password hashing built-in
- **Tested**: Used by millions of applications
- **Flexible**: Can be extended for custom needs

```python
from django.contrib.auth.models import User

# User fields
user.username  # Username
user.email     # shed password
user.first_nameEmail
user.password  # Ha
user.last_name
user.is_staff  # Admin access
user.is_active # Account status
user.is_superuser
user.date_joined
```

### Creating Users

**Why multiple methods?**
- Different methods serve different purposes
- Some hash passwords (secure), others don't (insecure)
- Always use methods that hash passwords!

```python
# In Django shell: python manage.py shell

from django.contrib.auth.models import User

# Method 1: create_user (recommended - hashes password)
user = User.objects.create_user(
    username='john',
    email='john@example.com',
    password='securepassword123'
)
```

**Why `create_user`?**
- **Password hashing**: Converts password to secure hash (can't be reversed)
- **Validation**: Checks username/email format
- **Default values**: Sets sensible defaults (is_active=True)
- **Security**: Never store plain text passwords!

**What is password hashing?**
- Plain password: `"mypassword123"` ❌ (stored as-is, anyone can read)
- Hashed password: `"pbkdf2_sha256$..."` ✅ (one-way encryption, can't be reversed)
- When user logs in, Django hashes their input and compares with stored hash

```python
# Method 2: create_superuser
admin = User.objects.create_superuser(
    username='admin',
    email='admin@example.com',
    password='admin123'
)
```

**Why `create_superuser`?**
- Creates user with admin privileges (`is_staff=True`, `is_superuser=True`)
- Can access Django admin panel
- Has all permissions by default
- Use for site administrators

```python
# Method 3: Using User.objects.create (not recommended - password not hashed)
```

**Why NOT use `User.objects.create`?**
- **Security risk**: Password stored in plain text!
- No validation
- No password hashing
- **Never use this for passwords!**

### Custom User Model

**Why create a custom user model?**

**The Problem**: Django's default User model has limited fields (username, email, password). What if you need:
- Phone numbers
- Profile pictures
- Bio/description
- Custom fields

**The Solution**: Extend Django's User model with your own fields.

**When to use custom user model?**
- **Production apps**: Almost always
- **Need extra fields**: Phone, avatar, bio, etc.
- **Different login**: Email instead of username
- **Before first migration**: Must set before creating database

**Why before first migration?**
- Changing user model after migration is very difficult
- Requires database restructuring
- Can cause data loss
- **Always decide early!**

```python
# api/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    phone_number = models.CharField(max_length=15, blank=True)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.username
```

**Explanation**:
- **`AbstractUser`**: Base class with all default User fields (username, email, password, etc.)
- **Extend it**: Add your custom fields
- **Inherits everything**: All User functionality still works
- **Add custom fields**: Phone, bio, avatar, etc.

**Update settings.py:**

```python
# core/settings.py
AUTH_USER_MODEL = 'api.CustomUser'
```

**What this does**:
- Tells Django to use your custom user model
- Replaces default User model everywhere
- Must be set before first migration!

**Important**: Set this before your first migration! Once migrations are created, changing is very difficult.

## DRF Authentication Classes

### Authentication vs Authorization

**These are two different but related concepts:**

- **Authentication**: Who are you? (Identity)
  - Verifies user's identity
  - "Are you really John?"
  - Examples: Login, password check, token verification
  
- **Authorization**: What can you do? (Permissions)
  - Determines what user can access
  - "Can John edit this book?"
  - Examples: Admin only, owner only, read-only

**Real-world analogy**:
- **Authentication** = Showing ID at entrance (proving who you are)
- **Authorization** = Security guard checking if you have access to VIP area (what you can do)

**Why both?**
- Authentication without authorization = Everyone can do everything
- Authorization without authentication = Can't identify users
- **Need both** for secure APIs!

**The Flow**:
```
1. User sends credentials → Authentication (who are you?)
2. If authenticated → Check permissions → Authorization (what can you do?)
3. If authorized → Allow access
4. If not authorized → Deny access (403 Forbidden)
```

### DRF Authentication Types

#### 1. Session Authentication

**What is Session Authentication?**

Uses Django's session framework. Stores authentication state on the server.

**How it works**:
1. User logs in with username/password
2. Server creates a session (stored in database/cache)
3. Server sends session ID in cookie to browser
4. Browser sends cookie with each request
5. Server checks session to identify user

**Why use it?**
- **Simple**: Built into Django
- **Secure**: Session stored server-side
- **Good for**: Web apps (same domain)
- **Automatic**: Browser handles cookies

**Limitations**:
- **Same-origin only**: Doesn't work well with mobile apps
- **Cookie-based**: Requires browser cookie support
- **Server storage**: Needs database/cache for sessions

**When to use**:
- Web applications (React, Vue on same domain)
- Traditional web apps
- When you control both frontend and backend

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**Why this configuration?**
- Sets default authentication for all views
- Can override per-view if needed
- Multiple authentication classes can be used (tries each until one works)

#### 2. Token Authentication

Simple token-based authentication. Good for client-server setups.

```bash
# Install
pip install djangorestframework-simplejwt
```

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',  # Add this
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

#### 3. JWT Authentication (Recommended)

**What is JWT?**

**JWT** = JSON Web Token

**How it works**:
1. User logs in with username/password
2. Server creates a JWT token (contains user info)
3. Server sends token to client
4. Client stores token (localStorage, memory)
5. Client sends token with each request in header
6. Server verifies token and extracts user info

**Why JWT is popular**:
- **Stateless**: No server-side storage needed
- **Scalable**: Works across multiple servers
- **Mobile-friendly**: Works with mobile apps
- **Industry standard**: Used by major companies
- **Self-contained**: Token has all user info

**JWT Structure**:
```
Header.Payload.Signature

Example:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxfQ.signature
```

**Parts**:
- **Header**: Algorithm and token type
- **Payload**: User data (ID, username, etc.)
- **Signature**: Ensures token wasn't tampered with

**Why stateless matters**:
- **Session auth**: Server stores session → needs database lookup
- **JWT auth**: Token contains info → no database lookup needed
- **Result**: Faster, more scalable

**Security considerations**:
- Tokens expire (access token: 1 hour, refresh token: 1 day)
- Tokens can be blacklisted if compromised
- Always use HTTPS in production (tokens in headers)

**When to use JWT**:
- ✅ Mobile apps (iOS, Android)
- ✅ Single Page Applications (SPA)
- ✅ Microservices
- ✅ APIs used by multiple clients
- ❌ Not ideal for traditional server-rendered apps

## JWT Authentication Setup

### Step 1: Install Package

```bash
pip install djangorestframework-simplejwt
pip freeze > requirements.txt
```

### Step 2: Add to INSTALLED_APPS

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',
    'api',
]
```

### Step 3: Configure JWT Settings

```python
# core/settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
}
```

**Explanation of JWT Settings**:

- **`ACCESS_TOKEN_LIFETIME`**: How long access token is valid (60 minutes)
  - **Why short?**: If stolen, expires quickly
  - **Trade-off**: Security vs convenience
  
- **`REFRESH_TOKEN_LIFETIME`**: How long refresh token is valid (1 day)
  - **Why longer?**: Used to get new access tokens
  - **More secure**: Stored more securely, used less often
  
- **`ROTATE_REFRESH_TOKENS`**: Issue new refresh token when used
  - **Why?**: If old token stolen, it becomes invalid
  - **Security**: Limits damage from token theft
  
- **`BLACKLIST_AFTER_ROTATION`**: Add old refresh token to blacklist
  - **Why?**: Prevents reuse of old tokens
  - **Security**: One-time use tokens
  
- **`UPDATE_LAST_LOGIN`**: Update user's last login time
  - **Why?**: Track user activity, security monitoring
  
- **`ALGORITHM`**: How token is signed ('HS256' = HMAC SHA-256)
  - **Why HS256?**: Fast, secure, widely supported
  - **Alternative**: RS256 (asymmetric, more secure but slower)
  
- **`SIGNING_KEY`**: Secret key to sign tokens (use Django SECRET_KEY)
  - **Why secret?**: Anyone with key can create valid tokens
  - **Security**: Never expose this key!
  
- **`AUTH_HEADER_TYPES`**: Token type in header ('Bearer')
  - **Why Bearer?**: Standard OAuth2 format
  - **Usage**: `Authorization: Bearer <token>`
  
- **`USER_ID_FIELD`**: Which User model field to use ('id')
- **`USER_ID_CLAIM`**: Where to store user ID in token ('user_id')
```

### Step 4: Add JWT URLs

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    # JWT endpoints
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### Step 5: Create User Registration View

```python
# api/serializers.py
from rest_framework import serializers
from django.contrib.auth.models import User

class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'password_confirm', 'first_name', 'last_name']

    def validate(self, data):
        if data['password'] != data['password_confirm']:
            raise serializers.ValidationError("Passwords don't match")
        return data

    def create(self, validated_data):
        validated_data.pop('password_confirm')
        user = User.objects.create_user(**validated_data)
        return user
```

```python
# api/views.py
from rest_framework import generics, status
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken
from django.contrib.auth.models import User
from .serializers import UserRegistrationSerializer

class UserRegistrationView(generics.CreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserRegistrationSerializer
    permission_classes = []  # Allow anyone to register

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        
        # Generate tokens
        refresh = RefreshToken.for_user(user)
        
        return Response({
            'user': serializer.data,
            'refresh': str(refresh),
            'access': str(refresh.access_token),
        }, status=status.HTTP_201_CREATED)
```

```python
# api/urls.py
from django.urls import path
from . import views

urlpatterns = [
    # ... other URLs ...
    path('register/', views.UserRegistrationView.as_view(), name='register'),
]
```

### Step 6: Test JWT Authentication

```bash
# Register user
curl -X POST http://localhost:8000/api/register/ \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "email": "test@example.com",
    "password": "testpass123",
    "password_confirm": "testpass123"
  }'

# Get token
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}'

# Use token
TOKEN="your_access_token_here"
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/
```

## Permissions

### What are Permissions?

**Permissions** determine what authenticated users can do.

**Why Permissions?**
- **Security**: Not all users should do everything
- **Data protection**: Users should only see/edit their own data
- **Role-based access**: Admins can do more than regular users
- **Compliance**: Meet security requirements

**Real-world analogy**:
- **No permissions** = Everyone has master key (bad!)
- **With permissions** = Different keys for different access levels (good!)

**Permission Levels**:
1. **Public**: Anyone can access (AllowAny)
2. **Authenticated**: Must be logged in (IsAuthenticated)
3. **Owner**: Can only access own data (IsOwnerOrReadOnly)
4. **Admin**: Only staff can access (IsAdminUser)

### Built-in Permission Classes

#### 1. AllowAny

**What it does**: Anyone can access, no authentication required.

**Why use it?**
- Public data (blog posts, product listings)
- Registration/login endpoints
- Public APIs
- When you want maximum accessibility

**Security note**: Use carefully! Only for truly public data.

**When to use**:
- ✅ Public blog posts
- ✅ Product catalogs
- ✅ Public user profiles
- ❌ User data
- ❌ Admin functions

```python
from rest_framework.permissions import AllowAny

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [AllowAny]
    # ...
```

**Note**: In this example, we use `TaskViewSet`, but the same pattern applies to any ViewSet (Book, Product, etc.).

#### 2. IsAuthenticated

**What it does**: User must be logged in (authenticated) to access.

**Why use it?**
- **Protect data**: Only logged-in users can access
- **User-specific content**: Show personalized data
- **Account required**: Force users to create accounts
- **Security**: Basic protection for most endpoints

**When to use**:
- ✅ User's own tasks/books
- ✅ User profiles
- ✅ Private data
- ✅ Any endpoint requiring user identity

**What happens if not authenticated?**
- Returns `401 Unauthorized`
- Client must login first
- Token/session required

```python
from rest_framework.permissions import IsAuthenticated

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    # ...
```

**Why this configuration?**
- Applies to all actions (list, create, retrieve, update, delete)
- Can override per-action if needed
- Most common permission for user data

**Note**: This is the most common permission for user-owned resources like tasks, where users should only see their own data.

#### 3. IsAdminUser

User must be staff/admin.

```python
from rest_framework.permissions import IsAdminUser

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAdminUser]
    # ...
```

**Note**: Use this for admin-only endpoints. For user data like tasks, you typically want `IsAuthenticated` instead.

#### 4. IsAuthenticatedOrReadOnly

**What it does**: 
- **Authenticated users**: Can do everything (read, create, update, delete)
- **Unauthenticated users**: Can only read (GET requests)

**Why use it?**
- **Public viewing**: Anyone can see data
- **Protected editing**: Only logged-in users can modify
- **Common pattern**: Blog posts, comments, products
- **User engagement**: Encourage registration to participate

**Real-world example**:
- Blog: Anyone can read posts, only logged-in users can comment
- Products: Anyone can browse, only logged-in users can add to cart
- Comments: Anyone can read, only logged-in users can post

**When to use**:
- ✅ Public content with user contributions
- ✅ Read-mostly APIs
- ✅ When you want public access but protected writes
- ❌ Private/sensitive data

```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
    # ...
```

**What this means**:
- `GET /api/tasks/` → Anyone can access ✅
- `POST /api/tasks/` → Must be authenticated ✅
- `PUT /api/tasks/1/` → Must be authenticated ✅
- `DELETE /api/tasks/1/` → Must be authenticated ✅

**Note**: For private user data like personal tasks, you'd typically use `IsAuthenticated` instead. This permission is better for public content.

### Custom Permissions

#### IsOwnerOrReadOnly

Only the owner can edit/delete, others can read.

```python
# api/permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners to edit/delete objects.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions for any request
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions only to the owner
        return obj.owner == request.user
```

**Use in views:**

```python
# api/views.py
from .permissions import IsOwnerOrReadOnly

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

**Why this works for tasks:**
- Users can read all tasks (or filter to their own in `get_queryset()`)
- Users can only edit/delete their own tasks
- Perfect for user-owned resources like tasks, notes, etc.

### Permission at View Level

**Why different permissions per action?**
- **List/Retrieve**: Maybe public (AllowAny) or authenticated (IsAuthenticated)
- **Create**: Usually requires authentication
- **Update/Delete**: Usually requires ownership (IsOwnerOrReadOnly)

```python
# Different permissions for different actions
class TaskViewSet(viewsets.ModelViewSet):
    def get_permissions(self):
        if self.action == 'create':
            permission_classes = [IsAuthenticated]  # Must be logged in to create
        elif self.action in ['update', 'partial_update', 'destroy']:
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]  # Must own to edit
        else:
            permission_classes = [IsAuthenticated]  # Must be logged in to view
        return [permission() for permission in permission_classes]
```

**Explanation**:
- **`create`**: Anyone authenticated can create tasks
- **`update/delete`**: Only task owner can modify
- **`list/retrieve`**: Authenticated users can view (you might filter to own tasks in `get_queryset()`)

**Alternative for private tasks** (users only see their own):
```python
class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]  # Same for all actions
    
    def get_queryset(self):
        # Users only see their own tasks
        return Task.objects.filter(owner=self.request.user)
```

## Filtering

### What is Filtering?

**Filtering** allows clients to request specific subsets of data based on criteria.

**The Problem**: Without filtering, clients get ALL data, even if they only need a subset.

**Example**:
- Without filtering: `GET /api/tasks/` → Returns all tasks (10,000 tasks) ❌
- With filtering: `GET /api/tasks/?completed=false` → Returns only incomplete tasks ✅

**Why Filtering?**
- **Performance**: Return only needed data (faster queries)
- **User experience**: Users find what they're looking for
- **Bandwidth**: Less data transferred
- **Database efficiency**: Queries only relevant records

**Real-world analogy**:
- **No filtering** = Library with no organization (find book by searching all shelves)
- **With filtering** = Library with sections (go directly to "Fiction" section)

**Common Filter Types**:
- **Exact match**: `?completed=true` (completed exactly equals true)
- **Contains**: `?title__icontains=django` (title contains "django")
- **Boolean**: `?completed=false` (filter by boolean field)
- **Date range**: `?created_after=2024-01-01` (created after date)

### Install django-filter

**Why django-filter?**
- Makes filtering easy and powerful
- Supports complex filters
- Works seamlessly with DRF
- Handles edge cases

```bash
pip install django-filter
```

### Configure in settings.py

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

### Basic Filtering

**How it works**:
1. Client adds query parameters to URL: `?completed=false`
2. Django Filter intercepts request
3. Applies filters to queryset
4. Returns filtered results

**Why `filter_backends`?**
- Tells DRF which filtering system to use
- Can use multiple backends (filtering + searching + ordering)
- Each backend handles different aspects

```python
# api/views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import viewsets, filters

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
    filter_backends = [DjangoFilterBackend]  # Enable filtering
    filterset_fields = ['completed', 'created_at']  # Which fields can be filtered
```

**Explanation**:
- **`filter_backends`**: List of filtering systems to use
- **`filterset_fields`**: Simple list of fields that can be filtered
- **Exact match**: `?completed=false` finds tasks where completed exactly equals False

**Usage:**

```bash
# Filter by completed status
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/?completed=false
```

**What happens**:
- URL parameter `completed=false`
- Django Filter finds tasks where `completed` field equals False
- Returns only incomplete tasks

```bash
# Filter by multiple fields (if you have date field)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false&created_at=2024-01-01"
```

**What happens**:
- Both filters applied: `completed=false` AND `created_at=2024-01-01`
- Returns tasks matching BOTH criteria
- Filters are combined with AND logic

### Advanced Filtering with FilterSet

**Why use FilterSet instead of `filterset_fields`?**
- **Advanced lookups**: `icontains`, `gte`, `lte`, etc.
- **Custom field names**: `created_after` instead of `created_at__gte`
- **More control**: Complex filtering logic
- **Better UX**: User-friendly filter names

```python
# api/filters.py
import django_filters
from .models import Task

class TaskFilter(django_filters.FilterSet):
    # Filter by title containing text (case-insensitive)
    title = django_filters.CharFilter(lookup_expr='icontains')
    
    # Filter by date range
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')
    
    # Filter by completed status
    completed = django_filters.BooleanFilter()

    class Meta:
        model = Task
        fields = ['completed', 'title', 'created_after', 'created_before']
```

**Explanation**:
- **`title`**: `?title=django` finds tasks with "django" in title (case-insensitive)
- **`created_after`**: `?created_after=2024-01-01` finds tasks created on or after date
- **`created_before`**: `?created_before=2024-12-31` finds tasks created before date
- **`completed`**: `?completed=false` finds incomplete tasks

```python
# api/views.py
from .filters import TaskFilter

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
    filterset_class = TaskFilter  # Use FilterSet instead of filterset_fields
```

**Usage:**
```bash
# Filter by title containing "django"
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?title=django"

# Filter by date range
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?created_after=2024-01-01&created_before=2024-12-31"

# Combine filters
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false&title=important"
```

## Searching

### What is Searching?

**Searching** allows users to find data by searching across multiple fields with a single query.

**Difference from Filtering**:
- **Filtering**: Exact matches on specific fields (`?author=John`)
- **Searching**: Partial matches across multiple fields (`?search=john` finds in title, author, description)

**Why Searching?**
- **User-friendly**: Users don't need to know exact field names
- **Flexible**: Search across multiple fields at once
- **Common pattern**: Search boxes in web apps
- **Better UX**: "Search for anything" vs "Filter by specific field"

**Real-world analogy**:
- **Filtering** = Library catalog with specific fields (author, genre, year)
- **Searching** = Google search (type anything, finds matches anywhere)

### SearchFilter

**How it works**:
1. User types search term: `?search=django`
2. DRF searches across specified fields
3. Returns records matching search term in ANY field
4. Uses case-insensitive partial matching by default

```python
# api/views.py
from rest_framework import viewsets, filters

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
    filter_backends = [filters.SearchFilter]  # Enable searching
    search_fields = ['title', 'desc']  # Fields to search in
```

**Explanation**:
- **`SearchFilter`**: DRF's built-in search backend (no extra package needed)
- **`search_fields`**: List of fields to search in
- **Default behavior**: Case-insensitive, partial matching
- **OR logic**: Matches if search term found in ANY field

**Usage:**

```bash
# Search in title and description
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?search=django"
```

**What happens**:
- Searches for "django" in `title` and `desc` fields
- Returns tasks where ANY field contains "django"
- Case-insensitive: "Django", "DJANGO", "django" all match
- Partial match: "Learn Django REST Framework" matches "django"

### Search Lookup Expressions

**Advanced search patterns:**

```python
class TaskViewSet(viewsets.ModelViewSet):
    search_fields = [
        '^title',      # Starts with (e.g., "Learn" matches "Learn Django")
        '=title',      # Exact match (e.g., "Learn Django" only)
        '@desc',       # Full text search (PostgreSQL only)
        '$title',      # Regex search (advanced)
    ]
```

**When to use each:**
- **`^title`**: When you want tasks starting with specific text
- **`=title`**: When you need exact match
- **`@desc`**: PostgreSQL full-text search (most powerful, database-specific)
- **`$title`**: Complex pattern matching (use sparingly)

**Most common**: Just use field name without prefix for simple case-insensitive search.

## Ordering

### What is Ordering?

**Ordering** controls the sequence in which results are returned.

**The Problem**: Without ordering, results come in unpredictable order (usually database insertion order).

**Why Ordering?**
- **User experience**: Show most relevant/newest first
- **Consistency**: Same query always returns same order
- **Flexibility**: Let users choose sort order
- **Default behavior**: Set sensible default (newest first, alphabetical, etc.)

**Real-world analogy**:
- **No ordering** = Bookshelf with random order
- **With ordering** = Bookshelf organized by date (newest first) or alphabetically

### OrderingFilter

**How it works**:
1. Client specifies order: `?ordering=-created_at`
2. DRF sorts queryset by specified field
3. `-` prefix means descending (newest first)
4. No prefix means ascending (oldest first)

```python
# api/views.py
from rest_framework import viewsets, filters

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
    filter_backends = [filters.OrderingFilter]  # Enable ordering
    ordering_fields = ['title', 'completed', 'created_at', 'updated_at']  # Allowed fields
    ordering = ['-created_at']  # Default ordering (newest first)
```

**Explanation**:
- **`OrderingFilter`**: DRF's built-in ordering backend
- **`ordering_fields`**: Which fields users can sort by (security: prevents sorting by sensitive fields)
- **`ordering`**: Default order if user doesn't specify
- **`-` prefix**: Descending order (newest/highest first)
- **No prefix**: Ascending order (oldest/lowest first)

**Usage:**

```bash
# Order by title (ascending: A-Z)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=title"
```

**What happens**: Tasks sorted alphabetically by title (A to Z)

```bash
# Order by created_at (descending: newest first)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=-created_at"
```

**What happens**: Tasks sorted by creation date, newest first (most recent at top)

```bash
# Multiple fields: First by completed status, then by created_at (descending)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=completed,-created_at"
```

**What happens**: 
1. First sorts by completed (False first, then True)
2. Within same completion status, sorts by created_at (newest first)
3. Useful for showing incomplete tasks first, then completed

## Pagination

### What is Pagination?

**Pagination** splits large result sets into smaller "pages" of data.

**The Problem**: Without pagination, returning 10,000 records causes:
- **Slow response**: Large JSON payload
- **High memory**: Server/client must handle all data
- **Poor UX**: User overwhelmed with data
- **Database load**: Fetching all records at once

**Why Pagination?**
- **Performance**: Faster responses (only fetch what's needed)
- **User experience**: Manageable chunks of data
- **Scalability**: Works with millions of records
- **Bandwidth**: Less data transferred
- **Mobile-friendly**: Smaller payloads for mobile networks

**Real-world analogy**:
- **No pagination** = Dumping entire library catalog on one page
- **With pagination** = Showing 10 books per page with "Next" button

**Common Pagination Types**:
1. **Page Number**: `?page=2` (most common, user-friendly)
2. **Limit/Offset**: `?limit=20&offset=40` (flexible, good for APIs)
3. **Cursor**: `?cursor=abc123` (best for large datasets, stable)

### PageNumberPagination

**How it works**:
1. Client requests page: `?page=2`
2. Server calculates which records to return (records 11-20 for page 2)
3. Returns page data + metadata (total count, next/previous links)
4. Client can navigate to next/previous pages

**Why Page Number?**
- **User-friendly**: Easy to understand ("Go to page 2")
- **Simple**: Just specify page number
- **Good for**: Web UIs, predictable navigation
- **Limitation**: Can be slow with very large datasets (offset calculation)

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,  # Records per page
}
```

**Explanation**:
- **`DEFAULT_PAGINATION_CLASS`**: Applies to all list endpoints
- **`PAGE_SIZE`**: How many records per page
- **Can override**: Per-view if needed

**Response format:**

```json
{
    "count": 100,  // Total number of records
    "next": "http://localhost:8000/api/tasks/?page=2",  // URL for next page
    "previous": null,  // URL for previous page (null if on first page)
    "results": [...]  // Actual data (10 records)
}
```

**Why this format?**
- **`count`**: Total records (for progress indicators: "Showing 1-10 of 100")
- **`next`**: Direct link to next page (convenient for clients)
- **`previous`**: Direct link to previous page
- **`results`**: Actual data array

**Usage:**

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?page=2"
```

**What happens**: Returns records 11-20 (page 2, assuming 10 per page)

**Example response:**
```json
{
    "count": 50,
    "next": "http://localhost:8000/api/tasks/?page=3",
    "previous": "http://localhost:8000/api/tasks/?page=1",
    "results": [
        {"id": 11, "title": "Task 11", ...},
        {"id": 12, "title": "Task 12", ...},
        ...
    ]
}
```

### LimitOffsetPagination

```python
# api/pagination.py
from rest_framework.pagination import LimitOffsetPagination

class BookLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 10
    limit_query_param = 'limit'
    offset_query_param = 'offset'
    max_limit = 100
```

```python
# api/views.py
from .pagination import BookLimitOffsetPagination

class BookViewSet(viewsets.ModelViewSet):
    pagination_class = BookLimitOffsetPagination
    # ...
```

**Usage:**

```bash
curl http://localhost:8000/api/books/?limit=20&offset=40
```

### CursorPagination

Best for large datasets. Provides stable, consistent pagination.

```python
# api/pagination.py
from rest_framework.pagination import CursorPagination

class BookCursorPagination(CursorPagination):
    page_size = 10
    ordering = '-created_at'
    cursor_query_param = 'cursor'
```

## Step-by-Step: Secure Book API

**Note**: This section uses Book API as an example of content that can be public or user-owned. The same patterns apply to Task API or any other model.

Let's upgrade the Book API with authentication and filtering.

### Step 1: Update Book Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True, blank=True, null=True)
    description = models.TextField(blank=True)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='books')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"{self.title} by {self.author}"

    class Meta:
        ordering = ['-created_at']
```

**Create migration:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Update Book Serializer

```python
# api/serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    owner_username = serializers.CharField(source='owner.username', read_only=True)

    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_date', 'isbn', 'description', 
                  'owner', 'owner_username', 'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']
```

### Step 3: Create Filter

```python
# api/filters.py
import django_filters
from .models import Book

class BookFilter(django_filters.FilterSet):
    author = django_filters.CharFilter(lookup_expr='icontains')
    published_after = django_filters.DateFilter(field_name='published_date', lookup_expr='gte')
    published_before = django_filters.DateFilter(field_name='published_date', lookup_expr='lte')

    class Meta:
        model = Book
        fields = ['author', 'published_after', 'published_before']
```

### Step 4: Update Book ViewSet

```python
# api/views.py
from rest_framework import viewsets, filters
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from django_filters.rest_framework import DjangoFilterBackend
from .models import Book
from .serializers import BookSerializer
from .filters import BookFilter
from .permissions import IsOwnerOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = BookFilter
    search_fields = ['title', 'author', 'description']
    ordering_fields = ['title', 'author', 'published_date', 'created_at']
    ordering = ['-created_at']

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Step 5: Test Secure Book API

```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}' \
  | python -c "import sys, json; print(json.load(sys.stdin)['access'])")

# Create book (authenticated)
curl -X POST http://localhost:8000/api/books/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Django for Beginners",
    "author": "John Doe",
    "published_date": "2023-01-15",
    "description": "Learn Django step by step"
  }'

# List books (anyone can read)
curl http://localhost:8000/api/books/

# Filter by author
curl "http://localhost:8000/api/books/?author=John"

# Search
curl "http://localhost:8000/api/books/?search=django"

# Order by title
curl "http://localhost:8000/api/books/?ordering=title"
```

## Step-by-Step: Secure Task API

**Note**: This section focuses on Task API as the primary example (consistent with Level 1). Tasks are typically private, user-owned data, making them perfect for demonstrating authentication and permissions.

### Step 1: Update Task Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Task(models.Model):
    title = models.CharField(max_length=100)
    desc = models.TextField(null=True, blank=True)
    completed = models.BooleanField(default=False)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']
```

**Create migration:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Update Task Serializer

```python
# api/serializers.py
class TaskSerializer(serializers.ModelSerializer):
    owner_username = serializers.CharField(source='owner.username', read_only=True)

    class Meta:
        model = Task
        fields = ['id', 'title', 'desc', 'completed', 'owner', 'owner_username', 
                  'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']
```

### Step 3: Create Task Filter

```python
# api/filters.py
import django_filters
from .models import Task

class TaskFilter(django_filters.FilterSet):
    completed = django_filters.BooleanFilter()
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')

    class Meta:
        model = Task
        fields = ['completed', 'created_after', 'created_before']
```

### Step 4: Update Task ViewSet

```python
# api/views.py
from rest_framework import viewsets, filters
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend
from .models import Task
from .serializers import TaskSerializer
from .filters import TaskFilter

class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = TaskFilter
    search_fields = ['title', 'desc']
    ordering_fields = ['title', 'completed', 'created_at']
    ordering = ['-created_at']

    def get_queryset(self):
        # Users can only see their own tasks
        return Task.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Step 5: Test Secure Task API

```bash
# Create task (authenticated)
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Learn DRF Level 2",
    "desc": "Complete authentication and permissions",
    "completed": false
  }'

# List my tasks
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/

# Filter by completed
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false"

# Search tasks
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?search=DRF"
```

## UserProfile API

### Step 1: Create UserProfile Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    bio = models.TextField(blank=True)
    phone_number = models.CharField(max_length=15, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    website = models.URLField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"{self.user.username}'s Profile"
```

### Step 2: Create UserProfile Serializer

```python
# api/serializers.py
class UserProfileSerializer(serializers.ModelSerializer):
    username = serializers.CharField(source='user.username', read_only=True)
    email = serializers.EmailField(source='user.email', read_only=True)

    class Meta:
        model = UserProfile
        fields = ['id', 'username', 'email', 'bio', 'phone_number', 
                  'avatar', 'website', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

### Step 3: Create UserProfile ViewSet

```python
# api/views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import UserProfile
from .serializers import UserProfileSerializer

class UserProfileViewSet(viewsets.ModelViewSet):
    serializer_class = UserProfileSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return UserProfile.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

## Throttling

### Configure Throttling

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

### Custom Throttling

```python
# api/throttles.py
from rest_framework.throttling import UserRateThrottle

class BookCreateThrottle(UserRateThrottle):
    rate = '5/day'
```

```python
# api/views.py
from .throttles import BookCreateThrottle

class BookViewSet(viewsets.ModelViewSet):
    throttle_classes = [BookCreateThrottle]
    # ...
```

## Exception Handling

### What is Exception Handling?

**Exception Handling** allows you to customize how errors are returned to clients.

**The Problem**: By default, DRF returns errors in its own format. You might want:
- Consistent error format across all endpoints
- More user-friendly error messages
- Additional error details
- Custom error codes

**Why Custom Exception Handling?**
- **Consistency**: All errors follow same format
- **User experience**: Clear, helpful error messages
- **Debugging**: Include useful debugging information
- **Security**: Don't expose sensitive details
- **Professional**: Production-ready error responses

**Real-world analogy**:
- **Default errors** = Generic error messages (not helpful)
- **Custom errors** = Clear, actionable error messages (helpful)

### Basic Exception Handler

**How it works**:
1. Exception occurs in view
2. DRF calls your custom exception handler
3. Handler processes exception
4. Returns custom error response

```python
# api/exceptions.py (create this file)
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status
import logging

logger = logging.getLogger(__name__)

def custom_exception_handler(exc, context):
    """
    Custom exception handler for DRF.
    Returns consistent error format for all exceptions.
    """
    # Call DRF's default exception handler first
    response = exception_handler(exc, context)
    
    # If DRF handled it, customize the response
    if response is not None:
        custom_response_data = {
            'success': False,
            'error': {
                'status_code': response.status_code,
                'message': get_error_message(response.status_code, exc),
                'details': response.data,
                'type': exc.__class__.__name__
            }
        }
        response.data = custom_response_data
        
        # Log error for debugging
        logger.error(f"Error {response.status_code}: {exc}", exc_info=True)
    
    # Handle exceptions DRF doesn't catch
    else:
        # For unexpected errors (500)
        custom_response_data = {
            'success': False,
            'error': {
                'status_code': 500,
                'message': 'An unexpected error occurred',
                'details': str(exc) if settings.DEBUG else 'Internal server error',
                'type': exc.__class__.__name__
            }
        }
        response = Response(custom_response_data, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
        
        # Log unexpected errors
        logger.exception(f"Unexpected error: {exc}")
    
    return response

def get_error_message(status_code, exc):
    """Get user-friendly error message based on status code"""
    messages = {
        400: 'Invalid request data',
        401: 'Authentication required',
        403: 'Permission denied',
        404: 'Resource not found',
        405: 'Method not allowed',
        429: 'Too many requests',
        500: 'Internal server error',
    }
    return messages.get(status_code, 'An error occurred')
```

**Add to settings.py:**

```python
# core/settings.py
REST_FRAMEWORK = {
    # ... other settings ...
    'EXCEPTION_HANDLER': 'api.exceptions.custom_exception_handler',
}
```

**Why this location?**
- Centralized: All errors go through one handler
- Consistent: Same format for all errors
- Maintainable: Update error format in one place

### Handling Authentication Errors

**Customize authentication error responses:**

```python
# api/exceptions.py
from rest_framework.exceptions import AuthenticationFailed, NotAuthenticated
from rest_framework import status

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # Handle authentication errors specifically
        if isinstance(exc, (AuthenticationFailed, NotAuthenticated)):
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': 'Authentication failed. Please provide valid credentials.',
                    'details': {
                        'code': 'authentication_failed',
                        'hint': 'Make sure you include a valid token in the Authorization header'
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
        
        # Handle permission errors
        elif isinstance(exc, PermissionDenied):
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': 'You do not have permission to perform this action',
                    'details': {
                        'code': 'permission_denied',
                        'required_permission': str(exc)
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
        
        # Handle other errors
        else:
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': get_error_message(response.status_code, exc),
                    'details': response.data,
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
    
    return response
```

**Why handle authentication errors separately?**
- **User-friendly**: Clear message about what went wrong
- **Actionable**: Tells user how to fix it
- **Security**: Doesn't expose sensitive details
- **Consistent**: Same format for all auth errors

### Handling Validation Errors

**Customize validation error format:**

```python
# api/exceptions.py
from rest_framework.exceptions import ValidationError

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # Handle validation errors
        if isinstance(exc, ValidationError):
            # Format field errors nicely
            formatted_errors = {}
            if isinstance(response.data, dict):
                for field, errors in response.data.items():
                    if isinstance(errors, list):
                        formatted_errors[field] = errors[0] if errors else 'Invalid value'
                    else:
                        formatted_errors[field] = str(errors)
            
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': 'Validation failed',
                    'details': {
                        'code': 'validation_error',
                        'fields': formatted_errors
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
        
        # ... handle other errors ...
    
    return response
```

**Why format validation errors?**
- **Clarity**: Each field error is clear
- **User-friendly**: Easy to understand what's wrong
- **Actionable**: User knows which fields to fix

### Handling JWT Token Errors

**Specific handling for JWT errors:**

```python
# api/exceptions.py
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # Handle JWT token errors
        if isinstance(exc, (InvalidToken, TokenError)):
            error_message = str(exc)
            
            # Provide helpful messages based on error
            if 'expired' in error_message.lower():
                message = 'Token has expired. Please refresh your token.'
                code = 'token_expired'
            elif 'invalid' in error_message.lower():
                message = 'Invalid token. Please login again.'
                code = 'token_invalid'
            else:
                message = 'Token error. Please login again.'
                code = 'token_error'
            
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': 401,
                    'message': message,
                    'details': {
                        'code': code,
                        'hint': 'Use /api/token/ to get a new token, or /api/token/refresh/ to refresh'
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
            response.status_code = 401
        
        # ... handle other errors ...
    
    return response
```

**Why handle JWT errors specifically?**
- **Clear guidance**: Tells user how to fix token issues
- **Actionable**: Provides endpoint URLs for token refresh
- **User-friendly**: Explains what went wrong

### Production vs Development Error Messages

**Show detailed errors in development, generic in production:**

```python
# api/exceptions.py
from django.conf import settings

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # In development: show detailed errors
        # In production: show generic errors
        if settings.DEBUG:
            error_details = {
                'message': str(exc),
                'traceback': traceback.format_exc() if hasattr(exc, '__traceback__') else None,
                'context': {
                    'view': context.get('view').__class__.__name__ if context.get('view') else None,
                    'request_method': context.get('request').method if context.get('request') else None,
                }
            }
        else:
            # Production: don't expose sensitive details
            error_details = {
                'message': get_error_message(response.status_code, exc),
                'code': get_error_code(response.status_code)
            }
        
        custom_response_data = {
            'success': False,
            'error': {
                'status_code': response.status_code,
                **error_details,
                'type': exc.__class__.__name__
            }
        }
        response.data = custom_response_data
    
    return response
```

**Why different messages?**
- **Development**: Detailed errors help debugging
- **Production**: Generic errors protect security
- **Best practice**: Never expose stack traces in production

### Complete Example

**Full-featured exception handler:**

```python
# api/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status
from rest_framework.exceptions import (
    AuthenticationFailed,
    NotAuthenticated,
    PermissionDenied,
    ValidationError,
    NotFound,
    MethodNotAllowed
)
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError
from django.conf import settings
import logging
import traceback

logger = logging.getLogger(__name__)

def custom_exception_handler(exc, context):
    """
    Comprehensive exception handler for DRF.
    Provides consistent, user-friendly error responses.
    """
    # Call DRF's default exception handler
    response = exception_handler(exc, context)
    
    # Request object
    request = context.get('request')
    
    if response is not None:
        # Customize based on exception type
        error_data = {
            'success': False,
            'error': {}
        }
        
        # Authentication errors
        if isinstance(exc, (AuthenticationFailed, NotAuthenticated)):
            error_data['error'] = {
                'status_code': 401,
                'message': 'Authentication required',
                'details': {
                    'code': 'authentication_required',
                    'hint': 'Include a valid token in Authorization header: Bearer <token>'
                }
            }
        
        # Permission errors
        elif isinstance(exc, PermissionDenied):
            error_data['error'] = {
                'status_code': 403,
                'message': 'Permission denied',
                'details': {
                    'code': 'permission_denied',
                    'hint': 'You do not have permission to perform this action'
                }
            }
        
        # JWT token errors
        elif isinstance(exc, (InvalidToken, TokenError)):
            error_data['error'] = {
                'status_code': 401,
                'message': 'Invalid or expired token',
                'details': {
                    'code': 'token_error',
                    'hint': 'Get a new token at /api/token/ or refresh at /api/token/refresh/'
                }
            }
        
        # Validation errors
        elif isinstance(exc, ValidationError):
            formatted_errors = format_validation_errors(response.data)
            error_data['error'] = {
                'status_code': 400,
                'message': 'Validation failed',
                'details': {
                    'code': 'validation_error',
                    'fields': formatted_errors
                }
            }
        
        # Not found errors
        elif isinstance(exc, NotFound):
            error_data['error'] = {
                'status_code': 404,
                'message': 'Resource not found',
                'details': {
                    'code': 'not_found',
                    'hint': 'The requested resource does not exist'
                }
            }
        
        # Method not allowed
        elif isinstance(exc, MethodNotAllowed):
            error_data['error'] = {
                'status_code': 405,
                'message': 'Method not allowed',
                'details': {
                    'code': 'method_not_allowed',
                    'allowed_methods': exc.allowed_methods if hasattr(exc, 'allowed_methods') else []
                }
            }
        
        # Generic error
        else:
            error_data['error'] = {
                'status_code': response.status_code,
                'message': get_error_message(response.status_code),
                'details': response.data if settings.DEBUG else {'code': 'error'}
            }
        
        # Add exception type for debugging
        error_data['error']['type'] = exc.__class__.__name__
        
        # Log error
        logger.error(
            f"Error {error_data['error']['status_code']}: {exc}",
            extra={
                'request_path': request.path if request else None,
                'request_method': request.method if request else None,
            },
            exc_info=settings.DEBUG
        )
        
        response.data = error_data['error']
    
    # Handle unexpected errors (500)
    else:
        error_data = {
            'success': False,
            'error': {
                'status_code': 500,
                'message': 'Internal server error',
                'details': {
                    'code': 'server_error',
                    'message': str(exc) if settings.DEBUG else 'An unexpected error occurred'
                },
                'type': exc.__class__.__name__
            }
        }
        
        # Log unexpected errors
        logger.exception(f"Unexpected error: {exc}")
        
        response = Response(error_data['error'], status=status.HTTP_500_INTERNAL_SERVER_ERROR)
    
    return response

def format_validation_errors(errors):
    """Format DRF validation errors into user-friendly format"""
    formatted = {}
    
    if isinstance(errors, dict):
        for field, field_errors in errors.items():
            if isinstance(field_errors, list):
                # Take first error message
                formatted[field] = field_errors[0] if field_errors else 'Invalid value'
            elif isinstance(field_errors, dict):
                # Nested errors
                formatted[field] = format_validation_errors(field_errors)
            else:
                formatted[field] = str(field_errors)
    
    return formatted

def get_error_message(status_code):
    """Get user-friendly error message"""
    messages = {
        400: 'Bad request',
        401: 'Authentication required',
        403: 'Permission denied',
        404: 'Resource not found',
        405: 'Method not allowed',
        429: 'Too many requests',
        500: 'Internal server error',
    }
    return messages.get(status_code, 'An error occurred')
```

**Why this comprehensive handler?**
- **Handles all error types**: Authentication, permissions, validation, etc.
- **User-friendly messages**: Clear, actionable error messages
- **Consistent format**: Same structure for all errors
- **Production-ready**: Different messages for dev/prod
- **Logging**: Tracks all errors for debugging

### Error Response Format

**Standard error response structure:**

```json
{
    "success": false,
    "error": {
        "status_code": 401,
        "message": "Authentication required",
        "details": {
            "code": "authentication_required",
            "hint": "Include a valid token in Authorization header"
        },
        "type": "AuthenticationFailed"
    }
}
```

**Why this format?**
- **`success`**: Quick check if request succeeded
- **`error.status_code`**: HTTP status code
- **`error.message`**: Human-readable message
- **`error.details`**: Additional information
- **`error.type`**: Exception class name (for debugging)

### Testing Exception Handler

**Test your exception handler:**

```python
# api/tests.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth.models import User

class ExceptionHandlerTest(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
    
    def test_authentication_error(self):
        # Request without token
        response = self.client.get('/api/tasks/')
        self.assertEqual(response.status_code, 401)
        self.assertIn('error', response.data)
        self.assertEqual(response.data['error']['details']['code'], 'authentication_required')
    
    def test_validation_error(self):
        # Login first
        self.client.force_authenticate(user=self.user)
        
        # Invalid data - empty title
        response = self.client.post('/api/tasks/', {
            'title': '',  # Empty title should fail validation
            'completed': False
        })
        self.assertEqual(response.status_code, 400)
        self.assertIn('error', response.data)
        self.assertIn('fields', response.data['error']['details'])
        self.assertIn('title', response.data['error']['details']['fields'])
    
    def test_permission_denied(self):
        # Login as regular user
        self.client.force_authenticate(user=self.user)
        
        # Try to access admin-only endpoint (if you have one)
        # This would return 403 if permission denied
        # response = self.client.get('/api/admin/tasks/')
        # self.assertEqual(response.status_code, 403)
```

**Why test exception handler?**
- **Verify format**: Ensure errors follow expected format
- **Catch bugs**: Find issues in error handling
- **Documentation**: Shows how errors are returned

### Best Practices

**1. Consistent Format**
- Use same structure for all errors
- Include status code, message, details
- Make it easy for clients to parse

**2. User-Friendly Messages**
- Avoid technical jargon
- Provide actionable hints
- Explain what went wrong

**3. Security**
- Don't expose sensitive details in production
- Hide stack traces from users
- Log detailed errors server-side

**4. Logging**
- Log all errors for debugging
- Include request context
- Use appropriate log levels

**5. Error Codes**
- Use consistent error codes
- Make codes searchable
- Document all error codes

## Testing Authenticated APIs

See [CURL_GUIDE.md](CURL_GUIDE.md) for detailed examples. Quick reference:

```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}' \
  | jq -r '.access')

# Use token
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/

# Refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "your_refresh_token"}'
```

## Exercises

### Exercise 1: Complete Secure Book API

1. Add owner field to Book model
2. Implement IsOwnerOrReadOnly permission
3. Add filtering by author and date range
4. Add search functionality
5. Add pagination
6. Test all endpoints with cURL

### Exercise 2: UserProfile API

1. Create UserProfile model
2. Create UserProfileSerializer and ViewSet
3. Allow users to view/edit only their own profile
4. Test with cURL

### Exercise 3: Task Filtering

1. Add priority field to Task model
2. Create TaskFilter with priority filtering
3. Add ordering by priority
4. Test filtering and ordering

## Add-ons

### Add-on 1: Custom JWT Claims

```python
# api/serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token['username'] = user.username
        token['email'] = user.email
        return token
```

```python
# api/views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import CustomTokenObtainPairSerializer

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

### Add-on 2: Rate Limiting

Implement different rate limits for different actions.

### Add-on 3: Permission Mixins

Create reusable permission mixins for common patterns.

## Common Errors and Solutions

### Error 1: `rest_framework.exceptions.AuthenticationFailed: Given token not valid for any token type`

**Error Message**:
```
rest_framework.exceptions.AuthenticationFailed: Given token not valid for any token type
```

**Why This Happens**:
- Token expired
- Invalid token format
- Token not sent correctly in header
- Wrong token type

**How to Fix**:
```bash
# Check token format
# Should be: Authorization: Bearer <token>
# NOT: Authorization: <token>
# NOT: Authorization: Token <token>

# Correct format
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  http://localhost:8000/api/tasks/

# Get new token if expired
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}'
```

**Prevention**: 
- Always use `Bearer` prefix for JWT tokens
- Check token expiration
- Refresh token before it expires

---

### Error 2: `rest_framework.exceptions.PermissionDenied: You do not have permission to perform this action`

**Error Message**:
```
rest_framework.exceptions.PermissionDenied: You do not have permission to perform this action
```

**Why This Happens**:
- User not authenticated (no token/session)
- User authenticated but lacks required permissions
- Permission class too restrictive

**How to Fix**:
```python
# Check if user is authenticated
# In view, check:
print(request.user)  # Should be User object, not AnonymousUser
print(request.user.is_authenticated)  # Should be True

# Check permissions
# Option 1: Make endpoint public (if appropriate)
class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [AllowAny]  # Anyone can access (rare for tasks)

# Option 2: Require authentication (most common for tasks)
class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]  # Must be logged in

# Option 3: Check if user has permission
if not request.user.has_perm('api.change_task'):
    return Response({'error': 'Permission denied'}, status=403)
```

**Prevention**: 
- Check permission classes in view
- Verify user is authenticated before accessing protected endpoints
- Test with different user roles

---

### Error 3: `django.contrib.auth.models.User.DoesNotExist: User matching query does not exist`

**Error Message**:
```
django.contrib.auth.models.User.DoesNotExist: User matching query does not exist
```

**Why This Happens**:
- Trying to get user that doesn't exist
- User ID in token doesn't match any user
- User was deleted but token still valid

**How to Fix**:
```python
# Option 1: Check if user exists
try:
    user = User.objects.get(id=user_id)
except User.DoesNotExist:
    return Response({'error': 'User not found'}, status=404)

# Option 2: Use get_object_or_404
from django.shortcuts import get_object_or_404
user = get_object_or_404(User, id=user_id)

# Option 3: Handle in JWT token validation
# In custom token serializer, verify user exists
def validate(self, attrs):
    user = authenticate(**attrs)
    if not user or not user.is_active:
        raise serializers.ValidationError('User not found or inactive')
    return attrs
```

**Prevention**: Always verify user exists before using, especially in token validation.

---

### Error 4: `django_filters.exceptions.FieldLookupError: Unsupported lookup 'icontains' for CharField`

**Error Message**:
```
django_filters.exceptions.FieldLookupError: Unsupported lookup 'icontains' for CharField
```

**Why This Happens**:
- Using wrong lookup expression in FilterSet
- Lookup not supported for field type
- Typo in lookup expression

**How to Fix**:
```python
# WRONG - icontains not available in basic filterset_fields
class TaskViewSet(viewsets.ModelViewSet):
    filterset_fields = ['title__icontains']  # ❌ Not supported

# CORRECT - Use FilterSet class
class TaskFilter(django_filters.FilterSet):
    title = django_filters.CharFilter(lookup_expr='icontains')  # ✅
    
    class Meta:
        model = Task
        fields = ['title']

# In ViewSet
class TaskViewSet(viewsets.ModelViewSet):
    filterset_class = TaskFilter  # ✅
```

**Prevention**: Use FilterSet class for advanced lookups, not `filterset_fields`.

---

### Error 5: `rest_framework.exceptions.ValidationError: {'refresh': ['This field is required.']}`

**Error Message**:
```
rest_framework.exceptions.ValidationError: {'refresh': ['This field is required.']}
```

**Why This Happens**:
- Trying to refresh token without providing refresh token
- Sending access token instead of refresh token
- Missing refresh token in request

**How to Fix**:
```bash
# WRONG - Using access token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "access_token_here"}'  # ❌ Wrong token type

# CORRECT - Use refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "refresh_token_here"}'  # ✅ Correct

# Get tokens first
TOKENS=$(curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}')

# Extract refresh token
REFRESH=$(echo $TOKENS | jq -r '.refresh')

# Use refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d "{\"refresh\": \"$REFRESH\"}"
```

**Prevention**: 
- Store both access and refresh tokens
- Use refresh token only for refreshing
- Don't confuse access and refresh tokens

---

### Error 6: `django.core.exceptions.FieldError: Related Field got invalid lookup: icontains`

**Error Message**:
```
django.core.exceptions.FieldError: Related Field got invalid lookup: icontains
```

**Why This Happens**:
- Using wrong lookup on ForeignKey field
- Need to use double underscore for related fields

**How to Fix**:
```python
# WRONG - Can't use icontains directly on ForeignKey
Task.objects.filter(owner__icontains='john')  # ❌

# CORRECT - Use double underscore to access related field
Task.objects.filter(owner__username__icontains='john')  # ✅

# Or in FilterSet
class TaskFilter(django_filters.FilterSet):
    owner_username = django_filters.CharFilter(
        field_name='owner__username',
        lookup_expr='icontains'
    )
    
    class Meta:
        model = Task
        fields = ['owner_username']
```

**Prevention**: Remember to use `__` (double underscore) to traverse relationships.

---

### Error 7: `rest_framework.exceptions.NotFound: Invalid page.`

**Error Message**:
```
rest_framework.exceptions.NotFound: Invalid page.
```

**Why This Happens**:
- Requesting page number that doesn't exist
- Page number out of range
- Invalid page parameter

**How to Fix**:
```python
# Check total pages first
# Response includes: {"count": 100, "next": "...", "previous": "..."}

# WRONG - Page 999 when only 10 pages exist
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?page=999"  # ❌

# CORRECT - Check count first, then request valid page
# If count=100 and page_size=10, valid pages are 1-10

# Handle in frontend
response = fetch('/api/tasks/?page=1', {
    headers: { 'Authorization': `Bearer ${token}` }
})
data = await response.json()
total_pages = Math.ceil(data.count / page_size)  # Calculate total pages
if (page > total_pages) {
    // Show error or redirect to last page
}
```

**Prevention**: 
- Always check `count` in pagination response
- Calculate total pages: `ceil(count / page_size)`
- Validate page number before requesting

---

### Error 8: `django.db.utils.IntegrityError: NOT NULL constraint failed: api_task.owner_id`

**Error Message**:
```
django.db.utils.IntegrityError: NOT NULL constraint failed: api_task.owner_id
```

**Why This Happens**:
- Trying to create object without required ForeignKey
- Owner not set in serializer
- `perform_create` not setting owner

**How to Fix**:
```python
# WRONG - Owner not set
def create(self, request):
    serializer = TaskSerializer(data=request.data)
    serializer.save()  # ❌ Owner not set!

# CORRECT - Set owner in perform_create
class TaskViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)  # ✅

# Or in serializer
class TaskSerializer(serializers.ModelSerializer):
    def create(self, validated_data):
        validated_data['owner'] = self.context['request'].user
        return super().create(validated_data)
```

**Prevention**: Always set owner in `perform_create` or serializer's `create` method.

---

### Error 9: `rest_framework.exceptions.MethodNotAllowed: Method 'PATCH' not allowed`

**Error Message**:
```
rest_framework.exceptions.MethodNotAllowed: Method 'PATCH' not allowed
```

**Why This Happens**:
- View doesn't support PATCH method
- Using wrong HTTP method
- ViewSet action not configured

**How to Fix**:
```python
# Check what methods are allowed
# ViewSet automatically supports: GET, POST, PUT, PATCH, DELETE

# If using APIView, must define method
class TaskAPIView(APIView):
    def patch(self, request, pk):  # ✅ Define PATCH method
        # ...

# If using ViewSet, PATCH is automatic
class TaskViewSet(viewsets.ModelViewSet):  # ✅ Supports PATCH automatically
    # ...
```

**Prevention**: 
- Use ViewSet for automatic method support
- Or explicitly define methods in APIView
- Check allowed methods in view

---

### Error 10: `django_filters.exceptions.FieldError: Meta.fields contains fields that are not defined on this FilterSet`

**Error Message**:
```
django_filters.exceptions.FieldError: Meta.fields contains fields that are not defined on this FilterSet
```

**Why This Happens**:
- Field in `Meta.fields` not defined in FilterSet
- Typo in field name
- Field doesn't exist in model

**How to Fix**:
```python
# WRONG - Field 'title_contains' not defined
class TaskFilter(django_filters.FilterSet):
    # title_contains not defined here!
    
    class Meta:
        model = Task
        fields = ['title_contains']  # ❌ Field not defined

# CORRECT - Define field first
class TaskFilter(django_filters.FilterSet):
    title_contains = django_filters.CharFilter(
        field_name='title',
        lookup_expr='icontains'
    )
    
    class Meta:
        model = Task
        fields = ['title_contains']  # ✅ Now defined
```

**Prevention**: Always define custom filter fields before listing them in `Meta.fields`.

---

### General Debugging Tips for Level 2

**1. Check Authentication**
```python
# In view, verify authentication
print("User:", request.user)
print("Authenticated:", request.user.is_authenticated)
print("Is anonymous:", isinstance(request.user, AnonymousUser))
```

**2. Test Permissions**
```python
# Check what permissions user has
print("Permissions:", request.user.get_all_permissions())
print("Is staff:", request.user.is_staff)
print("Is superuser:", request.user.is_superuser)
```

**3. Verify Token**
```python
# Decode JWT token to see contents
from rest_framework_simplejwt.tokens import AccessToken

token = request.META.get('HTTP_AUTHORIZATION', '').split(' ')[1]
decoded = AccessToken(token)
print("Token payload:", decoded.payload)
```

**4. Test Filters**
```bash
# Test each filter separately
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false"
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?search=django"
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=-created_at"
```

**5. Check Pagination**
```python
# Verify pagination settings
from rest_framework.pagination import PageNumberPagination
pagination = PageNumberPagination()
print("Page size:", pagination.page_size)
```

**6. Common Authentication Issues**
- **Token expired**: Get new token or refresh
- **Wrong header format**: Use `Authorization: Bearer <token>`
- **Token not sent**: Check request headers
- **User inactive**: Check `user.is_active`

**7. Common Permission Issues**
- **Too restrictive**: Check permission classes
- **User not authenticated**: Verify token/session
- **Wrong user**: Check if user owns resource
- **Missing permission**: Verify user has required permission

## Next Steps

Congratulations! You've completed Level 2. You now know:

- ✅ Django authentication system
- ✅ JWT authentication setup
- ✅ Permissions (built-in and custom)
- ✅ Filtering, searching, and ordering
- ✅ Pagination strategies
- ✅ Building secure APIs

**Ready for Level 3?** Continue to [Level 3: Advanced](LEVEL_3_ADVANCED.md) to learn about relationships, nested serializers, and query optimization!

---

**Resources:**
- [DRF Authentication](https://www.django-rest-framework.org/api-guide/authentication/)
- [DRF Permissions](https://www.django-rest-framework.org/api-guide/permissions/)
- [djangorestframework-simplejwt](https://github.com/jazzband/djangorestframework-simplejwt)
- [django-filter](https://django-filter.readthedocs.io/)

