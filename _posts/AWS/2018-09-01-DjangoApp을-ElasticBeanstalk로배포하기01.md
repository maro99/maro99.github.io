---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기01 "
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
date: "2018-09-01 18:50"  
---


## 1. 프로젝트 초기 설정      

먼저 다음과 같이 디렉토리구성 및 기본 파일을 추가했다.   

```  
projects/
    deploy/
          eb-docker-deploy/
              git, gitignore ->/.secrests추가
              pipenv install django
              PyCharm interpreter
              .secrets/base.json (django secret key generator이용 )
              settings 패키지화-> settings/base.py -> base.json으로부터 SECRET_KEY할당.
                                  settings/local.py

```     


## 2. CustomUsermodel만들기     

`manage.py startapp members`  
 
**app/members/models.py**  
```
from django.contrib.auth.models import AbstractUser
 
class User(AbstractUser):
    pass
 
```  
 
**app/members/admin.py**    
```   
from django.contrib import admin  
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
 
from .models import User
 
class UserAdmin(BaseUserAdmin):
    pass
 
admin.site.register(User, UserAdmin)
```
 
 
**app/confing/settings/base.py**  
 
```
AUTH_USER_MODEL = 'members.User'
```

## 3.Dockerfile작성      

**Dockerfile.base**  

```
FROM python:3.6.5-slim  slim- 최소한 필요한것만 깔림.
MAINTAINER nadcdc4@gmail.com
 
RUN apt -y update && apt -y dist-upgrade
```

`docker build -t eb-docker:base -f Dockerfile.base . `

 
 
**Dockerfile.local**  
```
FROM eb-docker:base
MAINTAINER nadcdc4@gmail.com
 
COPY . /srv/project
```

`docker build -t eb-docker:local -f Dockerfile.local .`   

도커런 해서 들어간 뒤 runserver해보겠다.    
`docker run --rm -it -p 9994:8000 eb-docker:local /bin/bash`

``` 
root@19a7f536f0a0:/# cd srv/project
root@19a7f536f0a0:/srv/project# ls -al
root@19a7f536f0a0:/srv/project# cd app
root@19a7f536f0a0:/srv/project/app# pip install django

 export DJANGO_SETTINGS_MODULE=config.settings.local   
^Croot@19a7f536f0a0:/srv/project/app# python manage.py runserver 0:8000  
```

http://localhost:9994 들어가보면 로켓 나온다.        


## 4. EB 시작, build.py작성       
[EB CLI 설치문서](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/eb-cli3-install.html) 참고중    
`pipenv install awsebcli --dev`  

pipfile가지고 그냥 설치하면 좋겠다.     
pipenv 이용해서 requriements만드는 방법이 있다.  
pipfile에 장고 설치됬다는 내용 밖으로 빼줬다가 해주고싶은데  
도커파일 안쪽에서는 힘들다.    
   
--------->해결법   
local 에서   
pipenv lock --requirements  
pipenv lock --requirements --dev 호출시   
출력되는 내용 가지고 requriement생성후 적용후 지워주고싶다.  
스크립트를 만들자.    

**build.py**  
```
#!/usr/bin/env python
import os
import subprocess
 
try:
    # pipenv lock으로 requirements.txt.생성
    subprocess.call('pipenv lock --requirements > requirements.txt' ,shell=True)
    # docker.build
    subprocess.call('docker build -t eb-docker:base -f Dockerfile.base .', shell=True)
 
finally:
    # 끝난 후 requirements,txt파일 삭제.
    os.remove('requirements.txt')
```  

 
**Dockerfile.base**    
에 다음을 추가     
```
# 로컬의 requirements.txt파일을 /srv에 복사후 pip install 실행
# (build하는 환경에 requirements.txt.가 있어야함!)
COPY ./requirements.txt /srv/
RUN pip install -r /srv/requirements.txt
```    

 
**Dokcerfile.local**

```
FROM eb-docker:base
MAINTAINER nadcdc4@gmail.com
 
# 이하가 다 바뀌었다.   
ENV BUILD_MODE local
ENV DJANGO_SETTINGS_MODULE config.settings.${BUILD_MODE}
 
COPY . /srv/project  
```  
  
지금은 build.py가 base만 빌드해 주고 있다.   
빌드를 실행할때.  
./build,py -m base  
./build,py -m local  
 
해서 각각을 빌드해 주고 싶다.  
   
그쟝 실행하면  
   
Select mode  
1:base  
2:local  
3: production  
Choice  
   
