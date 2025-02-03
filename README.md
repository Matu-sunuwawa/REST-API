# 🚀REST-API
+ We're going to create a simple API to allow admin users to view and edit the users and groups in the system.

## Project setup
```
python3 -m venv env
source env/bin/activate

# Install Django and Django REST framework into the virtual environment
pip install djangorestframework

django-admin startproject tutorial

cd tutorial
django-admin startapp quickstart
```

Now sync your database for the first time:
```
python manage.py migrate
```
We'll also create an initial user named admin with a password:
```
python manage.py createsuperuser --username admin --email admin@example.com
```

### Serializers
`tutorial/quickstart/serializers.py`:
```
from django.contrib.auth.models import Group, User
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ['url', 'name']
```

`tutorial/quickstart/views.py` and get typing:
```
from django.contrib.auth.models import Group, User
from rest_framework import permissions, viewsets

from tutorial.quickstart.serializers import GroupSerializer, UserSerializer


class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows groups to be viewed or edited.
    """
    queryset = Group.objects.all().order_by('name')
    serializer_class = GroupSerializer
    permission_classes = [permissions.IsAuthenticated]
```
Note That: Rather than write multiple views we're grouping together all the common behavior into classes called ViewSets.

`tutorial/urls.py`:
```
from django.contrib import admin
from django.urls import include, path
from rest_framework import routers

from quickstart import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

### Pagination
`tutorial/settings.py`:
```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```
Note That: Add `'rest_framework'` to INSTALLED_APPS.

### Testing our API
```
python manage.py runserver
```
Test on command-line, using tools like `curl`:
```
curl -u youradmin -H 'Accept: application/json; indent=4' http://127.0.0.1:8000/users/
```
output: $HTTP/1.1 200 OK

or 

on browser, by going to the URL `http://127.0.0.1:8000/users/`
Note: but you should have to login in your admin

🎉Good Job

## Generic setup for next tutorials:
```
python3 -m venv env
source env/bin/activate

pip install django
pip install djangorestframework
pip install pygments  # We'll be using this for the code highlighting

django-admin startproject tutorial
cd tutorial

python manage.py startapp snippets
```

We'll need to add our new snippets app and the rest_framework app to INSTALLED_APPS.

edit the snippets/models.py file:
```
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ['created']
```

```
python manage.py makemigrations snippets
python manage.py migrate snippets
```

Create a file in the snippets directory named `serializers.py` and add the following:
```
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

Before we go any further we'll familiarize ourselves with using our new Serializer class. Let's drop into the Django shell.
```
python manage.py shell
```

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```
We've now got a few snippet instances to play with. Let's take a look at serializing one of those instances.
```
serializer = SnippetSerializer(snippet)
serializer.data
```
At this point we've translated the model instance into Python native datatypes. To finalize the serialization process we render the data into json.
```
content = JSONRenderer().render(serializer.data)
content
```
### <mark>Deserialization</mark> is similar. First we parse a stream into Python native datatypes...
```
import io

stream = io.BytesIO(content)
data = JSONParser().parse(stream)
```
...then we restore those native datatypes into a fully populated object instance.
```
serializer = SnippetSerializer(data=data)
serializer.is_valid()

serializer.validated_data
serializer.save()
```
output:
```
# True
# {'title': '', 'code': 'print("hello, world")', 'linenos': False, 'language': 'python', 'style': 'friendly'}
# <Snippet: Snippet object>
```

*We can also serialize `querysets` instead of model instances. To do so we simply add a `many=True` flag to the serializer arguments.
```
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
```

Open the file snippets/serializers.py again, and replace the SnippetSerializer class with the following.
```
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style']
```

Open the Django shell with python manage.py shell
```
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
```

## Serialization (Tutorial 1)

Edit the snippets/views.py file, and add the following:
```

from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer

@csrf_exempt
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)

@csrf_exempt
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```
+ view that supports listing all the existing snippets, or creating a new snippet.
+ We'll also need a view which corresponds to an individual snippet, and can be used to retrieve, update or delete the snippet.
+ Finally we need to wire these views up. Create the snippets/urls.py file:
```
from django.urls import path
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>/', views.snippet_detail),
]
```
We also need to wire up the root urlconf, in the tutorial/urls.py file:
```
from django.urls import path, include

urlpatterns = [
    path('', include('snippets.urls')),
]
```

