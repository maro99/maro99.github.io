---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기05"
categories:  
- AWS  
tags:  
- Python    
- Docker      
- ElasticBeanstalk    
- deploy    
- nginx     
- uwsgi     
- platform hook   
- ebextensions   
comments : true    
date: "2018-09-09 18:50"  
---        

## 1. Dockerfile FROM을 개인저장소로 지정|(Dockerhub 사용 )    

지금 같은 경우는 우리가 쓰는 페키지 굉장히 적다.    
Dockerfile관찰해보면    
```
RUN apt -y update && apt -y dist-upgrade
RUN apt -y install build-essential
RUN apt -y install nginx supervisor
```  

파이썬 3.6.5 받아서 업데이트 하고 ~~다깔아도 시간 별로 안걸린다.   
하지만 이 기본 페키지 20개면 한번 Docker배포시마다 한번 build할때 이걸 처음부터 끝까지 빌드한다.   
정말 오래 걸릴 것이다.    
마치 base이미지 처럼... 앞부분을 담당. 문제는 이것이 local에만 있어서 Dockerfile에서는 사용이 불가 하다.   

Dockerfile쪼개서 base처럼 큰부분 웹에 미리 올려놓고 배포할때 가져다씀. 배포속도 아주 빨라진다.   

[https://hub.docker.com/](https://hub.docker.com/) 에 repository를 생성하고    

이 repository에 docker image를 push 해야한다.(마치 git같다. )
   
이때 로컬dockerfile 에서 파이썬 받을대처럼.. 같은 방식으로  nadcdc/fc-8th-eb-docker  유저명/저장소명 으로 가져다쓴다.     

먼저 우리 프로젝트에서 도커 기존 이미지 관찰해보자.   
`docker images`

그중 none표시 된것 지우는 코드      
`docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q) && docker rmi -f $(docker images -a -f 'dangling=true' -q)`     

docker images    
해보면 오래전 만든 base테그 가진 docker 이미지 있지만 달라졌을수도 있으니 새로 빌드해주자.   

./build.py -> base 빌드해준다.( 서버에서 현제는 이과정에 배포시마다 매번 이뤄지고 있다. )   

docker image해서 보면           
base테그 가진 도커 이미지는 500인데 그중 일부인 python slim 은 140이다...400정도를 매번 서버 배포시 비효율적으로  해주고 있던것.  

ec2-deploy ..dev .. df8e2bb27858 ..6 weeks ago .. 1.31GB  

이런식으로 하나의 이름으로 이루어 진것은 
공식 래파짓 토리거나 ,  로컬 래파짓토리 중 하나다. 
개인 명의의 온라인 래파짓 토리 가져올려면 nadcdc/ec2-deploy이런식으로 해야한다.      


`docke tag`
SOURCE_IMAGE를 참조하는 TARGET_IMAGE 태그를 만든다.   
우리가 만든 이미지가 eb-docker:base 테그 가지고 있다. ->이것을  nadcdc/fc-8th-eb-docker:base에 올려야한다. (hub.docker에)  
  
eb-docker:base  이 레파짓토리 명을 똑같이 가진 다른 이미지를 또 생성해야함.  
(eb-docker:base  이 이미지와 같은 레이어 가지는 다른 이미지 만든다.  여러개 이미지 있어도 각각의 이미지가 같은 레이어 참조하면 용량 차지 안한다.)   
```
                    원본이되는것      새로생성할 것 
docker tag eb-docker:base nadcdc/fc-8th-eb-docker:base
````

`docker images` 다시 해보면 

```
REPOSITORY                  TAG                 IMAGE ID                 CREATED                 SIZE
eb-docker                    base                 4aa3f9138850         10 minutes ago         549MB
nadcdc/fc-8th-eb-docker     base                 4aa3f9138850         10 minutes ago         549MB
```   

같은사이즈~   
래파짓토리는 다른데 tag는 같은 것 볼 수 있다.    
같은 곳을 차지해서?(같은 레이어를 가져서?) 이미지 아이디는 같다.    
   
이 테그명을 우리만든 레파짓이랑 같이하는게 좋다.(이것이 중요하다.)   

`docker login`  
`docker push nadcdc/fc-8th-eb-docker`   
  
hub.docker 들어가 보면 테그에 이미지 생성되어있다.   
  
다시한번 도커 푸쉬 해보면 순식간에 됨.
      
이이미지를 elasticbenastalk ec2 에서 배포과정에서 쓴다면 이 도커헙에서 이것만 받아서 따운받아서 풀면 바로사용할수있는 상태가됨. 
pipenv까지 다 설치된 상태로 !    
  
(베이스이 이미지 변경 안하고 ) 도커 푸쉬다시 해보면 금방 끝난다.   
 베이스 이미지 변경 안하면 계속 같은 이미지 쓴다는것.
  
( git status 처서 pip파일 변경된 경우에는 파이썬에서 파악해서 지금 처럼 베이스이미지 만들어서  dockerpush  하는것이 좋다.  
그 후에 배포까지 하는 스크립트 
나중에  만들어서  자동화 해라. )  


**Dockerfile**  다음과 같이 수정해 준다. 

```
FROM nadcdc/fc-8th-eb-docker:base #Dplcerfile FROM을 개인 저장소로 지정.
MAINTAINER nadcdc4@gmail.com



#이하 production에서 복사.( 여기 위의 base에 있던 내용들 다 삭제했다. )

ENV PROJECT_DIR /srv/project
ENV BUILD_MODE production

#nginx ,supervisor install
ENV DJANGO_SETTINGS_MODULE config.settings.${BUILD_MODE}


# Copy projects files
COPY . ${PROJECT_DIR}
#WORKDIR ${PROJECT_DIR}


# 로그파일 기록 위한 폴ㄷ더 생성
RUN mkdir /var/log/django

# Ngnix config
RUN cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/nginx.conf \
/etc/nginx/nginx.conf && \

# available에 nginx_app.conf파일 복사
cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/nginx_app.conf \
/etc/nginx/sites-available/ && \

# 이미 sites-enabled에 있던 모든 내용 삭제
# rm -f /etc/nginx/sites-enabled/* && \

# available에 있는 nginx_app.conf를 enabled로 링크.
ln -sf /etc/nginx/sites-available/nginx_app.conf \
/etc/nginx/sites-enabled

# Supervisor 설정복사
RUN cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/supervisor.conf \
/etc/supervisor/conf.d

# 7000번 포트 open
EXPOSE 7000

# RUN supervisor
CMD supervisord -n
```  

Dockerfile 을 add 해주고 ./deploy.sh 해본다.    
   
이때 EC2 서버에서 하는일은 도커헙에서 따운 받아서 압축 풀어서 도커런   
콜렉트 스테틱 까지!    
   
eb open하면 잘된다.            

---  


## 2. collectstatic자동화한것처럼. files이용해서 superuser생성       
[Writing custom django-admin conmmands](https://docs.djangoproject.com/ko/2.1/howto/custom-management-commands/) 이 문서 보면서 진행해보겠다.   

./manage.py 그대로 커맨드창에서 처보면  이렇게 뜬다. 

```
Type 'manage.py help <subcommand>' for help on a specific subcommand.

Available subcommands:
```

[auth] (우리가 아래 chpassword, crsuperuser할 수 있는 이유는 auth app이 있기 때문..--->실제로 이 app에 가보면 내용 있다.< django의 auth 엡 )  
```
    changepassword  
    createsuperuser  
  
[contenttypes]  
    remove_stale_contenttypes  
```
 
[django]( 이 내용들이 장고 app에서 쓰는 command들. 우리는 특정 app에 커맨드 추가할 수 있다.  그것이 custom command)  
```
    check  
    compilemessages
    createcachetable
    dbshell
    diffsettings
    dumpdata
    flush
    inspectdb
    loaddata
    makemessages
    makemigrations
    migrate
    sendtestemail
    shell
    showmigrations
    sqlflush
    sqlmigrate
    sqlsequencereset
    squashmigrations
    startapp
    startproject
    test
    testserver

[members]
    createsu

[sessions]
    clearsessions

[staticfiles]
    collectstatic
    findstatic
    runserver (런서버같은경우 여기있다. )
```   

문서 다시 보겠다.   
```
Applications can register their own actions with manage.py. For example, you might want to add a manage.py action for a Django app that you're distributing. In this document, we will be building a custom closepoll command for the pollsapplication from the tutorial.  
To do this, just add a management/commands directory to the application. Django will register a manage.py command for each Python module in that directory whose name doesn't begin with an underscore. For example:

응용 프로그램은 자신의 작업을 관리(manage.py) 에 등록할 수 있습니다.예를 들어 관리를 추가할 수 있습니다.배포하려는 Django 앱에 대한 py action. 이 문서에서는 튜토리얼에서 폴링 응용 프로그램에 대한 사용자 정의 클로즈폴 명령을 작성합니다. 이렇게 하려면 응용 프로그램에 관리/명령 디렉터리를 추가하십시오. 장고는 관리를 등록할 것이다.이름이 밑줄로 시작하지 않는 디렉토리의 각 Python 모듈에 대한 py 명령입니다. 예를 들면 다음과 같다.

polls/
 __init__.py
 models.py
 management/
 commands/
 _private.py
 closepoll.py
 tests.py
 views.py
In this example, the closepoll command will be made available to any project that includes the polls application in INSTALLED_APPS.
The _private.py module will not be available as a management command.
The closepoll.py module has only one requirement -- it must define a classCommand that extends BaseCommand or one of its subclasses.

이 예에서 클로즈폴 명령은 INSTALLED_APPS의 폴링 응용 프로그램을 포함하는 모든 프로젝트에서 사용할 수 있게 됩니다. 사적py 모듈은 관리 명령으로 사용할 수 없습니다. closepoll.py 모듈에는 BaseCommand 또는 해당 하위 클래스 중 하나를 확장하는 클래스 명령이 정의되어 있어야 합니다.
```  


우리가 사용하는 app 안에 디렉토리를 일단 만든다.   

```
우리가 사용하는 app 안에 디렉토리를 일단 만든다.
app/members/management/commands/createsu.py
```  

./manage.py 해보면 createsu 들어가있다.   
  
문서 보고 createsu.py 작성하자.   

**createsu.py**    
```
from django.core.management import BaseCommand

class Command(BaseCommand):
    def handle(self, *args, **option):
        print('createsu !!!!!!!')
        User.objects.create
```
./manage.py createsu 해보면 잘  출력됨.   

우리가 지금 쓰고 있는 user가 장고에있는 기본 유저   AbstractUser상속받아서 사용하고 있다.   
따라가 보면 이것은 기본적으로 Usermanger가지고 있는데  
Usermanagr e는 createsuperuser 사용하면 된다.  
  
그런데 우리는 Usre모델을 AbstractUser상속받아 어쨌든 새로 만들었다.    
그러니 이 User의 기본 manager 말고   
User manager를  새로 정의해서  쓰는 것이 좋겠다  

**app/menmbers/ models.py**  이것을 가져다 써보자.    
```
from django.db import models

from django.contrib.auth.models import AbstractUser, UserManager as DjangoManager


class UserManager(DjangoManager): # 이렇게 오버라이드 해주면 UserManager동작을 우리가 언제든지 바꿀 수 있다.
pass


class User(AbstractUser):
img_profile = models.ImageField(upload_to="user", blank=True)

objects = UserManager()
```

secret에 유저 정보 넣어놓자.    

**.secrets/base.json**  
```
{
"SECRET_KEY" : "qbqz%r_z40#k!0p1uzed23cte0&s6+",

"AWS_ACCESS_KEY_ID":"",
"AWS_SECRET_ACCESS_KEY":"",
"AWS_DEFAULT_ACL":"private",
"AWS_S3_REGION_NAME":"ap-northeast-2",
"AWS_S3_SIGNATURE_VERSION":"s3v4",


"SUPERUSER_USERNAME":"maro6",
"SUPERUSER_EMAIL":"",
"SUPERUSER_PASSWORD":"",
```    

**createsu.py**
```
import json
import os

from django.contrib.auth import get_user_model
from django.core.management import BaseCommand

from config import settings

User = get_user_model()

class Command(BaseCommand):
def handle(self, *args, **option):
secrets = json.load(open(os.path.join(settings.base.SECRET_DIR, 'base.json')))
if not User.objects.filter(username=secrets['SUPERUSER_USERNAME']).exists():
User.objects.create_superuser(
username=secrets['SUPERUSER_USERNAME'],
password=secrets['SUPERUSER_PASSWORD'],
email=secrets['SUPERUSER_EMAIL'],
)
```  

---  


## 3.custom settings backend 로 사용자 인증처리  <admin아이디로 로그인시 자동 생성 및 인증 되도록함  

앞서 한 방법보다 좋은 방법이    
우리가 본 백엔드에 이미 있다.    
settings backend기억 나는가 ?   
제이슨 파일 없이  사용자 인증 처리하도록   settings이용해서 백엔드 만들 수 있다.  
커멘드로 직접 만드는것 말고 만들어보자.  
  
위와 같은 경우에는 관리자 비번 등을 계속해서 유지해햐 한다. ( 이번경우에는 해쉬된 값을 가지고 있어도 되는게 차이인듯.)   
위에서는 caommnd로 createsu 처리한것.  
createsu는  superuser 우리가 직접 만들어 준것.      

admin /특정비밀번호
위 값으로 로그인 시도시 authenticate가 성공하도록 커스텀 Backend를 작성 해보자. 
members.backends모듈에 작성하고 
Backend 명은 Settings.Backend   
 password는 장고안의 문자열을 생성해서 써보겠다.   
 
 `app/ ./manage.py shell_plus`  입력후   
```
In [2]: from django.contrib.auth.hashers import make_password
In [3]: make_password('@@@@@')
Out[3]: 'pbkdf2_sha256$100000$zNIjdY5qwrae$dASPtmQ/vw7VQ9cFD69aYu7hTxTLQLoFFzLqUzxtq1I='
```

이함수를 사용해서 만들어진 문자열 이용해서 settings backend있는 내용을 참조해서   
custom  인증으로 db에 그 유저가 없다면 해당 유저 만들어 주고   
그렇지 않고 있다면 그 유저를 리턴해주는 방식   
settings backend 이해하고 custom backend에 적용해보자.  
(createsuperuser안해도 
user가 자동으로 로그인 하면 생성을 하거나 있는것 가져오가나 하도록 하자.(get_or_create쓰라는 것인듯.)


**settigns/base.py**

```
# Auth
ADMIN_USERNAME = 'admin'
ADMIN_PASSWORD = 'pbkdf2_sha256$120000$aAwW76yogDKH$1rHLC3JH5MkKaXdQSAYIg+PIrtIc/X3MWqsROuad1Wc='
AUTH_USER_MODEL = 'members.User'
AUTHENTICATION_BACKENDS=[
'django.contrib.auth.backends.ModelBackend',
'members.backends.SettingsBackend',
]
```

**members/backends.py**   



```
# admin /특정비밀번호
# 위 값으로 로그인 시도시 authenticate가 성송하도록 커스텀 Backend를 작성
# members.backends모듈에 작성
# Backend 명은 Settings.Backend
# password는 장고안의 문자열을 생성해서? 써야함.
from django.conf import settings
from django.contrib.auth import get_user_model
from django.contrib.auth.hashers import check_password

User = get_user_model()


class SettingsBackend:
'''
ADMIN = 'admin'
ADMIN_PASSWORD ='pbkdf2_sha256$100000$zNIjdY5qwrae$dASPtmQ/vw7VQ9cFD69aYu7hTxTLQLoFFzLqUzxtq1I='
'''

def authenticate(self, request, username=None,password=None):
login_valid = (settings.ADMIN_USERNAME == username) # 입력한 username과 settings의 username같은지 판별.
pwd_valid = check_password(password, settings.ADMIN_PASSWORD) # pwd유효한지 판단.

if login_valid and pwd_valid: # username, password 일치시에
try:
user = User.objects.get(username=username)
except User.DoesNotExist: # User가 없을 경우에 만들어준다.
# Create a new user. There's no need to set a password
# because only the password from settings.py is checked.
user = User(username=username) # createsuperuser는 password반드시 넣어줘야 되서 안쓰는듯.
user.is_staff = True # 이 계정은 password궂이 필요없는 계정.( 어짜피 위에서 입력시마다 authenticate로 검사하니까. <db 에서 가져오는것이 아님.> )
user.is_superuser = True
user.save()
return user # user 를 get했다면 여기서 반환. 없응면 except안에서 만들어서 반환.
return None # pwd,username유효하지 않은경우는 아무것도 돌려주지 않는다.


def get_user(self, user_id): #이건 문서그대로 넣는다. (custom하기 전과 같은듯.)
try:
return User.objects.get(pk=user_id)
except User.DoesNotExist:
return None
```


**로컬에서 runserver해서 테스트 해보자.**  
admin으로 로그인 하면 비번 없는 유저로 자동으로 생성됨.

dev, local 양쪽에서 다 확인할 수 있다. 
local에서 런서버해서 로그인해봄.---->sqlite 뷰어로 디비 확인해봄. (페스워드 안들어있는것 확인가능하다. )    

**settings**
```
AUTHENTICATION_BACKENDS=[
'django.contrib.auth.backends.ModelBackend',
'members.backends.SettingsBackend',
]
```   

settings설정 보면   
첫번째 백엔드에서 인증 실패해서 우리가 작성한 custom backend로 넘어가서 인증성공한 것이다.   
두번째 백엔드에서 pwd, username비교결과   
settings의 값들과 같네 ~~~라는것 인식하고 ~~except해서 비번 없는 유저 생성한것.  
  
이런식으로 하면   
우리가 궂이 superuser만들어준 필요없다.   
settings에 이미 다 박혀있니까.(hash값만 잘 기억하자.)  

---  


## 4.만약에 배포중에 ec2가 여러개이고 그중에 딱 한개에서만 슈퍼유저 생성하고 싶다면 ?   

배포후  아예 어드민 만들 필요가 없지만
가끔 배포중에 만들고 싶을때가 있다. ---->createsu 실행해라.

**.ebextensions/01_files.config** 안에 


```
"/opt/elasticbeanstalk/hooks/appdeploy/post/02_createsu.sh":
mode: "000755"
owner: root
group: root
content:
#!/usr/bin/env bash
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py migrate --noinput
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py createsu --noinput
```

이런식으로 작성하면 될것. 


createsu가 만약 이미 superuser있다면 생성 안할것임. 
이러한 동작이 중복이 되어도 크게 상관 없지만.   
  
**만약에 배포중에 ec2가 여러개이고 그중에 딱 한개에서만 슈퍼유저 생성하고 싶다면 ?**   

[Linux 서버](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/customize-containers-ec2.html) 이 문서를 보자.    
```
컨테이너 명령
container_commands 키를 사용하여 애플리케이션 소스 코드에 영향을 주는 명령을 실행할 수 있습니다. 컨테이너 명령은 애플리케이션과 웹 서버를 설정하고 애플리케이션 버전 아카이브의 압축을 푼 후 애플리케이션 버전을 배포하기 이전에 실행됩니다. 비컨테이너 명령과 기타 사용자 지정 작업은 추출하려는 애플리케이션 소스 코드보다 먼저 수행됩니다.
지정한 명령은 루트 사용자로 실행되며, 이름의 영문자 순서대로 처리됩니다. 컨테이너 명령은 준비 디렉터리에서 실행됩니다. 준비 디렉터리는 소스 코드를 애플리케이션 서버에 배포하기 이전에 추출하는 곳입니다. 컨테이너 명령을 사용하여 준비 디렉터리에서 소스 코드를 변경한 경우 변경 사항은 코스를 최종 위치에 배포할 때 포함됩니다.
컨테이너 명령 관련 문제를 해결하려면 인스턴스 로그의 출력을 확인합니다.
테스트 명령이 true로 평가될 때 test를 구성하거나, leader_only를 사용하여 단일 인스턴스에서 명령을 실행하기만 하면 됩니다. Leader-only 컨테이너 명령은 환경을 생성하고 배포하는 중에만 실행되고, 다른 명령 및 서버 사용자 지정 작업은 인스턴스를 프로비저닝하거나 업데이트할 때마다 수행됩니다. Leader-only 컨테이너 명령은 시작 구성을 변경(예: AMI ID 또는 인스턴스 유형 변경)하더라도 실행되지 않습니다.  


구문
 
container_commands:
name of container_command:
command: "command to run"
leader_only: true
name of container_command:
command: "command to run"  

옵션  

command        실행할 문자열 또는 문자열 배열입니다.
env            (선택 사항) 명령을 실행하기 이전에 기존 값을 재정의하여 환경 변수를 설정합니다.
cwd            (선택 사항) 작업 디렉터리입니다. 기본적으로 압축 해제된 애플리케이션의 준비 디렉터리입니다.
leader_only    (선택 사항) Elastic Beanstalk에서 선택한 단일 인스턴스에 대해서만 명령을 실행합니다. Leader-only 컨테이너 명령은 다른 컨테이너 명령보다 먼저 실행됩니다. 명령은 leader-only이거나 test를 포함할 수 있으나, 둘 다 사용할 수 없습니다(leader_only가 우선 적용됨).  


test           (선택 사항) 컨테이너 명령을 실행하려면 true를 반환해야 하는 테스트 명령을 실행합니다. 명령은 leader-only이거나 test를 포함할 수 있으나, 둘 다 사용할 수 없습니다(leader_only가 우선 적용됨).
ignoreErrors (선택 사항) 컨테이너 명령이 0이 아닌 값을 반환하는 경우(성공) 배포에 실패하지 않습니다. 활성화하려면 true로 설정합니다
```   

leader_only:ture의   
```
    " leader_only를 사용하여 단일 인스턴스에서 명령을 실행하기만 하면 됩니다" 
```    
이말은 즉  **ec2가 여러개여도 단 하나가 ledaer ec2 가 되서 하나에서만 실행된다.**

( 이 container command의 옵션  안쓰고 궂이 files 쓴 이유는? ----> 이것은 배포 이전에 실행되는 옵션인데 우리는 배포 이후에 실행을 원했다.)    


**이것을  혼합해서 쓸 수 있다.**  

```
                container_commands              
                        무조건 배포 이전에 작동 
                        leadr_only 옵션 존재
                         -----------> leader에만 "특정파일"을 생성

                files에 넣은 스크립트:
                    배포 이후에 작동하도록 할 수 있음 
                    leader_only옵션이 없음  
                    ------------> "x특정 파일" 이 있을 경우에만 작동.

```

**.ebextensions/00_commands.config**  
```
container_commands:
01_createsu:
command: "touch /tmp/createsu"
leader_only: true # leader격인
```  

**.ebextensions/01_file.config**  
```
files:
"/opt/elasticbeanstalk/hooks/appdeploy/post/01_collectstatic.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py collectstatic --noinput



"/opt/elasticbeanstalk/hooks/appdeploy/post/02_createsu.sh": # post다음 실행된다 해서 넣어봄...
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
if [ -f /tmp/createsu ] # 파일이 있으면 이하를 실행.
then
sudo docker exec `sudo docker ps -q` python
fi
```  


00_commands.config에서   
leader_only 옵션으로 파일을 넣기는 하지만   
기왕이면  01_file.config에서 실행하는 것들도     containercomand에 넣어 놓으면 좋겠다.    
(container에 모든 관련 코드 넣어놓고 leader_only옵션의 존제 유무를 한번에 보는것이 더 편할것.)  

migrate도 적어주자.   
**.ebextensions/00_commands.config**   
```
container_commands:
01_collectstatic:
command: "touch /tmp/collectstatic"

02_createsu:
command: "touch /tmp/createsu"
leader_only: true # leader격인 ec2에서만 파일 생성.

03_migrate:
command: "touch /tmp/migrate"
leader_only: true
# migrate 는 한번만 해도됨 사용하는 rds는 하나니까. leader에서만 해도됨.
# w지금까지 export환경하고 배포하고 migrate일일이 해줬는데 그럴필요없이 배포시 자동으로
```  

**.ebextensions/00_commands.config**
```
container_commands:
01_collectstatic:
command: "touch /tmp/collectstatic"

02_migrate:
command: "touch /tmp/migrate"
leader_only: true
# migrate 는 한번만 해도됨 사용하는 rds는 하나니까. leader에서만 해도됨.
# w지금까지 export환경하고 배포하고 migrate일일이 해줬는데 그럴필요없이 배포시 자동으로 migrate되도록 하자는것.

01_createsu:
command: "touch /tmp/createsu"
leader_only: true # leader격인 ec2에서만 파일 생성.
```      

**01_files.config**   
```
 files:
"/opt/elasticbeanstalk/hooks/appdeploy/post/01_collectstatic.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
if [ -f /tmp/collectstatic ] # 파일이 있으면 이하를 실행.
then
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py collectstatic --noinput
fi

"/opt/elasticbeanstalk/hooks/appdeploy/post/02_migrate.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
if [ -f /tmp/migrate ] # 파일이 있으면 이하를 실행.
then
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py migrate --noinput  # 그냥 마이그레이트 할지 옵션 선택 안해도됨 
fi


"/opt/elasticbeanstalk/hooks/appdeploy/post/03_createsu.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
if [ -f /tmp/createsu ] # 파일이 있으면 이하를 실행.
then
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py createsu
fi
```
migrate먼저 해주는것 주의.
여기도 if~ 다 적어줘서 궂이 command쪽 안봐도 의미 해석 한번에 가능하다.    
  
 
이렇게하면 배포중에     
createsu, collectstatic, migrate가 다 자동으로 이루어진다.   

  
./deploy해보고   
유저 있으면 지우고 admin로그인 해본다 -----> 된다.   
  
amdin 지우고 ./deploy해보면  createsu에 설정한 유저 잘 생긴다.   
   
collectstatic도 자동으로 잘 하게 되어있어야 한다.   

