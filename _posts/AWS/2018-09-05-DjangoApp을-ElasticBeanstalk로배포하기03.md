---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기03"
categories:  
- AWS  
tags:  
- Python    
- Docker      
- ElasticBeanstalk    
- deploy    
- nginx     
- uwsgi     
comments : true    
date: "2018-09-05 18:50"  
---      

## 1. settings/production.py 에 debug=True안바꾸고 장고에서 보내주는 log볼수 있도록 log파일 남기기.  

앞선 bad_request  는 장고가 보내준 것이다.  왜인지 알 수 가 없다.!     
장고에서 log설정을 하고 봐야한다.   debug가 True옵션이면 쉽게 볼 수 있다.   
하지만 그렇게하면 production의 안전성 떨어지기에.. 아예 애러 발생시 log파일을 남기는 설정을 해보자.    

**local.py**   
```
from .base import *

DEBUG = True
ALLOWED_HOSTS = []

WSGI_APPLICATION = 'config.wsgi.local.application'

DATABASES = {
'default': {
'ENGINE': 'django.db.backends.sqlite3',
'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
}
}

STATIC_URL = '/static/'



LOG_DIR = os.path.join(ROOT_DIR, '.log')
if not os.path.exists(LOG_DIR):
os.makedirs(LOG_DIR)

LOGGING = {
'version': 1,
'disable_existing_loggers': False,
'filters': {
'require_debug_false': {
'()': 'django.utils.log.RequireDebugFalse',
},
'require_debug_true': {
'()': 'django.utils.log.RequireDebugTrue',
},
},
'formatters': {
'django.server': {
'format': '[%(asctime)s] %(message)s',
}
},
'handlers': {
'console': {
'level': 'INFO',
'filters': ['require_debug_true'],
'class': 'logging.StreamHandler',
},
'file_error': {
'class': 'logging.handlers.RotatingFileHandler',
'level': 'ERROR',
'formatter': 'django.server',
'backupCount': 10,
'filename': os.path.join(LOG_DIR, 'error.log'),
'maxBytes': 10485760,
}
},
'loggers': {
'django': {
'handlers': ['console', 'file_error'],
'level': 'INFO',
'propagate': True,
}
}
}
```  

**production.py**  
```
from .base import *

DEBUG = False
ALLOWED_HOSTS = []

WSGI_APPLICATION = 'config.wsgi.production.application'

DATABASES = {
'default': {
'ENGINE': 'django.db.backends.sqlite3',
'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
}
}

STATIC_URL = '/static/'

LOG_DIR = '/var/log/django'
LOGGING = {
'version': 1,
'disable_existing_loggers': False,
'filters': {
'require_debug_false': {
'()': 'django.utils.log.RequireDebugFalse',
},
'require_debug_true': {
'()': 'django.utils.log.RequireDebugTrue',
},
},
'formatters': {
'django.server': {
'format': '[%(asctime)s] %(message)s',
}
},
'handlers': {
'console': {
'level': 'INFO',
'filters': ['require_debug_true'],
'class': 'logging.StreamHandler',
},
'file_error': {
'class': 'logging.handlers.RotatingFileHandler',
'level': 'ERROR',
'formatter': 'django.server',
'backupCount': 10,
'filename': os.path.join(LOG_DIR, 'error.log'),
'maxBytes': 10485760,
}
},
'loggers': {
'django': {
'handlers': ['console', 'file_error'],
'level': 'INFO',
'propagate': True,
}
}
}
```
 
**Dockerfile**  에 다음 추가   
```
# 로그파일 기록 위한 폴더 생성
RUN mkdir /var/log/django
```     

**.secrets/production.json** 에 다음 추가   
```
{
"ALLOWED_HOSTS" : [
".elasticbeanstalk.com"
]
}
```  

다시 배포하고 접속해본다.    
`git add -A`
`git add -f .secrets`
`eb deploy --profile fc-8th-eb --staged`
`eb open`    
  
다음과 같은 창이 뜬다.   
```
Not Found
The requested URL / was not found on this server
```  
index.html  지정한적 없으니까 이렇게 뜬다. 
notfound는 에러로그가 아니라서 안찍힌다.   
log찍히도록 설정은 일단 된것이니 추후에 애러 상황 발생시 이어서 진행해 보겠다.       


---  


## 2.rds 세팅, s3 버킷 생성(dev,용 pro 용 분할)   

s3먼저 만들어 보겠다.   
먼저 boto를 깔아야함.   
`pipenv install boto3 --dev`   
```
In [1]: import boto3
 
In [2]: session = boto3.Session(profile_name='fc-8th-s3')
 
In [3]: client = session.client('s3')
 
In [5]: client.create_bucket(
...: Bucket='fc-8th-eb-docker-deploy-mr-dev',
...: CreateBucketConfiguration={
...: 'LocationConstraint':'ap-northeast-2'}
...: )
```  

rds도 만들어 보겠다.   