### Testing our first attempt at a Web API
+ Quit out of the shell...
+ start up Django development server
```
python manage.py runserver
```
On other terminal (do not stop the server ... it will not work.)
+ We can test our API using curl or httpie. Httpie is a user friendly http client that's written in Python. Let's install that. You can install httpie using pip:
```
pip install httpie
```
```
http GET http://127.0.0.1:8000/snippets/ --unsorted
```
and
```
http GET http://127.0.0.1:8000/snippets/2/ --unsorted
```
output: `HTTP/1.1 200 OK`


🎉 Let's Celebrate


## Requests and Responses (Tutorial 2)
### Request objects
+ The core functionality of the Request object is the request.data attribute, which is similar to request.POST, but more useful for working with Web APIs.
```
request.POST  # Only handles form data.  Only works for 'POST' method.
request.data  # Handles arbitrary data.  Works for 'POST', 'PUT' and 'PATCH' methods.
```
### Response objects
```
return Response(data)  # Renders to content type as requested by the client.
```
### Status codes
+ REST framework provides more explicit identifiers for each status code, such as HTTP_400_BAD_REQUEST in the status module.

### Wrapping API views
REST framework provides two wrappers you can use to write API views.
+ The @api_view decorator for working with function based views.
+ The APIView class for working with class-based views.
These wrappers provide a few bits of functionality such as making sure you receive Request instances in your view, and adding context to Response objects so that content negotiation can be performed.
  
`views.py:`
```
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
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

@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
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

### Adding optional format suffixes to our URLs
+ Using format suffixes gives us URLs that explicitly refer to a given format, and means our API will be able to handle URLs such as http://example.com/api/items/4.json.
```
def snippet_list(request, format=None)
```
and
```
def snippet_detail(request, pk, format=None)
```

Now update the snippets/urls.py file slightly
```
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>/', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```
We don't necessarily need to add these extra url patterns in, but it gives us a simple, clean way of referring to a specific format.
RUN:
```
http http://127.0.0.1:8000/snippets/
```
We can control the format of the response that we get back, either by using the Accept header:
```
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
```
or by appending a format suffix
```
http http://127.0.0.1:8000/snippets.json  # JSON suffix
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
```

we can control the format of the request that we send, using the Content-Type header.
```
# POST using form data
http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"

# POST using JSON
http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"
```

🎉Good Job

## Class-based Views (Tutorial 3)
### Rewriting our API using <mark>class-based views</mark>
modify `views.py`:
```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
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
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
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
refactor our `snippets/urls.py`:
```
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.SnippetList.as_view()),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

### <mark>Using mixins</mark>
+ One of the big wins of using class-based views is that it allows us to easily compose reusable bits of behavior.
replace `snippets/views.py`:
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
+ `GenericAPIView`, `ListModelMixin` and `CreateModelMixin`: provides the core functionality, and the mixin classes provide the .list() and .create() actions.
+ `GenericAPIView` class to provide the core functionality, and adding in mixins to provide the .retrieve(), .update() and .destroy() actions.

### <mark>Using generic</mark> class-based views

replace `views.py`:
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
+ that's pretty concise and do the same operation as previous codes.

🎉Let's Celebrate


## Authentication & Permissions (Tutorials 4)
Add the following two fields to the Snippet model in `models.py`:

