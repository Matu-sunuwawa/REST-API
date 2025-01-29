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













