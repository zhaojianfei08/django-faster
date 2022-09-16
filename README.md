# django-faster
django-faster 10行代码实现一张表的增删改查


# 1 知识储备

## 1.1 创建项目



- 创建django项目

  ```python
  创建一个空文件夹，用户存放django项目
  
  django-admin startproject faster  # faster 是项目名称
  ```

- 创建两个django app

  ```shell
  cd faster
  
  python manage.py startapp fast 
  
  python manage.py startapp testapp 
  ```

- 创建成功后的项目目录结构

  ```python
  --demo 
    --faster
      --fast
        --migrations
        --__init__.py
        --admin.py
        --apps.py
        --models.py
        --tests.py
        --views.py
      --testapp
        --migrations
        --__init__.py
        --admin.py
        --apps.py
        --models.py
        --tests.py
        --views.py
      --faster
        --__init__.py
        --asgi.py
        --settings.py
        --urls.py
        --wsgi.py
      --manage.py
  ```

  



## 1.2 django程序启动前预留的钩子 



- faster/testapp/apps/

  ```python
  from django.apps import AppConfig
  
  
  class TestappConfig(AppConfig):
      default_auto_field = "django.db.models.BigAutoField"
      name = "testapp"
  
  ```

  ```python
  class AppConfig:
  	def ready(self):
          """
          Override this method in subclasses to run code when Django starts.
          在子类重写这个方法，当django启动时，执行这个方法中的代码
          """
          
  ```



- 测试

  ```python
  class TestappConfig(AppConfig):
      default_auto_field = "django.db.models.BigAutoField"
      name = "testapp"
      def ready(self):
          print('django启动前我被执行了')
  ```

  ```shell
  F:\auth_demo\faster\testapp\apps.py changed, reloading.
  django启动前我被执行了
  Watching for file changes with StatReloader
  Performing system checks...
  
  System check identified no issues (0 silenced).
  
  You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
  Run 'python manage.py migrate' to apply them.
  September 11, 2022 - 10:46:56
  Django version 4.1.1, using settings 'faster.settings'
  Starting development server at http://127.0.0.1:8000/
  Quit the server with CTRL-BREAK.
  ```



## 1.3 django路由分发的本质



首先说一下视图的类型，从视图的类型划分，有CBV和FBV即类视图和函数视图，一般类视图适合用于生成restful风格的API。函数视图使用比较简单。



### 1.3.1 路由的两种基本使用方式：

- 方式一：

  这是最简单的路由设置，直接在项目文件夹下面的urls.py中配置路由规则和视图函数，浏览器访问http://127.0.0.1:8000/test/ 就会返回驶入函数的返回值。

  faster/faster/urls.py

  ```python
  from django.contrib import admin
  from django.urls import path
  from testapp import views
  
  
  urlpatterns = [
      path("admin/", admin.site.urls),
      path('test/', views.test)
  ]
  ```

  faster/testapp/views.py

  ```python
  from django.shortcuts import HttpResponse
  
  
  def test(request):
      return HttpResponse('test!')
  ```

- 方式二

  使用include分发路由

  faster/faster/urls.py

  ```python
  from django.contrib import admin
  from django.urls import path, include
  
  urlpatterns = [
      path("admin/", admin.site.urls),
      # 路由以test开头，那么分发到testapp的urls模块中，namespace用于反向生成url
      path('test/', include('testapp.urls', namespace='test'))
  ]
  ```

  faster/testapp/urls.py

  ```python
  from django.urls import path
  
  from . import views
  
  app_name = 'test_app'
  
  urlpatterns = [
      path("home/", views.test, name='home'),
  ]
  ```

  faster/testapp/views.py

  ```python
  from django.shortcuts import HttpResponse, reverse
  
  
  def test(request):
      # 如果没有路由分发，反向生成url只需要使用name属性即可
      # 如果有路由分发，反向生成url需要 'namespace:name' 这种格式
      url = reverse('test:home')
      print(url) # 输出 /test/home/
      return HttpResponse('test!')
  ```



### 1.3.2 路由分发的本质