```

...

from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight

...

class Snippet(models.Model):

    ...

    from pygments.lexers import get_lexer_by_name
    from pygments.formatters.html import HtmlFormatter
    from pygments import highlight

    # Authentication and permission
    owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
    highlighted = models.TextField()

    def save(self, *args, **kwargs):
        """
        Use the `pygments` library to create a highlighted HTML
        representation of the code snippet.
        """
        lexer = get_lexer_by_name(self.language)
        linenos = 'table' if self.linenos else False
        options = {'title': self.title} if self.title else {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos,
                                  full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super().save(*args, **kwargs)

    ...

```
When that's all done we'll need to update our database tables. let's just delete the database and start again.
```
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```
```
python manage.py createsuperuser
```
### Adding endpoints for our User models
In `serializers.py` add:
```
...

from django.contrib.auth.models import User

...

class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ['id', 'username', 'snippets']

```
We'd like to just use read-only views for the user representations, so we'll use the ListAPIView and RetrieveAPIView generic class-based views.
In `views.py`:
```
...

from snippets.serializers import UserSerializer
from django.contrib.auth.models import User

...

...

class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer

class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

Finally we need to add those views into the API.
in `snippets/urls.py`:
```
...

urlpatterns = [
    ...

    path('users/', views.UserList.as_view()),
    path('users/<int:pk>/', views.UserDetail.as_view()),
]

...

```
### Associating Snippets with Users
Right now, if we created a code snippet, <mark>there'd be no way of associating the user that created the snippet</mark>, with the snippet instance. The user isn't sent as part of the serialized representation, but is instead a property of the incoming request.
The way we deal with that is by overriding a .perform_create() method on our snippet views, that allows us to modify how the instance save is managed, and handle any information that is implicit in the incoming request or requested URL.
inside `views.py` On the `SnippetList view clas`s, add the following method:
```

...

class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

...

```

Add the following field to the serializer definition in serializers.py:
```

...

class SnippetSerializer(serializers.ModelSerializer):
    # Add
    owner = serializers.ReadOnlyField(source='owner.username')

    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style', 'owner']

...

```
### Adding required permissions to views
update `views.py`:
```

...

from rest_framework import permissions

class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

...

```
### Adding login to the Browsable API
We can add a login view for use with the browsable API, by editing the URLconf in our project-level `urls.py` file.
```
...
from django.urls import path, include

urlpatterns = [
    ...
    path('api-auth/', include('rest_framework.urls')),
]
```
+ The 'api-auth/' part of pattern can actually be whatever URL you want to use.

### Object level permissions
+ Really we'd like all code snippets to be visible to anyone, but also make sure that only the user that created a code snippet is able to update or delete it.
create a new file, `permissions.py`:
```
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions are only allowed to the owner of the snippet.
        return obj.owner == request.user
```
Now we can add that custom permission to our snippet instance endpoint, by editing the permission_classes property inside `views.py` on the `SnippetDetail view class`:
```

...

from rest_framework import permissions
from snippets.permissions import IsOwnerOrReadOnly

...

class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    # Add
    permission_classes = [permissions.IsAuthenticatedOrReadOnly,IsOwnerOrReadOnly]

...

```
Now, if you open a browser again, you find that the 'DELETE' and 'PUT' actions only appear on a snippet instance endpoint if you're logged in as the same user that created the code snippet.

### Authenticating with the API
+ Because we now have a set of permissions on the API, we need to authenticate our requests to it if we want to edit any snippets.
+ If we try to create a snippet without authenticating, we'll get an error:
```
http POST http://127.0.0.1:8000/snippets/ code="print(123)"
```
We can make a successful request by including the username and password of one of the users we created earlier.
```
http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print(789)"
```

🎉Good Job


## Relationships & Hyperlinked APIs (Tutorials 5)
+ improve the cohesion and discoverability of our API, by instead using `hyperlinking` for relationships.

### Creating an endpoint for the root of our API
+ To create `single entry point to our API`, we'll use a regular function-based view and the @api_view decorator we introduced earlier
+ in `snippets/views.py`:
```

...

from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse

...

@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```
Two things should be noticed here. 
+ First, we're using REST framework's reverse function in order to return fully-qualified URLs;
+ second, URL patterns are identified by convenience names that we will declare later on in our snippets/urls.py.

### Creating an endpoint for the highlighted snippets
+ Instead of using a `concrete generic view`, we'll use the base class for representing instances, and create our own .get() method. In your `snippets/views.py` add:
```

...

