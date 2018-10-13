# Django1.11 学习手记

[TOC]

## 验证码插件

###  安装

```python
pip3 install django-simple-captcha
```

### 使用



#### 配置

`settings.py` 中注册这个应用

```python
INSTALLED_APPS = [    
    'captcha',
]
```



#### 生成数据表

```python
makemigrations
migrate
```

#### 路由

`urls.py`

```python
url(r'^captcha/', include('captcha.urls')),
```

#### Form

```python
from django import forms
from captcha.fields import CaptchaField

class CaptchaTestForm(forms.Form):
    myfield = AnyOtherField()
    captcha = CaptchaField(error_messages={
        "invalid": "无效的验证码"
    })
```

…or, as a `ModelForm`:

```python
from django import forms
from captcha.fields import CaptchaField

class CaptchaTestModelForm(forms.ModelForm):
    captcha = CaptchaField()
    class Meta:
        model = MyModel
```

#### 视图

`views.py`

```python
def some_view(request):
    if request.POST:
        form = CaptchaTestForm(request.POST)
        # check the input
        if form.is_valid():
            human = True
    else:
        form = CaptchaTestForm()

    return render_to_response('template.html',locals())
```

####  模板

```html
<form method="post" action="{% url 'register' %}">
   {{ register_form.captcha }}
   {{ register_form.errors.captcha }}
   <button type="submit">提交</button>
   {% csrf_token %}
 </form>
```

#### 渲染后端 `html`

```html
<img src="/captcha/image/43c4a85c5a430fde9acd4ff708d9b3f8659ff305/" alt="captcha" class="captcha">
<input type="hidden" name="captcha_0" value="43c4a85c5a430fde9acd4ff708d9b3f8659ff305" required="" id="id_captcha_0">

<input type="text" name="captcha_1" required="" id="id_captcha_1" autocapitalize="off" autocomplete="off" autocorrect="off" spellcheck="false">
```

### 用户注册功能

#### `model`

假如有下面的 `model`

```python
from django.db import models
from django.contrib.auth.models import AbstractUser

class UserProfile(AbstractUser):
    pass

# 这里是继承了 Django 的 model
```

#### 视图函数

配合验证码中的视图继续编写用户注册逻辑

```python

from django.shortcuts import render
from django.contrib.auth.hashers import make_password
from users.models import UserProfile

def some_view(request):
    if request.POST:
        form = CaptchaTestForm(request.POST)
        # check the input
        if form.is_valid():
            user_name = register_form.cleaned_data.get(
                'email')
            password = register_form.cleaned_data.get(
                'password')
            # 给用户的密码加密
            password = make_password(password)
            UserProfile.objects.create(
                **{"username": user_name,
                   "email": user_name,
                   "password": password})
            return render(request, 'login.html')
    else:
        form = CaptchaTestForm()

    return render(request, 'register.html',locals())
```

### Django 发送邮件

#### 配置

下面一126邮箱为例

`settings.py`

```python
EMAIL_HOST = "smtp.126.com"
EMAIL_PORT = 25
EMAIL_HOST_USER = 'youremail@126.com'
EMAIL_HOST_PASSWORD = 'yourpassword' # 这个是授权密码,不是登录密码
EMAIL_USE_TLS = False
EMAIL_FROM = EMAIL_HOST_USER
```



#### 发送邮件

```python
from django.core.mail import send_mail
from MyProject import settings

send_mail(
	subject="邮件标题",
    message="邮件内容",
    from_email=settings.EMAIL_FROM,
    recipient_list=['1@1.com','2@2.com']  # 收件箱地址列表
    )
```

### 发送激活链接

#### 场景

1. 可以用于审批事情多链接
2. 用于用户注册后的激活链接 

#### 逻辑

下面我用用户注册的功能来表述一下。

一般用户注册成功后，会发给用户的邮箱中一条激活用户的邮件，这个邮件中含有一个激活链接。其实就是服务器生成一条 `url` ，这条 `url` 中有服务器生成的随机字符串，用于识别每条 `url` 的唯一性。

逻辑如下：

用户表里会有一个字段，这个字段用于记录注册成功后用户是否激活。

在用户注册时候，服务器向用户提交的邮箱中发送用于激活的链接，同时会生成一串随机字符串，这个字符串会加入到激活链接中，同时会存放到一个专门用于存放这个字符串的表。

当用户点击这个链接时，服务器从 `url` 中获取到这个字符串，再从数据库中获取这个字符串，假如能获取到，就修改 用户表中的激活字段，激活用户。



#### 具体实现代码

1. 创建一个生成随机字符串的函数

        ```python
def generate_random_code(length=8):
    """
    生成给定长度的随机字符串
    :return:
    """
    import random
    s = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
    return ''.join(random.sample(s, length))
        ```

2. 创建一个发送链接的函数

 ```python
from django.core.mail import send_mail

def send_to_register_email(email, to_emails, 
                           send_type='register'):
    """
    用于给指定的邮箱地址发送链接
    :param email: 发件人邮箱地址
    :param to_emails: 收件箱地址，必须是列表或者元祖
    :param send_type: 类型 注册或者找回密码
    :return: 
    """
    if send_type == 'register':
        email_title = "鲨鱼在线学习平台激活链接"
        email_body = "请点击此链接地址激活账号:"
        url = "http://www.sharkyun.com/active/{}/"
        email_body = email_body + url.format(
            generate_random_code(16))  # 使用了上面的函数
        
    elif send_type == 'r':
        email_title = "鲨鱼在线学习平台找回密码"
        email_body = "请点击此链接地址:"
        url = "http://www.sharkyun.com/reback/{}/"
        email_body = email_body + url.format(
            generate_random_code(16))

    else:
        email_title, email_body = '', ''

    send_status = send_mail(
        subject=email_title,
        message=email_body,
        from_email=email,
        recipient_list=to_emails
    )
    if send_status == 1:
        print('发送成功')
 ```