- include函数的源码：

  ```python
  def include(arg, namespace=None):
      app_name = None
      if isinstance(arg, tuple):
          # Callable returning a namespace hint.
          try:
              urlconf_module, app_name = arg
          except ValueError:
              if namespace:
                  raise ImproperlyConfigured(
                      "Cannot override the namespace for a dynamic module that "
                      "provides a namespace."
                  )
              raise ImproperlyConfigured(
                  "Passing a %d-tuple to include() is not supported. Pass a "
                  "2-tuple containing the list of patterns and app_name, and "
                  "provide the namespace argument to include() instead." % len(arg)
              )
      else:
          # No namespace hint - use manually provided namespace.
          urlconf_module = arg
      if isinstance(urlconf_module, str):
          urlconf_module = import_module(urlconf_module)
      patterns = getattr(urlconf_module, "urlpatterns", urlconf_module)
      app_name = getattr(urlconf_module, "app_name", app_name)
      if namespace and not app_name:
          raise ImproperlyConfigured(
              "Specifying a namespace in include() without providing an app_name "
              "is not supported. Set the app_name attribute in the included "
              "module, or pass a 2-tuple containing the list of patterns and "
              "app_name instead.",
          )
      namespace = namespace or app_name
      # Make sure the patterns can be iterated through (without this, some
      # testcases will break).
      if isinstance(patterns, (list, tuple)):
          for url_pattern in patterns:
              pattern = getattr(url_pattern, "pattern", None)
              if isinstance(pattern, LocalePrefixPattern):
                  raise ImproperlyConfigured(
                      "Using i18n_patterns in an included URLconf is not allowed."
                  )
      return (urlconf_module, app_name, namespace)
  ```

  整个逻辑很简单，做了一些判断，关键是return，返回了一个含有三个元素的元组，分别是url配置信息的文件路径、应用名称、命名空间

- 路由分发的方式一

  faster/faster/urls.py

  ```python
  from django.contrib import admin
  from django.urls import path, include
  from testapp import urls
  
  urlpatterns = [
      path("admin/", admin.site.urls),
      #path('test/', include('testapp.urls', namespace='test'))
      path('test/', (urls, None, None)) # urls是python的模块，python中一切皆对象，所以模块也是对象
  ]
  ```

  faster/testapp/urls.py

  ```python
  from django.urls import path
  
  from . import views
  
  app_name = 'test_app'
  
  urlpatterns = [
      path("home/", views.test, name='home'),
  ]
  
  ```

  faster/testapp/views.py

  ```python
  from django.shortcuts import HttpResponse, reverse
  
  
  def test(request):
      return HttpResponse('test!')
  
  ```

  

- 路由分发的方式二

  faster/faster/urls.py

  ```python
  from django.contrib import admin
  from django.urls import path, include
  from testapp import views
  
  urlpatterns = [
      path("admin/", admin.site.urls),
      path('level1/', ([path('item1/', ([path('content/', views.content1), ], None, None)),
                        path('item2/', ([path('content/', views.content2), ], None, None))
                        ], None, None))
  ]
  
  ```

  faster/testapp/views.py

  ```python
  from django.shortcuts import HttpResponse, reverse
  
  
  def content1(request):
      return HttpResponse('content1')
  
  
  def content2(request):
      return HttpResponse('content2')
  ```

  路由分发结果如下:

  http://127.0.0.1:5000/level1/

  -----------------------------------------item1/

  --------------------------------------------------content/

  -----------------------------------------item2/

  --------------------------------------------------content/



## 1.5 自动在app包中查找某个模块并执行



### 1.5.1 python中的包和模块

==包==---是一个包含\__init__.py文件的文件夹

模块---就是一个后缀名为py的文件



```python
--testapp # Python中的包
      --migrations
      --__init__.py 
      --admin.py   # 从这往下都是Python中的模块
      --apps.py
      --models.py
      --tests.py
      --views.py
```



Python中一切皆对象，包和模块也是对象。



### 1.5.2 django-admin源码浅析



大家在使用django-admin的时候

第一步是在每个app的admin.py下面将模型类注册到site中。

faster/test_app/apps.py

```python
from django.apps import AppConfig
from django.utils.module_loading import autodiscover_modules


class TestappConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "testapp"

    def ready(self):
        autodiscover_modules('test')

        

```

faster/test_app/test.py

```python
def func():
    print('我被调用了')

func()
```

执行结果： 

```shell
F:\.venvs\py3.10\.venv\Lib\site-packages\django\urls\conf.py changed, reloading.
我被调用了
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
September 11, 2022 - 19:22:05
Django version 4.1.1, using settings 'faster.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CTRL-BREAK.

```



## 1.6 单例模式



- 方式一：通过包的方式

  >通过利用Python模块导入的特性：在Python中，如果已经导入过的文件再次被重新导入时候，python不会再重新解释一遍，而是选择从内存中直接将原来导入的值拿来用。

  a.py

  ```python
  class Foo:
      def __init__(self, a=1):
          self.a = a
  
   
  foo = Foo() # foo就是一个单例
  
  # 单  一个
  # 例  对象
  
  ```

  b.py

  ```python
  import a
  
  print(a.foo)  # <a.Foo object at 0x00000205C1787C40>
  
  print(a.foo)  # <a.Foo object at 0x00000205C1787C40>
  ```

  