from rest_framework import renderers

...

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = [renderers.StaticHTMLRenderer]

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```
in `snippets/urls.py`:
```
path('', views.api_root),
# add a url pattern for the snippet highlights:
path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view()),
```
### Hyperlinking our API
There are a number of different ways that we might choose to represent a relationship between entities:
+ Using `primary keys`.
+ ==>Using `hyperlinking` between entities.
+ Using a unique identifying `slug field` on the related entity.
+ Using the default `string representation` of the related entity.
+ Nesting the `related entity` inside the parent representation.
+ Some other custom representation.
Replace HyperlinkedModelSerializer instead of ModelSerializer
The `HyperlinkedModelSerializer` has the following differences from `ModelSerializer`:
+ It does not include the id field by default.
+ It includes a url field, using HyperlinkedIdentityField.
Relationships use `HyperlinkedRelatedField`, instead of `PrimaryKeyRelatedField`.

We can easily re-write our existing serializers to use hyperlinking. In your snippets/serializers.py add:
```

...

class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ['url', 'id', 'highlight', 'owner','title', 'code', 'linenos', 'language', 'style']


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ['url', 'id', 'username', 'snippets']
```
### Making sure our URL patterns are named
`snippets/urls.py`:
```
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns([
    path('', views.api_root),
    path('snippets/',
        views.SnippetList.as_view(),
        name='snippet-list'),
    path('snippets/<int:pk>/',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    path('snippets/<int:pk>/highlight/',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    path('users/',
        views.UserList.as_view(),
        name='user-list'),
    path('users/<int:pk>/',
        views.UserDetail.as_view(),
        name='user-detail')
])
```
modifying our `tutorial/settings.py`:
```

...

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```
If we open a browser and navigate to the browsable API, you'll find that you can now work your way around the API simply by following links.
You'll also be able to see the 'highlight' links on the snippet instances, that will take you to the highlighted code HTML representations.

🎉Good job

## ViewSets & Routers
+ `ViewSets`, that allows the developer to concentrate on modeling the state and interactions of the API, and leave the URL construction to be handled automatically.

### Refactoring to use ViewSets
+ First of all let's refactor our UserList and UserDetail classes into a single UserViewSet class
In the `snippets/views.py`:
```

...

from rest_framework import viewsets

...

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    This viewset automatically provides `list` and `retrieve` actions.
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
+ Here we've used the ReadOnlyModelViewSet class to automatically provide the default 'read-only' operations.
+ 
Next we're going to replace the SnippetList, SnippetDetail and SnippetHighlight view classes.
```

...

from rest_framework import permissions
from rest_framework import renderers
from rest_framework.decorators import action
from rest_framework.response import Response

class SnippetViewSet(viewsets.ModelViewSet):
    """
    This ViewSet automatically provides `list`, `create`, `retrieve`,
    `update` and `destroy` actions.

    Additionally we also provide an extra `highlight` action.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

```
+ This time we've used the ModelViewSet class in order to get the complete set of default read and write operations.
+ Notice that we've also used the @action decorator to create a custom action, named highlight

### Binding ViewSets to URLs explicitly
In the `snippets/urls.py`:
```
...

from rest_framework import renderers

from snippets.views import api_root, SnippetViewSet, UserViewSet

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

urlpatterns = format_suffix_patterns([
    path('', api_root),
    path('snippets/', snippet_list, name='snippet-list'),
    path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
    path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
    path('users/', user_list, name='user-list'),
    path('users/<int:pk>/', user_detail, name='user-detail')
])
```

### Using Routers
+ Because we're using ViewSet classes rather than View classes, we actually don't need to design the URL conf ourselves.
in `urls.py`:
```
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from snippets import views

router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet, basename='snippet')
router.register(r'users', views.UserViewSet, basename='user')

urlpatterns = [
    path('', include(router.urls)),
]
```

If we open a browser and navigate to the browsable API, you'll find that you can now work your way around the API simply by following links.
You'll also be able to see the 'highlight' links on the snippet instances, that will take you to the highlighted code HTML representations.

🎉Good job