aws 에 접속해서 만드는데 하라는 대로 그냥 하면 된다..  
내경우 다음과 같이 설정 해줬다.   
```
DB 인스턴스 식별자정보  : eb-docker
마스터 사용자 이름정보  : maro 
 
보안그룹. : RDS Security Group eb-deploy
 
데이터베이스이름 : eb_deploy_rds_dev
```
다만 기본적인 디비 생성은 rds에 하나만 초반에 생성되므로 내경우 추가적으로 만들어 줬다.     

`psql --host=eb-docker.c0zcd.ap-northeast-2.rds.amazonaws.com --user=maro --port=5432 postgres
CREATE DATABASE eb_deploy_rds_pro OWNER maro;`  

rds에 디비가 있는지 확인 하는 방법은     
`psql --host=marolab2.cbzcd.ap-northeast-2.rds.amazonaws.com --user=maro --port=5432 postgres`

특정 이름으로 디비를 조회하는 방법은   
`psql --host=marolab2.cbzcd.ap-northeast-2.rds.amazonaws.com --user=maro --port=5432 ec2_deploy_rds
 `
 
`\dt ` 하면됨.       


---  

## 3. dev, production에 S3, RDS각각 설정 및 runserver확인  

Dockerfile.dev를 사용시와 Dockerfile.production을 사용시 RDS내의 각각 다른 DB, 다른 S3버킷을 사용하도록 설정 했다.   

**.secrets/base.json**  
```
{
"SECRET_KEY" : "",
"AWS_ACCESS_KEY_ID":"",
"AWS_SECRET_ACCESS_KEY":"",
"AWS_DEFAULT_ACL":"private",
"AWS_S3_REGION_NAME":"ap-northeast-2",
"AWS_S3_SIGNATURE_VERSION":"s3v4"
}
```  

**.secrets/dev.json**  
```
{
"DATABASES": {
"default": {
"ENGINE": "django.db.backends.postgresql",
"HOST": "eb-dock ap-northeast-2.rds.amazonaws.com",
"PORT": 5432,
"USER": "maro",
"PASSWORD": "",
"NAME": "eb_deploy_rds_dev"
}
},
"AWS_STORAGE_BUCKET_NAME":"fc-8th-ebdeploy-mr-dev",
}
```  

**.secrets/production.json**  

```
{
"ALLOWED_HOSTS" : [
".elasticbeanstalk.com"
],

"DATABASES": {
"default": {
"ENGINE": "django.db.backends.postgresql",
"HOST": "eb-docker.....ap-northeast-2.rds.amazonaws.com",
"PORT": 5432,
"USER": "maro",
"PASSWORD": "",
"NAME": "eb_deploy_rds_pro"
}
},
"AWS_STORAGE_BUCKET_NAME":"fc-8th-ebdeploy-mr-production"
}
```  


s3 설정을 위해 storage.py만들겠다.   
`pipenv install django-storages`  

**config/settings/sotrages.py**  
```
from storages.backends.s3boto3 import S3Boto3Storage
# S3Boto3 Storage로 STATICFILES_STORAGE설정하신 분들은
# 해제하고 ROOT_DIR/.static 을 STATIC_ROOT로 사용하도록 수정

__all__ = (
'S3DefaultStorage',
)

class S3DefaultStorage(S3Boto3Storage):
    location = 'media'
```

**config/settings/dev.py**       
에서  다음과 같이 주석처리하겠다.   
```
# MEDIA
DEFAULT_FILE_STORAGE = "config.storages.S3DefaultStorage"
# STATICFILES_STORAGE = 'config.storages.S3StaticStorage'
```          

