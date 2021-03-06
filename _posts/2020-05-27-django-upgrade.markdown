---
layout:     post
title:      "项目升级Django版本"
date:       2020-05-27 11:00:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - Python
---
# 项目升级Django版本

## 一、准备工作
- 在最新dev下切出feature分支
- 项目Dokcerfile
- 项目Pipfile/requirements.txt

## 二、更新依赖
- 进入容器: make bash
- 安装pipenv: `pip install --user pipenv`
- 创建并进入虚拟环境: `pipenv shell`
- 更改Pipfile/requirements.txt中想要升级的包: 例如Django==2.2
- 安装依赖包并处理依赖关系:`pipenv install --skip-lock` 或 `pipenv install -r requirements.txt --skip-lock`
- 查看依赖关系是否满足: `pipenv graph`
- 更新requirements.txt文件:`pipenv lock -r > requirements.txt`
- 退出容器，重新build镜像。如果后续遇到依赖不兼容问题问题，重复以上步骤

## 三、更改代码
- 将Pycharm的Python Interpreter设置为build的镜像
- 进入容器: make bash
- 运行并查看日志: `python manage.py runserver -v 3 0.0.0.0:8080`

(-v 的{0,1,2,3}分别表示error, warning, debug, info。调试各个API查看控制台是否有过期warning报出)
- 根据错误日志，在Pycharm中更改代码

### 主要更改部分
- base.py

1.APPEND_SLASH=False 用于解决以'/'结尾的警告
https://docs.djangoproject.com/en/3.0/releases/2.0.8/

2.MIDDLEWARE_CLASS更名为MIDDLEWARE并且由tuple变为list.

- urls.py

1.包引入变为from django.urls import include, re_path

2.由于path都是正则写的，所以都用re_path

3.include("api.urls", namespace='api/v1') 变为include(("api.urls", "api/v1"), namespace='api/v1')

4.实际上namespace可去掉，它的应用场景主要在于template渲染和reverse()函数反查url，目前没用。

- models.py

1.使用ForeignKey的field都需要加上on_delete属性，老版本默认为model.CASCADE，由于数据库已经设置关闭constraint，所以均改为models.SET_NULL. 而models.SET_NULL需要blank=True, null=True以保证设置成功。

2.Django原来使用Mysql的默认事务隔离级别:可重复读，现在默认为: 读提交级别。
是由于Django models的设计思想就是Read commited的。
Repeatable read之所以data loss的原因是：比如get_or_create()是先get发现无记录，然后create的时候遇到并发而失败了。然后再去get。结果由于读到的数据是事务开始的版本，所以还是返回None.

- migrations.py

1.删除，因为on_delete不兼容会报错，无法runserver

2.在容器中运行`python manage.py makemigrations direct_message`生成新的迁移文件, 然后运行
`python manage.py migrate direct_message --fake`假装对新生成的migrations文件进行apply，这样以后的数据库迁移还能继续关联到数据库。

- serializer.py

1.对应升级了drf框架版本，导致MsgSerializer的CharField报错；因为当前的MsgSerializer中定义了所有消息类型的所有字段，导致AttributeError，而在老版本drf中是被允许的。
例如：发送text消息，feed和audio字段均为None，无法序列化feed_url, audio_url等，所以现在需要显式地设置默认值: CharField(default='')

## 四、收尾
- 在容器中pipenv lock更新Pipfile.lock文件及requirements.txt文件
- 本地运行feature分支，使用charles的mapRemote进行调试
- test环境回测