- 方式二：通过重写\__new__方法的方式

  ```python
  # 实例是通过类创建的，类创建实例要先调用类的__new__方法，然后再执行__init__方法
  
  class SingleTon:
  
      def __new__(cls, *args, **kwargs):
          if not hasattr(cls, '_instance'):
              cls._instance = super().__new__(cls, *args, **kwargs)
          return cls._instance
  
  
  s1 = SingleTon()
  s2 = SingleTon()
  
  print(s1)
  print(s2)
  ```

## 1.7 简述元类



几乎所有面向对象的语音中，都有类和对象的概念。

对象是由类创建出来的，类规范了对象的状态和方法。

但是你有考虑过类是由什么创建的吗？类都是由元类创建的，type是所有类的默认元类。

继承和实例化是两个概念。

同时需要注意以下以下几个特殊方法和内部函数：

- \__new__()  创建一个实例对象
- \__init__() 实例化一个实例对象
- \__call__() 实例() 会执行实例的类的 \__call__方法 
- isinstance(obj, cls) obj是不是cls的实例对象
- issubclass(cls1, cls2) cls1是不是cls2的子类



## 1.8 django中获取app名称和模型类

  通过模型类的_meta属性获取到模型类所属的app名称和模型名称(小写字母)



```python
models.UserInfo._meta.app_label  # app名称
models.UserInfo._meta.model_name  # 数据表名称，模型类转换成小写形式
```





## 1.9 方法和函数的区别



- 函数

  ```python
  def xxx():
  	pass 
  
  print(type(xxx))  #<class 'function'>
  ```

- 方法

  ```python
  class Foo:
      def test(self):
          pass
  
  
  f = Foo()
  
  print(type(f.test)) # <class 'method'>
  
  ```

- 特殊的函数定义：

  ```python
  class Foo:
      def test(self):
          pass
  
  print(type(Foo.test)) #<class 'function'>
  ```

- 如何判断一个对象属于哪个类，类==实例化==得到对象，实例化(创造)关系与继承关系一定要分清楚，否则学不明白面向对象，概念不清晰更别提应用了，面向对象是那么的灵活。所以基础概念一定要牢固

  ```python
  from types import FunctionType, MethodType
  
  
  def xxx():
      pass
  
  
  class Foo:
      def test(self):
          pass
  
  
  f = Foo()
  
  if __name__ == '__main__':
      print(isinstance(Foo.test, MethodType))  # False
      print(isinstance(xxx, FunctionType))  # True
      print(isinstance(f.test, FunctionType))  # False
  ```



## 1.10 生成器与yield 



- 生成器表达式

  ```python
  a = (i for i in range(100))
  
  b = [i for i in range(100)]
  
  print(a)  # <generator object <genexpr> at 0x000001D2763004A0>
  
  print(type(a))  # <class 'generator'>
  
  print(type(b))  # <class 'list'>
  
  import sys
  
  print(sys.getsizeof(a))  # 104
  
  print(sys.getsizeof(b))  # 920
  ```

  从运行结果可以开出，生成器可以大量节省系统内存，并且生成器是特殊的迭代器，可以使用for循环遍历出里面的元素

  举个例子：

  小红和小绿都很喜欢吃面包，小红从工厂直接买回来100块，放在家里堆起来，小绿则是饿了的时候去商店买一块，小红就类似与列表，把所有的元素都放到内存中，小绿就类似于生成器，需要的时候才来取。

- yield实现生成器

  ```python
  def xxx():
      yield 1
  
  
  print(xxx())  # <generator object xxx at 0x000001F0EC9304A0>
  
  print(next(xxx()))  # 1
  ```

- 生成器还有一些其他用法，如，send、next、throw、yield from、协程等，本组件开发暂时用不到，就不多说了。



## 1.11 反射



反射是静态面向对象语言具有的特性，比如Java就严重依赖反射。

Python同样具有反射的特性，只要使用内建方法中的getattr(obj, attr) 



```python
class Foo:
    # 类变量
    x = 123

    @classmethod
    def f1(cls):
        print('类方法')

    @staticmethod
    def f2():
        print('静态方法')

    def f3(self):
        print('实例方法')

    def __init__(self, name='foo'):
        # 实例变量
        self.name = name


foo = Foo()

getattr(foo, 'f1')()
getattr(foo, 'f2')()
getattr(foo, 'f3')()

print(getattr(Foo, 'x'))
print(getattr(foo, 'name'))
print(getattr(foo, 'x'))
```



除了getattr之后还有

- setattr

  ```python
  setattr(obj, attr, value)  等价于  obj.attr = value
  ```

  

- hasattr

  ```python
  hasattr(obj, attr)   返回True  or   False
  ```

  

- delattr

  ```python
  delattr(obj, attr)  删除obj的attr属性
  ```

  

## 1.12 装饰器