이제 collectstatic으로 밖의 .static폴더로 들어갈 수 있도록 설정하자. 
(collectstatic 할때마다 파일 집어넣는 put요청 너무 많이간다.
달에 2000번 무료라 20번만 하면 끝난다... static파일은 그냥 ec2에서 서빙하도록 하자. 
크게상관없는 이유는  소스 파일로 존제하는 정적파일이라서 ec2에서 보내줘도 사용상 문제 없다. 
(미디어 파일은 문제가 있다. 유저가 올리는거라서. 그래서일단 static과 달리 살려놓음)     

**config/settings/dev.py**    
```
from .base import *
from ..storages import S3DefaultStorage

DEBUG = True
ALLOWED_HOSTS = [

]

WSGI_APPLICATION = 'config.wsgi.dev.application'

INSTALLED_APPS += [
'storages',
]

# DB
DATABASES = secrets['DATABASES']

# Media
DEFAULT_FILE_STORAGE = "config.storages.S3DefaultStorage"
AWS_STORAGE_BUCKET_NAME = secrets["AWS_STORAGE_BUCKET_NAME"]
```   

**config/setting/local.py**  
```
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(ROOT_DIR, '.media')
```  

**config/settings/production.py**   
```
from .base import *

secrets = json.load(open(os.path.join(SECRET_DIR,'production.json')))

DEBUG = False
ALLOWED_HOSTS =secrets['ALLOWED_HOSTS']


WSGI_APPLICATION = 'config.wsgi.production.application'
INSTALLED_APPS += [
'storages',
]
# DB
DATABASES = secrets['DATABASES']

# Media
DEFAULT_FILE_STORAGE = "config.storages.S3DefaultStorage"
AWS_STORAGE_BUCKET_NAME = secrets["AWS_STORAGE_BUCKET_NAME"]
```

STATIC_ROOT, STATIC_URL 를 dev,produtcion에서 공통으로 쓰고 있어서 base로 빼주겠다.      
**confing/settings/base.py**  
```
# Static
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(ROOT_DIR, '.static')


## AWS
AWS_ACCESS_KEY_ID = secrets["AWS_ACCESS_KEY_ID"]
AWS_SECRET_ACCESS_KEY = secrets["AWS_SECRET_ACCESS_KEY"]
AWS_DEFAULT_ACL =  secrets["AWS_DEFAULT_ACL"]
AWS_S3_REGION_NAME =  secrets["AWS_S3_REGION_NAME"]
AWS_S3_SIGNATURE_VERSION =  secrets["AWS_S3_SIGNATURE_VERSION"]
```   

이렇게 base로 뻬준 상태에서 collectstatic 해준다.   
(일반적인 베포에서는 정적파일도 원래 s3로 올리는게 맞기는 함. 
우리는 요금 때문에. )   

이제 각각 환경따라 디비가 다르기대문에 각 local, dev, production에 대해 migration 먼저 해준다.    


각각 모드에서 환경변수 export 해준후 runserver잘 되는지 이제 확인해보자.  

local -> 잘된다. 
dev -> psycopg2 없어서 애러떴고 `pipenv install pycopg2-binary` 로 설치후 런서버 확인했다.   
production -> error log파일 없다고 애러뜸   
```
ValueError: Unable to configure handler 'file_error': [Errno 2] No such file or directory: '/var/log/django/error.log'
```   
production모드에서 runserver시에 /var/log/django가 dockerfile에 설정에 의해  생성되지 않은 상태이기 때문에...
궂이 production 모드를 local 에서 runerver해줄 일은 없겠지만....
이것을 docker로 ec2에서 생성해 줄때랑 단순히 로컬에서 runserver해줄때의 경우를 분리해 줘야 한다.   

/var/log/django/error.log  이 디렉토리 존제하면 이걸 쓰고    
없으면 ebdocker 이하의 디렉토리에 생성하고 쓰도록 하자 .
```
LOG_DIR = '/var/log/django'
if not os.path.exists(LOG_DIR):
LOG_DIR = os.path.join(ROOT_DIR, '.log')
if not os.path.exists(LOG_DIR):
os.makedirs(LOG_DIR)
```  

런서버가 잘 됬으니 각각에 settings에 입력한 mediaroot대로 잘 파일이 업로드 되는지 확인해 보도록하자.   

위에서 local에만 미디어 유알엘, 미디어 루트 넣어줬다.
일단 로컬에서 미디어(유저 model의 이미지) 올렸을때 되는지 안되는지 확인해보자.     

그뒤 , 각 dev,production 환경에서 미디어 올려서 확인해보자.   

---   



## 4.User에 img_profile필드 추가  

파이썬 이미징 라이브러리부터 설치한다.   
`pipenv install pillow`   

다음과 같이 admin, models 코드 추가한다.    

**app/members/admin.py**  
```
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.utils.translation import gettext_lazy as _

from.models import User

class UserAdmin(BaseUserAdmin):
fieldsets = (
(None, {'fields':('username','password')}),
(_('Personal info'), {'fields': ('first_name', 'last_name', 'email', 'img_profile')}),
(_('Permissions'), {'fields': ('is_active', 'is_staff', 'is_superuser',
'groups', 'user_permissions')}),
(_('Important dates'), {'fields': ('last_login', 'date_joined')}),
)

admin.site.register(User,UserAdmin)
```   

**app/members/models.py**   
```
from django.db import models

from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
img_profile = models.ImageField(upload_to="user", blank=True)
```

이후 각각의 환경에서 변경사항을 migration해준다. 
  
  
## 5. DEBUG=True일 때 MEDIA_URL동작 설정   

**urls.py**  
```
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path

urlpatterns = [
path('admin/', admin.site.urls),
]+ static(
prefix=settings.MEDIA_URL,
document_root=settings.MEDIA_ROOT,
)
```  


만약 page not found 뜬다면 ?    
\+ static(  
prefix=settings.MEDIA_URL,  
document_root=settings.MEDIA_ROOT,  
)  
이거 url에 추가해줘서 로켓페이지 안뜬거다.    
기본에서 하나더 추가해주면 확인할 필요없으니까인듯. 뷰를 만들어주면된다. 

애러가 한가지 더있다.     
urls,py에 static(prefix, = settings.MEDIA_URL, document_root= settings.MEDIA_ROOT) 를 추가할 시에 
local에서만 런서버 되고 dev, production 에서는 안된다.  
----------> dev, production 에는 MEDIAL_URL, MEDIA_ROOT설정 없고 DEFAULT_FILE_STORAGE 만 있어서 인듯. 
위의 링크에 보이듯이 base로 MEDIA_URL, MEDIA_ROOT설정을 빼줬다. 
**base.py**
```
# Media
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(ROOT_DIR, '.media')
```  

















