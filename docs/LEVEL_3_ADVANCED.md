# Level 3: Advanced - Relationships, Serializers, and Complex Logic

## Goal

Build APIs with nested, related data and business logic. Master model relationships, nested serializers, query optimization, and background tasks. By the end of this level, you'll be able to build complex, efficient APIs.

## Note on Examples

This guide uses **two different API examples** to demonstrate different relationship patterns and advanced concepts:

- **Blog API (Post/Comment/Tag)**: Used in concept explanations and the main step-by-step section to demonstrate:
  - Complex relationships (ForeignKey, ManyToMany)
  - Nested serializers with multiple related models
  - Query optimization across relationships
  - Real-world content management patterns
  
- **Enhanced Task API**: Used in a separate step-by-step section to show:
  - How to extend the Task API from Level 2 with relationships
  - Adding categories, priorities, and assigned users
  - Building on previous knowledge incrementally
  - Practical task management features

**Why two examples?**
- **Blog API** is ideal for demonstrating complex relationships (posts have comments, tags, authors)
- **Enhanced Task API** builds on Level 2's Task API, showing progression
- Different examples help you understand patterns that apply to any model
- You can apply all concepts to your own projects (Task, Book, Product, Post, etc.)

## Table of Contents

1. [Model Relationships](#model-relationships)
2. [Nested Serializers](#nested-serializers)
3. [SerializerMethodField](#serializermethodfield)
4. [Custom Serializer Fields](#custom-serializer-fields)
5. [Overriding Serializer Methods](#overriding-serializer-methods)
6. [Validation](#validation)
7. [Query Optimization](#query-optimization)
8. [Django Signals](#django-signals)
9. [Background Tasks](#background-tasks)
10. [Step-by-Step: Blog API](#step-by-step-blog-api)
11. [Step-by-Step: Enhanced Task API](#step-by-step-enhanced-task-api)
12. [Soft Delete](#soft-delete)
13. [Advanced Query Patterns](#advanced-query-patterns)
14. [Exercises](#exercises)
15. [Add-ons](#add-ons)

## Model Relationships

### ForeignKey (Many-to-One)

One model references another. Example: Many Posts belong to One User.

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

**Access patterns:**

```python
# Get all posts by a user
user.posts.all()

# Get author of a post
post.author
```

### OneToOneField (One-to-One)

One-to-one relationship. Example: One User has One Profile.

```python
# api/models.py
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    bio = models.TextField(blank=True)
    phone_number = models.CharField(max_length=15, blank=True)
```

**Access patterns:**

```python
# Get user's profile
user.profile

# Get profile's user
profile.user
```

### ManyToManyField (Many-to-Many)

Many-to-many relationship. Example: Posts can have many Tags, Tags can be on many Posts.

```python
# api/models.py
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    tags = models.ManyToManyField(Tag, related_name='posts', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

**Access patterns:**

```python
# Add tag to post
post.tags.add(tag)

# Get all tags for a post
post.tags.all()

# Get all posts with a tag
tag.posts.all()

# Remove tag
post.tags.remove(tag)

# Clear all tags
post.tags.clear()
```

### Related Managers

Django automatically creates related managers for relationships:

```python
# ForeignKey creates reverse manager
user.posts.all()  # All posts by user
user.posts.filter(title__icontains='django')

# ManyToMany creates manager
post.tags.all()
post.tags.add(tag1, tag2)
```

### on_delete Options

```python
# CASCADE: Delete related objects
author = models.ForeignKey(User, on_delete=models.CASCADE)

# PROTECT: Prevent deletion if related objects exist
author = models.ForeignKey(User, on_delete=models.PROTECT)

# SET_NULL: Set to NULL (requires null=True)
author = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)

# SET_DEFAULT: Set to default value
author = models.ForeignKey(User, on_delete=models.SET_DEFAULT, default=1)

# DO_NOTHING: Do nothing (not recommended)
author = models.ForeignKey(User, on_delete=models.DO_NOTHING)
```

## Nested Serializers

### Read-Only Nested Serializer

Display related data but don't allow creation/update through nested data.

```python
# api/serializers.py
from rest_framework import serializers
from .models import Post, User, Tag

class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name']

class PostSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    author_id = serializers.IntegerField(write_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'author_id', 'created_at']
        read_only_fields = ['id', 'created_at']

    def create(self, validated_data):
        author_id = validated_data.pop('author_id', None)
        if author_id:
            validated_data['author_id'] = author_id
        return Post.objects.create(**validated_data)
```

### Writable Nested Serializer

Allow creating/updating related objects through the parent.

**⚠️ Important**: When using nested serializers with ManyToMany fields, ensure the instance is saved before accessing ManyToMany relationships to avoid `AttributeError: 'NoneType' object has no attribute '_meta'`.

```python
# api/serializers.py
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name']

class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'content', 'author', 'created_at']
        read_only_fields = ['id', 'created_at']

class PostSerializer(serializers.ModelSerializer):
    # Overriding model's ManyToMany 'tags' field with nested serializer
    tags = TagSerializer(many=True, required=False)
    comments = CommentSerializer(many=True, read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'tags', 'comments', 'created_at']
        read_only_fields = ['id', 'author', 'created_at']

    def create(self, validated_data):
        tags_data = validated_data.pop('tags', [])
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            raise serializers.ValidationError("Authentication required to create posts")
        # CRITICAL: Create instance first, then add ManyToMany relationships
        # Author is read-only, so set it from request
        post = Post.objects.create(author=request.user, **validated_data)
        
        # Now safe to add ManyToMany relationships (instance is saved)
        # tags_data is a list of dicts: [{'name': 'django'}, {'name': 'python'}]
        for tag_data in tags_data:
            tag, created = Tag.objects.get_or_create(name=tag_data['name'])
            post.tags.add(tag)
        
        return post

    def update(self, instance, validated_data):
        tags_data = validated_data.pop('tags', None)
        
        # Update post fields
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        # CRITICAL: Save instance before updating ManyToMany
        instance.save()
        
        # Update tags if provided (after instance is saved)
        if tags_data is not None:
            instance.tags.clear()
            for tag_data in tags_data:
                tag, created = Tag.objects.get_or_create(name=tag_data['name'])
                instance.tags.add(tag)
        
        return instance
```

### Nested Serializer Best Practices

1. **Use read_only=True** for display-only nested data
2. **Override create() and update()** for writable nested serializers
3. **Use get_or_create()** to avoid duplicate related objects
4. **Handle ManyToMany** relationships carefully in update()

### Important: ManyToMany Field Conflicts

**⚠️ Common Error**: When overriding a ManyToMany field with a nested serializer using the same field name, you may encounter:
```
AttributeError: 'NoneType' object has no attribute '_meta'
```

**Solution Options**:

**Option 1: Separate Read/Write Fields (Recommended)**
Use different field names for reading (nested serializer) and writing (ID list):

```python
class TaskSerializer(serializers.ModelSerializer):
    # Read: nested serializer for display
    assigned_to = UserSerializer(many=True, read_only=True)
    # Write: list of IDs for creation/update
    assigned_to_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    
    def create(self, validated_data):
        assigned_to_ids = validated_data.pop('assigned_to_ids', [])
        task = Task.objects.create(**validated_data)
        if assigned_to_ids:
            task.assigned_to.set(assigned_to_ids)  # Set after creation
        return task
```

**Option 2: Nested Serializer (Use with Caution)**
If using the same field name, ensure the instance is saved before accessing ManyToMany:

```python
class PostSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True, required=False)  # Same name as model field
    
    def create(self, validated_data):
        tags_data = validated_data.pop('tags', [])
        # CRITICAL: Create instance first, then add ManyToMany relationships
        post = Post.objects.create(**validated_data)
        # Now safe to add ManyToMany relationships
        for tag_data in tags_data:
            tag, created = Tag.objects.get_or_create(name=tag_data['name'])
            post.tags.add(tag)
        return post
```

**Why Option 1 is Better**: Avoids field name conflicts and is more explicit about read vs. write operations.

## SerializerMethodField

Use SerializerMethodField to add computed or custom fields.

```python
# api/serializers.py
class PostSerializer(serializers.ModelSerializer):
    author_name = serializers.SerializerMethodField()
    comment_count = serializers.SerializerMethodField()
    is_author = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'author_name', 
                  'comment_count', 'is_author', 'created_at']

    def get_author_name(self, obj):
        return f"{obj.author.first_name} {obj.author.last_name}"

    def get_comment_count(self, obj):
        return obj.comments.count()

    def get_is_author(self, obj):
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False
```

**Important**: SerializerMethodField is read-only by default.

## Custom Serializer Fields

### Creating Custom Fields

```python
# api/serializers.py
from rest_framework import serializers

class UppercaseCharField(serializers.CharField):
    def to_representation(self, value):
        return super().to_representation(value).upper() if value else value

class PostSerializer(serializers.ModelSerializer):
    title = UppercaseCharField()
    # ...
```

### Custom Field with Validation

```python
# api/serializers.py
class AgeField(serializers.IntegerField):
    def to_internal_value(self, data):
        age = super().to_internal_value(data)
        if age < 0 or age > 150:
            raise serializers.ValidationError("Age must be between 0 and 150")
        return age

class UserProfileSerializer(serializers.ModelSerializer):
    age = AgeField()
    # ...
```

## Overriding Serializer Methods

### Override create()

```python
# api/serializers.py
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'tags']

    def create(self, validated_data):
        # Get request user
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            raise serializers.ValidationError("Authentication required to create posts")
        
        # Set author to current user
        validated_data['author'] = request.user
        
        # Handle tags - pop before creating instance
        tags_data = validated_data.pop('tags', [])
        # CRITICAL: Create instance first, then add ManyToMany relationships
        post = Post.objects.create(**validated_data)
        
        # Now safe to add ManyToMany relationships (instance is saved)
        for tag_data in tags_data:
            tag, created = Tag.objects.get_or_create(name=tag_data['name'])
            post.tags.add(tag)
        
        return post
```

### Override update()

```python
# api/serializers.py
class PostSerializer(serializers.ModelSerializer):
    def update(self, instance, validated_data):
        # Update simple fields
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        
        # Handle tags
        if 'tags' in validated_data:
            tags_data = validated_data.pop('tags')
            # CRITICAL: Save instance before updating ManyToMany
            instance.save()
            instance.tags.clear()
            for tag_data in tags_data:
                tag, created = Tag.objects.get_or_create(name=tag_data['name'])
                instance.tags.add(tag)
        else:
            instance.save()
        
        return instance
```

### Override to_representation()

Customize how data is serialized.

```python
# api/serializers.py
class PostSerializer(serializers.ModelSerializer):
    def to_representation(self, instance):
        representation = super().to_representation(instance)
        
        # Add custom fields
        representation['author_full_name'] = f"{instance.author.first_name} {instance.author.last_name}"
        representation['comment_count'] = instance.comments.count()
        
        # Format dates
        representation['created_at'] = instance.created_at.strftime('%Y-%m-%d %H:%M:%S')
        
        return representation
```

## Validation

### Field-Level Validation

```python
# api/serializers.py
class PostSerializer(serializers.ModelSerializer):
    title = serializers.CharField(max_length=200)

    def validate_title(self, value):
        if len(value) < 10:
            raise serializers.ValidationError("Title must be at least 10 characters")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Title cannot contain spam")
        return value
```

### Object-Level Validation

```python
# api/serializers.py
class PostSerializer(serializers.ModelSerializer):
    def validate(self, data):
        # Check if title and content match
        if 'title' in data and 'content' in data:
            if data['title'].lower() in data['content'].lower():
                raise serializers.ValidationError({
                    'content': 'Content should not repeat the title'
                })
        return data
```

### Custom Validators

```python
# api/validators.py
from rest_framework import serializers

def validate_no_profanity(value):
    profanity_words = ['spam', 'badword']
    if any(word in value.lower() for word in profanity_words):
        raise serializers.ValidationError("Content contains inappropriate words")
    return value

# api/serializers.py
from .validators import validate_no_profanity

class PostSerializer(serializers.ModelSerializer):
    content = serializers.CharField(validators=[validate_no_profanity])
    # ...
```

## Query Optimization

### The N+1 Problem

**Problem**: Making multiple database queries when one would suffice.

```python
# BAD: N+1 queries
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # One query per post!
```

**Solution**: Use `select_related` and `prefetch_related`.

### select_related (ForeignKey, OneToOne)

Follows foreign key relationships and fetches related objects in a single query.

```python
# GOOD: Single query
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.username)  # No additional queries!
```

### prefetch_related (ManyToMany, Reverse ForeignKey)

Prefetches related objects in a separate query.

```python
# BAD: Multiple queries
posts = Post.objects.all()
for post in posts:
    print(post.tags.all())  # One query per post!

# GOOD: Two queries total
posts = Post.objects.prefetch_related('tags').all()
for post in posts:
    print(post.tags.all())  # No additional queries!
```

### Combined Optimization

```python
# Optimize multiple relationships
posts = Post.objects.select_related('author').prefetch_related('tags', 'comments').all()

# With filtering
posts = Post.objects.select_related('author').prefetch_related('tags').filter(
    author__is_active=True
).all()
```

### only() and defer()

Fetch only needed fields.

```python
# Fetch only specific fields
posts = Post.objects.only('id', 'title', 'author_id').all()

# Defer heavy fields
posts = Post.objects.defer('content').all()  # Don't fetch content field
```

### annotate() and aggregate()

Add computed fields and aggregations.

```python
from django.db.models import Count, Avg

# Add comment count to each post
posts = Post.objects.annotate(comment_count=Count('comments')).all()

# Get average comment count
avg_comments = Post.objects.aggregate(avg_comments=Avg('comments__id'))
```

### Using in Views

```python
# api/views.py
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Post.objects.select_related('author').prefetch_related(
            'tags', 'comments', 'comments__author'
        ).annotate(comment_count=Count('comments')).all()
```

## Django Signals

### What are Signals?

Signals allow decoupled applications to get notified when certain actions occur.

### Common Signals

```python
# api/signals.py
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete
from django.dispatch import receiver
from .models import Post

@receiver(pre_save, sender=Post)
def pre_save_post(sender, instance, **kwargs):
    """Called before Post is saved"""
    if not instance.slug:
        instance.slug = slugify(instance.title)

@receiver(post_save, sender=Post)
def post_save_post(sender, instance, created, **kwargs):
    """Called after Post is saved"""
    if created:
        print(f"New post created: {instance.title}")
        # Send notification, update cache, etc.

@receiver(pre_delete, sender=Post)
def pre_delete_post(sender, instance, **kwargs):
    """Called before Post is deleted"""
    # Backup, logging, etc.
    print(f"Post being deleted: {instance.title}")
```

### Register Signals

```python
# api/apps.py
from django.apps import AppConfig

class ApiConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'api'

    def ready(self):
        import api.signals  # Import signals
```

## Background Tasks

### Option 1: Celery (Production)

```bash
# Install
pip install celery redis
```

```python
# core/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core.settings')
app = Celery('core')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

```python
# core/__init__.py
from .celery import app as celery_app
__all__ = ('celery_app',)
```

```python
# core/settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
```

```python
# api/tasks.py
from celery import shared_task
from django.core.mail import send_mail

@shared_task
def send_post_notification(post_id):
    post = Post.objects.get(id=post_id)
    send_mail(
        subject=f'New Post: {post.title}',
        message=post.content,
        from_email='noreply@example.com',
        recipient_list=['admin@example.com'],
    )
```

```python
# api/views.py
from .tasks import send_post_notification

class PostViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        post = serializer.save()
        send_post_notification.delay(post.id)  # Async task
```

### Option 2: Django-Q (Simpler Alternative)

```bash
pip install django-q2
```

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'django_q',
]

Q_CLUSTER = {
    'name': 'DjangORM',
    'workers': 4,
    'timeout': 90,
    'retry': 120,
    'queue_limit': 50,
    'bulk': 10,
    'orm': 'default'
}
```

```python
# api/views.py
from django_q.tasks import async_task

def send_notification(post_id):
    # Your notification logic
    pass

class PostViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        post = serializer.save()
        async_task(send_notification, post.id)  # Async task
```

## Step-by-Step: Blog API

Let's build a complete Blog API with relationships.

### Step 1: Create Models

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    tags = models.ManyToManyField(Tag, related_name='posts', blank=True)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='comments')
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Comment by {self.author.username} on {self.post.title}"

    class Meta:
        ordering = ['created_at']
```

**Create migrations:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Create Serializers

```python
# api/serializers.py
from rest_framework import serializers
from .models import Post, Comment, Tag
from django.contrib.auth.models import User

class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'first_name', 'last_name']

class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name']

class CommentSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    post_id = serializers.IntegerField(write_only=True)

    class Meta:
        model = Comment
        fields = ['id', 'content', 'post_id', 'author', 'created_at']
        read_only_fields = ['id', 'author', 'created_at']

class PostSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    # Custom tags field - this overrides the model's ManyToMany 'tags' field
    # DRF will use this instead of the model field, avoiding the _meta conflict
    tags = TagSerializer(many=True, required=False)
    comments = CommentSerializer(many=True, read_only=True)
    comment_count = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'tags', 'comments', 
                  'comment_count', 'published', 'created_at', 'updated_at']
        read_only_fields = ['id', 'author', 'created_at', 'updated_at']

    def get_comment_count(self, obj):
        return obj.comments.count()

    def create(self, validated_data):
        tags_data = validated_data.pop('tags', [])
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            raise serializers.ValidationError("Authentication required to create posts")
        # Create post first (without tags)
        post = Post.objects.create(author=request.user, **validated_data)
        
        # Handle tags after post is saved (ManyToMany requires saved instance)
        # tags_data is a list of dicts from TagSerializer (e.g., [{'name': 'django'}, {'name': 'python'}])
        for tag_data in tags_data:
            tag, created = Tag.objects.get_or_create(name=tag_data['name'])
            post.tags.add(tag)
        
        return post

    def update(self, instance, validated_data):
        tags_data = validated_data.pop('tags', None)
        
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.published = validated_data.get('published', instance.published)
        instance.save()
        
        if tags_data is not None:
            instance.tags.clear()
            for tag_data in tags_data:
                tag, created = Tag.objects.get_or_create(name=tag_data['name'])
                instance.tags.add(tag)
        
        return instance
```

### Step 3: Create ViewSets

```python
# api/views.py
from rest_framework import viewsets, filters
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from django_filters.rest_framework import DjangoFilterBackend
from django.db.models import Count
from .models import Post, Comment, Tag
from .serializers import PostSerializer, CommentSerializer, TagSerializer

class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['published', 'author']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    ordering = ['-created_at']

    def get_queryset(self):
        return Post.objects.select_related('author').prefetch_related(
            'tags', 'comments', 'comments__author'
        ).annotate(comment_count=Count('comments')).all()

class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        post_id = self.request.query_params.get('post', None)
        queryset = Comment.objects.select_related('author', 'post').all()
        if post_id:
            queryset = queryset.filter(post_id=post_id)
        return queryset

    def perform_create(self, serializer):
        post_id = serializer.validated_data.get('post_id')
        serializer.save(author=self.request.user, post_id=post_id)

class TagViewSet(viewsets.ModelViewSet):
    queryset = Tag.objects.all()
    serializer_class = TagSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [filters.SearchFilter]
    search_fields = ['name']
```

### Step 4: Configure URLs

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'posts', views.PostViewSet, basename='post')
router.register(r'comments', views.CommentViewSet, basename='comment')
router.register(r'tags', views.TagViewSet, basename='tag')

urlpatterns = [
    path('', include(router.urls)),
]
```

### Step 5: Test Blog API

```bash
# Create post with tags
curl -X POST http://localhost:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "My First Post",
    "content": "This is the content",
    "tags": [{"name": "django"}, {"name": "python"}],
    "published": true
  }'

# Get post with nested data
curl http://localhost:8000/api/posts/1/

# Create comment
curl -X POST http://localhost:8000/api/comments/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "post_id": 1,
    "content": "Great post!"
  }'
```

## Step-by-Step: Enhanced Task API

Add relationships to your Task model.

### Step 1: Update Task Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Category(models.Model):
    name = models.CharField(max_length=50, unique=True)
    color = models.CharField(max_length=7, default='#000000')  # Hex color
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

class Task(models.Model):
    PRIORITY_CHOICES = [
        ('low', 'Low'),
        ('medium', 'Medium'),
        ('high', 'High'),
    ]
    
    title = models.CharField(max_length=100)
    desc = models.TextField(null=True, blank=True)
    completed = models.BooleanField(default=False)
    priority = models.CharField(max_length=10, choices=PRIORITY_CHOICES, default='medium')
    due_date = models.DateTimeField(null=True, blank=True)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True, blank=True, related_name='tasks')
    assigned_to = models.ManyToManyField(User, related_name='assigned_tasks', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']
```

### Step 2: Create Serializers

```python
# api/serializers.py
class CategorySerializer(serializers.ModelSerializer):
    task_count = serializers.SerializerMethodField()

    class Meta:
        model = Category
        fields = ['id', 'name', 'color', 'task_count', 'created_at']
        read_only_fields = ['id', 'created_at']

    def get_task_count(self, obj):
        return obj.tasks.count()

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class TaskSerializer(serializers.ModelSerializer):
    owner = UserSerializer(read_only=True)
    category = CategorySerializer(read_only=True)
    category_id = serializers.IntegerField(write_only=True, required=False, allow_null=True)
    assigned_to = UserSerializer(many=True, read_only=True)
    assigned_to_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    is_overdue = serializers.SerializerMethodField()

    class Meta:
        model = Task
        fields = ['id', 'title', 'desc', 'completed', 'priority', 'due_date',
                  'owner', 'category', 'category_id', 'assigned_to', 'assigned_to_ids',
                  'is_overdue', 'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']

    def get_is_overdue(self, obj):
        if obj.due_date and not obj.completed:
            from django.utils import timezone
            return obj.due_date < timezone.now()
        return False

    def create(self, validated_data):
        assigned_to_ids = validated_data.pop('assigned_to_ids', [])
        category_id = validated_data.pop('category_id', None)
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            raise serializers.ValidationError("Authentication required to create tasks")
        # Create task first (without ManyToMany relationships)
        task = Task.objects.create(owner=request.user, **validated_data)
        
        # Set ForeignKey after creation if needed
        if category_id is not None:
            task.category_id = category_id
            task.save()
        
        # Set ManyToMany relationships after instance is saved
        if assigned_to_ids:
            task.assigned_to.set(assigned_to_ids)
        
        return task

    def update(self, instance, validated_data):
        assigned_to_ids = validated_data.pop('assigned_to_ids', None)
        category_id = validated_data.pop('category_id', None)
        
        # Update simple fields
        instance.title = validated_data.get('title', instance.title)
        instance.desc = validated_data.get('desc', instance.desc)
        instance.completed = validated_data.get('completed', instance.completed)
        instance.priority = validated_data.get('priority', instance.priority)
        instance.due_date = validated_data.get('due_date', instance.due_date)
        
        # Update ForeignKey if provided
        if category_id is not None:
            instance.category_id = category_id
        
        # Save instance before updating ManyToMany
        instance.save()
        
        # Update ManyToMany relationships after save
        if assigned_to_ids is not None:
            instance.assigned_to.set(assigned_to_ids)
        
        return instance
```

### Step 3: Update ViewSet

```python
# api/views.py
class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['completed', 'priority', 'category']
    search_fields = ['title', 'desc']
    ordering_fields = ['title', 'priority', 'due_date', 'created_at']
    ordering = ['-created_at']

    def get_queryset(self):
        queryset = Task.objects.select_related('owner', 'category').prefetch_related(
            'assigned_to'
        ).filter(owner=self.request.user)
        
        # Filter by overdue
        if self.request.query_params.get('overdue') == 'true':
            from django.utils import timezone
            queryset = queryset.filter(due_date__lt=timezone.now(), completed=False)
        
        return queryset

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

## Soft Delete

### Implement Soft Delete

```python
# api/models.py
from django.db import models
from django.utils import timezone

class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)

class Task(models.Model):
    # ... fields ...
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    objects = SoftDeleteManager()
    all_objects = models.Manager()  # Access deleted items
    
    def delete(self, using=None, keep_parents=False):
        self.deleted_at = timezone.now()
        self.save()
    
    def restore(self):
        self.deleted_at = None
        self.save()
    
    class Meta:
        ordering = ['-created_at']
```

## Advanced Query Patterns

### Complex Filtering

```python
from django.db.models import Q, Count, Avg

# OR conditions
tasks = Task.objects.filter(
    Q(priority='high') | Q(due_date__lt=timezone.now())
)

# AND conditions
tasks = Task.objects.filter(
    Q(completed=False) & Q(priority='high')
)

# Exclude
tasks = Task.objects.exclude(completed=True)

# Annotate with related count
posts = Post.objects.annotate(
    comment_count=Count('comments'),
    tag_count=Count('tags')
).filter(comment_count__gte=5)
```

## Exercises

### Exercise 1: Complete Blog API

1. Create Post, Comment, Tag models with relationships
2. Create nested serializers
3. Implement query optimization
4. Add filtering and search
5. Test all endpoints

### Exercise 2: Enhanced Task API

1. Add Category and User relationships
2. Implement ManyToMany for assigned users
3. Add computed fields (is_overdue)
4. Optimize queries
5. Test with cURL

### Exercise 3: Signals and Background Tasks

1. Add signals for post creation
2. Implement email notification task
3. Test signal triggers

## Add-ons

### Add-on 1: Custom Manager

```python
# api/models.py
class PublishedPostManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Post(models.Model):
    # ... fields ...
    objects = models.Manager()
    published = PublishedPostManager()
```

### Add-on 2: Advanced Validation

Implement complex validation rules with custom validators.

### Add-on 3: Serializer Context

Use context to pass request data to serializers.

## Next Steps

Congratulations! You've completed Level 3. You now know:

- ✅ Model relationships (ForeignKey, OneToOne, ManyToMany)
- ✅ Nested serializers (read-only and writable)
- ✅ SerializerMethodField and custom fields
- ✅ Query optimization (select_related, prefetch_related)
- ✅ Django signals
- ✅ Background tasks

**Ready for Level 4?** Continue to [Level 4: Scalable API Design](LEVEL_4_SCALABLE.md) to learn about caching, versioning, file uploads, and async views!

---

**Resources:**
- [DRF Serializers](https://www.django-rest-framework.org/api-guide/serializers/)
- [Django Model Relationships](https://docs.djangoproject.com/en/stable/topics/db/models/#relationships)
- [Django Query Optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [Celery Documentation](https://docs.celeryproject.org/)