- 语法糖写法

  ```python
  import time
  
  
  def wrapper(func):
      print('装饰器执行的时机有点特殊')
      print(func.__name__)
  
      def inner(*args, **kwargs):
          print('start...')
          start_time = time.time()
          result = func(*args, **kwargs)
          print(f'end...total use{time.time() - start_time} s')
          return result
  
      inner.__name__ = func.__name__
      return inner
  
  
  @wrapper
  def func1():
      time.sleep(2)
      print('就是玩')
  
  
  @wrapper
  def func2():
      time.sleep(1)
      print('还得玩')
  
  
  func1()
  
  print(func1.__name__)
  
  func2()
  
  print(func2.__name__)
  
  ```

  - 关键点1:

    文件被执行的时候就会被调用。

  - 关键点2：

    装饰器会改变原函数的元信息

- 非语法糖写法

  ```python
  wrapper(func1)()
  # 与语法糖相同
  ```



## 1.13 组合查询Q对象





## 1.14 request.GET  对象





# 2 faster组件使用说明



首先需要注册app：

项目文件夹/settings.py 

```python
INSTALLED_APPS = [
    ###
    'fast',
]
```



```python
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
            'libraries': {  # Adding this section should work around the issue.
                'staticfiles': 'django.templatetags.static',
            },
        },
    },
]
```



在你的app中创建一个fast.py模块：

--testapp

  ----fast.py 

所有的配置都将写在fast.py中



--testapp

​	----models.py

```python
class UserInfo(models.Model):
    """
    用户模型
    """
    user_name = models.CharField(verbose_name='用户名', max_length=32)
    password = models.CharField(verbose_name='密码', max_length=32)
    mobile = models.CharField(verbose_name='手机号', max_length=11)
    token = models.OneToOneField(verbose_name='token', to='UserToken', on_delete=models.SET_NULL, null=True, blank=True)
    depart = models.ForeignKey(verbose_name='部门', to='Depart', on_delete=models.SET_NULL, null=True, blank=True)

    def __str__(self):
        return self.user_name
```



## 2.1 入门示例



- fast.py 

  ```python
  from fast.service.main import site, FastHandler, get_choice_text
  from . import models
  from django import forms
  
  
  class HandlerUserInfo(FastHandler):
      list_display = [FastHandler.display_checkbox, 'id', 'user_name', 'mobile', FastHandler.display_edit, FastHandler.display_delete]
  
      add_model_form_class = UserInfoModelForm
  
      has_add_btn = True
  
      per_page_count = 2
  
      order_list = ['-user_name']
  
      action_list = [FastHandler.action_multi_delete, ]
  
      search_list = ['user_name__contains']
      
  site.register(models.UserInfo, HandlerUserInfo)
  ```



.

## 2.2 列表功能-list_display



在继承FastHandler的类变量中添加list_diaplay可以定制需要显示的列：

除去常规字段，比如’id‘、'user_name'、’mobile‘ 

还可以配置特殊字段：

- FastHandler.display_checkbox :

  ![image-20220916093417342](F:\auth_demo\faster\readme.assets\image-20220916093417342.png) 

- FastHandler.display_edit

  ![image-20220916093450209](F:\auth_demo\faster\readme.assets\image-20220916093450209.png)

- FastHandler.display_delete

  ![image-20220916093514673](F:\auth_demo\faster\readme.assets\image-20220916093514673.png)



## 2.3 新增按钮-has_add_btn & add_model_form_class



- has_add_btn 默认为True , 将其修改为False，界面不具备新增功能。

- add_model_form_class，新增字段的form表单，可以自定义modelform

  ```python
  class UserInfoModelForm(forms.ModelForm):
      class Meta:
          model = models.UserInfo
          fields = '__all__'
  
      def clean_user_name(self):
          user_name = self.cleaned_data['user_name']
          if user_name != 'kobe':
              raise forms.ValidationError('user_name 验证错误')
          return user_name
  ```

  自定义modelform，设置add_model_form_class = 类名即可，本实例中可以写成：add_model_form_class = UserInfoModelForm





## 2.4 按字段排序-order_list 



- order_list

  - ['id', ]  按id升序排
  - ['-id',] 按id降序排
  - ['id', '-user_name'] 按id升序排，user_name 降序排
  - ['c1','c2','c3',......] 按多个字段排

  

  

  

## 2.5 关键词搜索



- search_list 
  - ['user_name__contains']  等价于sql中的like
  - ['user_name']  需要精准匹配上
  - 其他django ORM filter语法



## 2.6 分页

- per_page_count 必须为整型

  per_page_count = 1 每页显示1条

  per_page_count = 3 每页显示3条

  以此类推。。。。



## 2.7 批量操作

- action_list = [FastHandler.action_multi_delete, ]

  目前只支持批量删除

  ![image-20220916094846740](F:\auth_demo\faster\readme.assets\image-20220916094846740.png)