이중 선택할 수 있도록 [이것](https://docs.python.org/3/library/argparse.html) 참고해서 바꿔보자.  

**./build.py**
```
#!/usr/bin/env python
import os
import subprocess
import argparse



# 사용자가 입력한 mode

def build_base():
    print('build_base_called')

    try:
        # pipenv lock으로 requirements.txt 생성
        subprocess.call('pipenv lock --requirements > requirements.txt', shell=True)
        # docker.build
        subprocess.call('docker build -t eb-docker:base -f Dockerfile.base .',shell=True)

    finally:
        # 끝난 후 requirements.txt 파일 삭제
        os.remove('requirements.txt')

def build_local():
    print('build_local_called')

    try:
        # pipenv lock으로 requirements.txt 생성
        subprocess.call('pipenv lock --requirements > requirements.txt', shell=True)
        # docker.build
        subprocess.call('docker build -t eb-docker:local -f Dockerfile.local .',shell=True)

    finally:
        # 끝난 후 requirements.txt 파일 삭제
        os.remove('requirements.txt')


def mode_function(mode):

    if mode =='base':
        build_base()
    elif mode == 'local':
        build_local()
    else:
        raise ValueError(f'{MODES} 에 속하는 모드만 가능합니다.')



if __name__ =='__main__':

    MODES = ['base', 'local']

    # ./build.py --mode <mode>
    # ./build.py -m<mode>
    parser = argparse.ArgumentParser()
    parser.add_argument('-m', '--mode',
                        help='Docker build mode[base,local]'
                        )

    args = parser.parse_args()

    # 모듈 호출에 옵션으로 mode를 전달한 경우
    if args.mode:
        mode = args.mode.strip().lower()
        mode_function(mode)

    # 옵션을 입력하지 않았을 경우 (./build.py)
    else:
        while True:
            print('Select mode')
            print('1.base')
            print('2.local')
            selected_mode = input('Choice:')

            try:
                mode_index = int(selected_mode) -1
                mode =MODES[mode_index]
                break
            except IndexError:
                print('1~2 번을 입력하세요')

    # 선택된 mode에 해당하는 함수를 실행
    mode_function(mode)

```    

`./build.py -m local`    
`docker run --rm -it -p 9994:8000 eb-docker:local python /srv/project/app/manage.py runserver 0:8000`  


추가적으로 docker run 시에 runserver명령 따로 안서줘도 실행되도록 Dockeffile안에 runserver CMD 남겨놓음  

**Dockerfile.local**  

```
# 파일 복사 후 runserver 0:8000 실행
FROM eb-docker:base
MAINTAINER nadcdc4@gmail.com

ENV BUILD_MODE local
ENV DJANGO_SETTINGS_MODULE config.settings.${BUILD_MODE}

COPY . /srv/project

WORKDIR /srv/project/app
CMD python manage.py runserver 0:8000
```
`docker run --rm -it -p 9994:8000 eb-docker:local`     



## 5.Dockerfil.dev작성해 dev mode로 docker실행하도록함      

docker dev 환경설정 후, localhost:9999 welcome 페이지 보는것을 해보겠다.     

다음과 같은 작업이 필요하다.   
1. `.config - dev` 디렉토리 생성 후, uwsgi, nginx, supervisrod 설정하기   
2. dockerfile.dev 에서 supervisord 실행   
3. localhost:9999 들어가서 welcome 페이지 보기   
 
다음과 같은구성이 추가된다.     
```
settings/dev.py만들고 
wsgi페키지화후 dev만들고 
.config
    dev
         uwsgi.ini
          supervisor.conf
          nginx.conf
          nginx_app.conf 다 있어야 한다. 
```

먼저 uwsgi인스톨해줌      
`pipenv install uwsgi`    

**Docker.base**    
```
FROM python:3.6.5-slim
MAINTAINER nadcdc4@gmail.com


RUN apt -y update && apt -y dist-upgrade

RUN apt -y install build-essential  # 코드 빌드할때 필요한 필수 페키지. 그냥 docker run해서 들어가서 uwsgi설치하려하면 안된다.  
# nginx, supervisor install
RUN apt -y install nginx supervisor 

# 로컬의 requriements.txt. 파일을 /srv 에 복사한 후 pip install 실행
# (build 하는 환경에 requirements.txt 가 있어야 함.)
COPY ./requirements.txt /srv/
RUN pip install -r /srv/requirements.txt  

```  
  
**nginx.conf**
```
user root;
daemon off;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
worker_connections 768;
# multi_accept on;
}

http {

##
# Basic Settings
##

sendfile on;
tcp_nopush on;
tcp_nodelay on;
keepalive_timeout 65;
types_hash_max_size 2048;
# server_tokens off;

# server_names_hash_bucket_size 64;
# server_name_in_redirect off;

include /etc/nginx/mime.types;
default_type application/octet-stream;

##
# SSL Settings
##

ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
ssl_prefer_server_ciphers on;

##
# Logging Settings
##

access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;

##
# Gzip Settings
##

gzip on;
gzip_disable "msie6";

# gzip_vary on;
# gzip_proxied any;
# gzip_comp_level 6;
# gzip_buffers 16 8k;
# gzip_http_version 1.1;
# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

##
# Virtual Host Configs
##

include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
}


#mail {
# # See sample authentication script at:
# # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
# # auth_http localhost/auth.php;
# # pop3_capabilities "TOP" "USER";
# # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
# server {
# listen localhost:110;
# protocol pop3;
# proxy on;
# }
#
# server {
# listen localhost:143;
# protocol imap;
# proxy on;
# }
#}
```  

**nginx_app.conf**
``` 
server {
# 80번 포트로부터 request를 받는다
listen 80;

# 도메인명이 'localhost'인 경우에 해당
server_name localhost;

# 인코딩방식 지정
charset utf-8;

# reuqest/response의 최대 사이즈 지정 (기본값이 매우 작음)
client_max_body_size 128M;

# '/' (모든 URL로의 연결에 대해)
location / {
# uwsgi와의 연결에 unix소켓 (/tmp/app.sock 파일)을 사용한다
uwsgi_pass unix:///tmp/app.sock;
include uwsgi_params;
}
location /static/ {
alias /srv/project/.static/;
}
location /media/ {
alias /srv/project/.media/;
}
}
``

**superviosr.conf**  
```
[program:uwsgi]
command=uwsgi --ini /srv/project/.config/dev/uwsgi.ini #앞에 가상환경 pip어쩌고는 지워줌.

[program:nginx]
command=nginx

```
 
**uwsgi.ini**  
home = 가상환경 경로 필요없다.  
```  
[uwsgi]
;파이썬 프로젝트로 change directory
chdir = srv/project/app

;chdir로 바꾼 파이썬 프로젝트에서 wsgi모듈의 경로(path가 아닌 파이썬 모듈 경로)
module = config.wsgi.dev:application

;socket을 사용해 연결을 주고받음
socket = /tmp/app.sock

;uWSGI가 종료되면 자동으로 소켓파일을 삭제
vacuum = true

;log
logto = /var/log/uwsgi.log
```  

**seetigns/dev.py**  
local.py복사해온다.--> 에전에 한대로 dev, local 을 넣어준다.  
   
**wsgi 패키지화** 하고  dev, local 을 넣어준다.  
 

 
**Dockerfile.dev**  
```
FROM eb-docker:base
MAINTAINER nadcdc4@gmail.com


ENV PROJECT_DIR /srv/project
ENV BUILD_MODE dev

#nginx ,supervisor install
ENV DJANGO_SETTINGS_MODULE config.settings.${BUILD_MODE}

# dev용 requirements 설치
COPY ./requirements.txt /srv/
RUN pip install -r /srv/requirements.txt


# Copy projects files
COPY . ${PROJECT_DIR}
#WORKDIR ${PROJECT_DIR}



# Nginx 설정파일들 복사 및 enabled로 링크
RUN cp -f /srv/project/.config/${BUILD_MODE}/nginx.conf \
/etc/nginx/nginx.conf && \

# available에 nginx_app.conf파일 복사
cp -f /srv/project/.config/${BUILD_MODE}/nginx_app.conf \
/etc/nginx/sites-available/ && \

# 이미 sites-enabled에 있던 모든 내용 삭제
rm -f /etc/nginx/sites-enabled/* && \

# available에 있는 nginx_app.conf를 enabled로 링크.
ln -sf /etc/nginx/sites-available/nginx_app.conf \
/etc/nginx/sites-enabled/


# Supervisor 설정복사
RUN cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/supervisor.conf \
/etc/supervisor/conf.d

# RUN supervisor
CMD supervisord  
```   

`/build.py -m base`       
`/build.py -m dev`        
`docker run --rm -it -p 9994:80 eb-docker:dev` 해본다.  


## 6.production용 Dockerfile, settings, wsgi, .config작성  

production관련된 파일들도 도 만들자.     
 
**Dockerfile.production** ---> ENV BUILD_MODE production 이부분 정도만 Dockerfile.dev과 바뀐듯. /dev용requirements설치위한 부분 제거     
**.config/production,** --->./config/dev 내용 복붙 후 dev들어간부분 -->production으로 바꿔줌.         
**settings/prodiction.py** -->local.py, dev.py 와 같고 production만 붙여줬고, debug=False  Allowedhost에  localhost만 추가해줌.   
                               
**wsgi/production.py** -->production만 추가   
 
**build.py**에  -->production 관련된것 전부 추가.   
   
./build.py -m production         
docker run --rm -it -p 9994:80 eb-docker:production해본다.     
welcome 아니라 notfound 404 뜨는게 맞다.      
