- 新建程序步骤：
    - 自动建一个项目文件夹
        ```
        django-admin startproject project_name
        ```
    - 建一个新的app
        ```
        python manage.py startapp app_name
        ```
    - 找到*setting.py*, 把新的app'xx',添加到*installed_apps*里
    - *views.py* 建一个新的def, 把return后面的功能（比如request，或者httpresponse）import到文件里
    - 在对应app文件夹下建一个*urls.py*（注意一个project可以有几个app，每个app都可以有对应的urls文件）
        - urls.py 里建新的urlpattern， 注意要import上一步创建好的views, .的意思是当前文件夹
    - 到上一级的文件夹里找总的*urls.py*文件，把上一步的urls文件include到urlpattern里，然后import include

- 如果想要用户登录时用邮箱认证（默认是username）或者阻止已登录用户通过url进入登录或注册页面，可在app目录下创建backends/middleware.py, 代码如下

    ```
    from django.contrib.auth.backends import ModelBackend
    from django.contrib.auth import get_user_model
    from django.http import HttpResponseRedirect
    from django.urls import reverse

    class EmailBackend(ModelBackend):
        def authenticate(self, request, email=None, password=None, **kwargs):
            User = get_user_model()
            try:
                user = User.objects.get(email=email)
                if user.check_password(password):
                    return user
            except User.DoesNotExist:

                return None

    class RedirectAuthenticatedUsersMiddleware:
        def __init__(self, get_response):
            self.get_response = get_response

        def __call__(self, request):
            if request.user.is_authenticated:

                if request.path == reverse('core:login') or request.path == reverse('core:register'):
                    return HttpResponseRedirect(reverse('network:index'))
            
            response = self.get_response(request)
            return response

    ```
    然后在*setting.py*里设置如下:
    
    ```
        MIDDLEWARE = [
        # Other middleware classes...
        # path, file_name could be backends/middle.py but just use backends/middleware here  
        'app_name.file_name.RedirectAuthenticatedUsersMiddleware',
    ]
    ```
    ```
        AUTHENTICATION_BACKENDS = [
        'app_name.file_name.EmailBackend', 
        'django.contrib.auth.backends.ModelBackend',
    ]
    ```

- 用meidad的步骤:
    - 在**Django**中启用media文件夹储存图片，首先在项目文件夹下面建media文件夹
    - 在*setting.py*里

        - *TEMPLATES*下*context_processors*加上
            ```
            django.template.context_processors.media
            ```
        - 加上
            ```
            MEDIA_URL = '/media/'
            MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
            ```
    - *urls.py*里加上
        ```
        from django.conf import settings
        from django.conf.urls.static import static

        urlpatterns = [... 
        ] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
        ```
    - 最后template里url如下
        ```
        {{ MEDIA_URL }}items/{{ item_number }}/{{ image }}
        ```
- **MySQL**
    - 在*setting.py*里设置

        ```
            DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'core',
            'USER': 'admin',
            'PASSWORD': '1234',
            'HOST': 'localhost',
            'PORT': '3306',
        },

        'network': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'project_network', # yourdatabase name
            'USER': 'admin',
            'PASSWORD': '1234',
            'HOST': 'localhost',
            'PORT': '3306',
            }
        }
        ```
    -  如果把默认user的数据库改成用 **MySQL** *setting.py* 设置
        ```
        AUTH_USER_MODEL = 'core.CustomUser'
        ```
    - *AUTH_USER_MODEL*必须是default database， 否则非常麻烦
    - 如果用两个不同的database储存不同app的数据，需要设置router, *setting.py*里加上
        ```
        DATABASE_ROUTERS = ['myapp.routers.netwotkRouter', 'myapp.routers.coreRouter']
        ```
    - router文件放在*project/myapp*里，示例如下
        ```
        class networkRouter:
            
            route_app_labels = {"network"}

            def db_for_read(self, model, **hints):
                if model._meta.app_label in self.route_app_labels:
                    return "network"
                return None

            def db_for_write(self, model, **hints):

                if model._meta.app_label in self.route_app_labels:
                    return "network"
                return None

            def allow_relation(self, obj1, obj2, **hints):

                if (
                    obj1._meta.app_label in self.route_app_labels
                    or obj2._meta.app_label in self.route_app_labels
                ):
                    return True
                return None

            def allow_migrate(self, db, app_label, model_name=None, **hints):

                if app_label in self.route_app_labels:
                    return db == "network"
                return False
            
        class coreRouter:

            def db_for_read(self, model, **hints):

                return "default"

            def db_for_write(self, model, **hints):

                return "default"

            def allow_relation(self, obj1, obj2, **hints):
                """
                Relations between objects are allowed if both objects are
                in the primary/replica pool.
                """
                db_set = {"network", "default"}
                if obj1._state.db in db_set and obj2._state.db in db_set:
                    return True
                return None

            def allow_migrate(self, db, app_label, model_name=None, **hints):
        
                return True
        ```
    
