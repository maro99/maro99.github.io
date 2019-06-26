---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기04"
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
date: "2018-09-07 18:50"  
---      

## 1.production에서 runserver실행 시 DEBUG 및 ALLOWED_HOSTS설정 분리  

**settings/production.py**    
```
ALLOWED_HOSTS =secrets['ALLOWED_HOSTS']   
```
이렇게 되어있는데 이것을   
runserver 로 실행했을때만 127.0.01 , localhost로   고정해놓고 싶다.     

settings/production.py 안에서   
`import sys`  
`print(sys.argv)`    

['./manage.py', 'runserver'] 이렇게 출력된다.    

argv란 ? 
>argv란 ?
argument vector: 가변적으로 들어올수 있는 인자. 
sys.argv 란 ?
파이썬 스크립트에 전달된 명령 행 인수 목록  

우리 같은 경우에는 manage.py runserver 이렇게 실행했으므로 위와 같이 인자들이 들어있다.   

우리의 경우   runsrever가 아니면 secert에서 가져다 Allowedhost지정하고 
아니면 기본값을 쓰자.    

`settings/production.py`  
```
RUNSERVER = sys.argv[1] == 'runserver'
DEBUG = False
ALLOWED_HOSTS = secrets['ALLOWED_HOSTS']
if RUNSERVER:
DEBUG = TrueDDd
ALLOWED_HOSTS = [
'localhost',
'127.0.0.1',
```  

local,dev,procution에서 각각 유저 이미지 넣고 잘 들어갔나 확인해보자.  
  
 `export DJANGO_SETTINGS_MODULE=config.settings.production`  
`./manage.py runserver` 한다.   
  
`export DJANGO_SETTINGS_MODULE=config.settings.production`  
`manage.py createsuperuser`
사용자 이름: maro3   

localhost:8000/admin 들어가본다.   
사진추가    
s3 버킷 페이지에서 잘 들어갔는지 확인한다.   

dev 에서도   
사진추가   
s3 버킷 페이지에서 잘 들어갔는지 확인한다.    


rds내용도 확인해보자.     
production   
```
(eb-docker-deploy-8Vvp4IdC) ➜  eb-docker-deploy git:(master) ✗ psql --host=eb-docker.c0d.ap-northeast-2.rds.amazonaws.com --user=maro --port=5432 eb_deploy_rds_pro
Password for user maro:
psql (10.5 (Ubuntu 10.5-0ubuntu0.18.04), server 10.4)
SSL connection (protocol: TLSv1.2, cipher: ECDHEM-SHA384, bits: 256, compression: off)
Type "help" for help.

eb_deploy_rds_pro=> select id, img_profile from members_user;
id |                 img_profile                  
----+----------------------------------------------
  1 | user/Screenshot_from_2018-08-18_13-39-36.png
(1 row)
```   
dev에서 같은 방법으로 해보면    
```
eb_deploy_rds_dev=> select id, img_profile from members_user;
id |                 img_profile                  
----+----------------------------------------------
  1 | user/Screenshot_from_2018-09-27_17-53-09.png
(1 row)
```  
디비가 각각 분리된것 이렇게 확인했다.   

## 2.deploy.sh작성, production모드에서 runserver판단기준 수정  

**deploy.sh**   
``` 
#!/usr/bin/env bash


# requirements만들기
pipenv lock --requirements > requirements.txt

# .secrets와 requriements를 starging area에 추가.
git add -f .secrets/ requirements.txt

# eb deploy 실행.
eb deploy --profile fc-8th-eb --staged

# .secrets와 requirements를 staging area에서 제거
git reset HEAD requirements.txt, .secrets

# requriements.txt 삭제
rm -f requirements.txt
```   

 correct static을 미리 하거나 or 서버에 docker올라간 뒤 correct static할 수 있는데 지금 같은 경우에는 서버에 올라간 뒤에 cellectstatic 하도록 하자.   
 
 `chmod 755 deploy.sh`
`./deploy.sh`

`eb open`    

애러뜬다.--------->Internal Server Error    

`eb ssh` 로 ec2안에 들어가서 찾아보자.   
`sudo docker exec -it $(sudo docker ps -q) /bin/bash`  
( sudo docker exex -it  +   sudo docker -q (컨테이너   아이디만 나옴 ) )     

uwsgi 의 log를 읽자.   
cd /var/log   
cat uwsgi.log   
```
RUNSERVER = sys.argv[1] = 'runserver'  
IndexError: list index out of range  
```
여기서 인덱스 에러가 났다.    

production모드에서 runserver판단기준을  수정하겠다.   
**app/config/settings/production.py**  
```
# Django가 runserver로 켜졌는지 확인
RUNSERVER = 'runserver' in sys.argv
```
커밋하고
`./deploy.sh`
`eb open`    


---



## 3.RDS 보안그룹에서 EB Envs를 Inbound에 추가.    

eb open을 하니 이번에는 이게 뜬다.  
```
Not Found
The requested URL / was not found on this server.
```
       
admin 페이지는 현제 url설정되서 가능. 로그인 해본다. 
이번에는 이게뜬다.        
```
504 게이트웨이 타임아웃..
```   
admin페이지의 staitc도 안나옴.
---->rds아이피가 ec2에 있는것 허용 안해서.  


ec2에 ip추가해야한다.   
하지만 문제는 eb는   몇개의 ec2 가질지 모르니까   
중간에 있는 load balancer에서 설정해줘야한다.  


AWS 홈페이지 rds security-group 가본다     
보안그룹 ->인바운드 에서 허용하고있는  ip를 봐라.     
-->현제 fast-campus ip만 보이는 상태.    
504 gateway timeout을 보내주는 서버는 어디에 있는가?   ---> aws 에 있는서버. 그쪽 ip는 우리 local과 다르다.   
     
eb deploy에서 연것이기 때문에     
http://maro.ap-northeast-2.elasticbeanstalk.com 가 존제하는 어딘가 이다.        
  
이것이 ec2 로 특정이 되는것이 아니라 로드벨런스 상에서 특정되기 때문에      
여기서 security-group filter없음을 본다. 두개의 보안그룹 더 나온다.    

```
Fastcampus8th-EBDockerDeploy-review   
sg-00d54811
awseb-e-vfmpxm9qx7-stack-AWSEBSecurityGroup-2H75HXYLMVG9
vpc-3af7fc52
SecurityGroup for ElasticBeanstalk environment.
 

 Fastcampus8th-EBDockerDeploy-review
sg-064b6b03
awseb-e-vfmpxm9qx7-stack-AWSEBLoadBalancerSecurityGroup-L38SK37PRB5
vpc-3af7fc52
Elastic Beanstalk created security group used when no ELB security groups are specified during ELB creation
```   
awseb-e-vfmpxm9qx7-stack-AWSEBSecurityGroup, (지금 우리에게 중요한건 이것. eb의 환경에서 쓰는 security group)   
awseb-e-vfmpxm9qx7-stack-AWSEBLoadBalancerSecurityGroup (로드벨런서의 security group) .....   
이렇게 두가지 보인다.   
   
-AWSEBSecurityGroup 이 그룹 자체가 eb로 만들어진 모든 환경이 같이 사용하는 보안그룹이다.         
( eb를 이용해서 만들어진 환경들 -<ec2에서 작동한다 그 ec2들이 전부 -AWSEBSecurityGroup 에 포함되어 있다.>  에서 rds로 접속할 수 있어야 한다.   
이 보안그룹 자체를 Rds 보안그룹에 추가해야한다. ( rds에서  eb로 만들어진 환경들 ~ ec2에서 보낸 인바운드를 받을 수 있도록 추가해야 한다.)     
해당하는 그룹 ID 를 sg-00d548114을    
Rds security-group의 편집 에 postgressql  선택  > sg-00d548114 > 입력하면 나오는데 그것 클릭.> 저장.  

다시 접속해서 로그인 해봄.   
http://fastcampus8th-ebdockerdeploy-maro.ap-northeast-2.elasticbeanstalk.com/admin/login/?next=/admin/

이번에는 로그인 잘 된다 .( css 적용안된상태. debug=False여서)   

여기서 locin이 안된다면 rds 보안그룹에 추가를 안해서 이다.       
SecurityGroup for ElasticBeanstalk environment. 인 보안그룹의 아이디를 복사해서  RDS Security Group eb-deploy 의 인바운드에 추가.  
편집, 포트, 5432 소스 에 넣는다.      

---   

## 4.eb ssh -> docker안에서 collect static해줌.    

runserver 할때는 이것(static) 이 db에 있는지 없는지 런서버 하는 순간에 검사한다.    
그런데 uwsgi, nginx로 연결할때는 실제에 db 연결하기 전까지는 db에 연결 안한다.(static파일 서빙 안한다는 뜻인듯)   
db에 연결하지 않고 보여줄수 있는 페이지는 보여주되 
db에 연결을 시도하는 순간 애러가 나는것.   

static적용안된 이 admin 페이지 에서 
css파일 개발자 도구로 열어보면 404 Notfound 뜬다.   
---->**collectstatic이 안되어 있기 때문에**      

collectstatic 한번 실행 해줘 보자. 

`sudo docker exec -it $(sudo docker ps -q) /bin/bash
cd /srv/project/app`    
`python manage.py collectstatic`   
   
그뒤 다시 eb open해서 해당 페이지 들어가니 static파일 잘 보인다.        


## 5.collectstatic 배포시마다 자동화 .(플랫폼 후크 사용)   
도커에 들어가서. collectstatic 자동으로 해주면 좋겠다. 
collectstaic 해주는 거도 배포가 끝나고  ec2에 있는 내용가지고 collectstatic 자동화 하고싶다.    

일단 우리가 아는건 서버에 들어가서 도커 들어가고 collectstatic실행하면 된다는것을 안다. 
이걸 우리가 매번 해줄 수는 없다. 

deploy 스크립트에 넣울 수도 있지만 
eb 자체에서 배포가 끝났을때 collectstatic같은걸 처리하게 할 수 있다. (서버에서 실행하는것.)   

eb 내부의 설정을 써야지 서버에서 collectstatic가능하다. 
[elasticbeanstalk문서](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/GettingStarted.html) 찾아보자.    

```  
고급 구성 문서.
   구성 옵션
         
         Elastic Beanstalk에서는 환경의 동작과 환경에 포함된 리소스를 구성하는 데 사용할 수 있는 많은 구성 옵션을 정의합니다. 
         (배포 환경을 구성할때 어떤 시점에 ~을 하라는것을 eb자체에 설정으로 남겨놓을 수 있다. 

구성 옵션은 환경의 Auto Scaling 그룹에 대한 옵션을 정의하는 aws:autoscaling:asg와 같은 네임스페이스로 구성됩니다.
Elastic Beanstalk 콘솔 및 EB CLI는 사용자가 환경을 생성할 때 사용자가 명시적으로 설정한 옵션을 비롯한 구성 옵션과 클라이언트가 정의한 권장 값을 설정합니다. 또한 저장된 구성 및 구성 파일에서 구성 옵션을 설정할 수도 있습니다. 여러 위치에 동일한 옵션이 설정되어 있는 경우 사용되는 값은 우선 순위에 따라 결정됩니다.
구성 옵션 설정은 텍스트 형식으로 구성하고 환경을 생성하기 전에 저장할 수 있으며, 지원되는 모든 클라이언트를 사용하여 환경을 생성하는 동안 적용할 수 있고, 환경을 생성한 후 추가, 수정 또는 제거할 수 있습니다. 이러한 각 세 단계에서 구성 옵션 작업에 사용할 수 있는 모든 방법에 대한 세부 분석은 다음 주제를 참조하십시오.
환경 생성 이전에 구성 옵션 설정 
환경이 생성되는 동안 구성 옵션 설정
환경이 생성된 후 구성 옵션 설정  **(collectstatic은 이 시점에서 해야할 것.)**       
```
[환경이 생성된 후 구성 옵션 설정
](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/environment-configuration-methods-after.html) 열어봤다.    

```
저장된 구성을 적용하거나, 구성 파일과 함께 새 소스 번들(.ebextensions)을 업로드하거나, JSON 문서를 사용하여 실행 중인 환경에서 옵션 설정을 수정할 수 있습니다. EB CLI 및 Elastic Beanstalk 콘솔에는 클라이언트별로 구성 옵션을 설정하고 업데이트하는 기능도 있습니다.
....
...

구성 파일(.ebextensions) 사용
.ebextensions의 프로젝트 폴더에 .config 파일을 넣어 애플리케이션 코드와 함께 배포
   웹 애플리케이션의 소스 코드에 AWS Elastic Beanstalk 구성 파일(.ebextensions)을 추가하여 환경을 구성하고 환경에 있는 AWS 리소스를 사용자 지     정할 수 있습니다. 구성 파일은 .config 파일 확장명을 사용하는 YAML이나 JSON 형식 문서로, .ebextensions 폴더에 놓고 애플리케이션 원본 번들로 배포합니다.
.....
구성 파일(.ebextensions) 사용
```  
다시 [구성 파일(.ebextensions) 사용](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/environment-configuration-methods-after.html#configuration-options-after-ebcli-ebextensions) 여기 들어가서  linux서버 항목을 참고했다.    

이중 키 항목     
```
패키지  
             eb ssh하면 어딘가에 연결됨. 
             이것도 리눅스. 엘라스틱 빈스톡으로 자동으로 만든 .ec2안에 리눅스. 센트오에스 기반 아마존 linux . 
             apt(debian 계열 페키지 관리자)  없다.  yum(contos 계열 페키지 관리자 ) 쓴다.  여기서도 vim 깔 수 있다. 

             packages 키를 사용하여 사전 패키지된 애플리케이션 및 구성 요소를 다운로드하고 설치할 수 있습니다.
            (우리는 도커 쓰니까 여기는 최대한 안건들겠슴. 여기 건들여서 설치하고 어쩌는것도 배포 시간 잡아먹어서... ) 

파일
              files 키를 사용하여 EC2 인스턴스에서 파일을 생성할 수 있습니다. 
         
명령
              커맨드 안에서 명령 남겨놓을 수 있다. (ec2안에서 어떤 명령 실행하도록~~)
              우리의 경우 배포 다한뒤에  ~ collectstatic하도록~남겨놓으면 된다.  

컨테이너 명령
            container_commands 키를 사용하여 애플리케이션 소스 코드에 영향을 주는 명령을 실행할 수 있습니다. 
            컨테이너 명령은 애플리케이션과 웹 서버를 설정하고 애플리케이션 버전 아카이브의 압축을 푼 후 애플리케이션 버전을 배포하기 이전에 실행됩니다.
            비컨테이너 명령과 기타 사용자 지정 작업은 추출하려는 애플리케이션 소스 코드보다 먼저 수행됩니다.

            배포 이전에 실행됨. ( 위의 명령과는 조금 다르다.

```        

eb ssh로 ec2들어가서    
명령어 한줄쓰면    
돌아가는 docker container 안애서 collectstatic    실행되도록 하고싶다. 이걸 명령어 한줄로 하고싶다.   

```
sudo docker exec -it 2142 /bin/bash
/srv/project/app# python manage.py collectstatic
sudo docker exec -it $(sudo docker ps -q) /bin/bash
```  
이걸 한줄로 바꾸면!  

`"sudo docker exec $(sudo docker ps -q) python /srv/project/app/manage.py collectstatic --noinput"`   


**.ebextensions/01_commands.config**  을 작성 
```
container_commands:
01_collectstatic:
command: "sudo docker exec $(sudo docker ps -q) python /srv/project/app/manage.py collectstatic --noinput
```     

add ~ commit 후 deploy  -> admin들어가 봤다.   

스테틱 css파일들이 안나온다.....    

진짜 없는지 확인해본다.   
.static없다.   
command    복사해서 붙여넣기를 해버리면 안된다.   
   
우리 의도돼로 실행됬나 혹인은 로그로 확인가능하다.   
01_static 이부분... 응 실행됬는데?   

이부분 믄제 있는것 같다.      
명령어가 실행된것이 컨테이너 실행되기 전이여서.   
컨테이너 여러개이니까     
명령 실행후 새 컨테이너가 만들어 진것.   

이 명령어를 컨테이너 생성 후로 미뤄야 한다.       



여기서의 문제는 새로 에플리케이션이 배포가 되기전에 container_commands(컨테이너 명령은) 실행되기 때문에 
이것이 문제이다. ... 배포 이후에 실행되는 옵션이 있으면 좋겠지만 그런건 없다.     


aws설명서 로 돌아와서 새로운것 해보자.  
```
AWS 설명서 » AWS Elastic Beanstalk » 개발자 안내서 » Elastic Beanstalk 플랫폼 » 사용자 지정 플랫폼 » 플랫폼 후크    이 경로로 들어갔다.     
```     
[플랫폼 후크](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/custom-platform-hooks.html)   
```
Elastic Beanstalk는 사용자 지정 플랫폼을 지원합니다. 사용자 지정 플랫폼은 몇 가지 측면에서 사용자 지정 이미지보다 더 세부적인 사용자 지정을 제공합니다. 사용자 지정 플랫폼을 사용할 경우 플랫폼을 완전히 새로 개발하여 Elastic Beanstalk가 플랫폼 인스턴스에서 실행하는 운영 체제, 추가 소프트웨어 및 스크립트를 사용자 지정할 수 있습니다.    

플랫폼 후크(어떤 시점 잡는다는뜻)
Elastic Beanstalk는 후크에 표준화된 디렉터리 구조를 사용합니다. 후크는 수명 주기 이벤트 중, 그리고 관리 작업(환경의 인스턴스가 시작될 때 또는 사용자가 배포를 초기화하거나 애플리케이션 서버 재시작 기능을 사용할 때)에 응답할 때 실행되는 스크립트입니다.
후크는 다음 폴더에 정리되어 있습니다.  

         appdeploy – 애플리케이션을 배포할 때 실행되는 스크립트입니다. 새 인스턴스가 시작될 때와 클라이언트에서 새 배포 버전을 초기화했을 때 Elastic Beanstalk가 애플리케이션 배포를 수행합니다.
         
        configdeploy – 인스턴스에서 소프트웨어 구성에 영향을 미치는 업데이트(예: 환경 속성 설정 또는 Amazon S3 로그 순환 활성화)를 클라이언트에서 수행하면 실행되는 스크립트입니다.(업데이트 후 실행 )
        
        restartappserver – 클라이언트에서 앱 서버 작업 재시작을 수행하면 실행되는 스크립트입니다.
        
        preinit – 인스턴스 부트스트래핑 중 실행되는 스크립트입니다.(인스턴스 실행 전)
        
        postinit – 인스턴스 부트스트래핑 후 실행되는 스크립트입니다.(인스턴스 실행 및 초기화 후)
        
        appdeploy, configdeploy 및 restartappserver 폴더에는 pre, enact 및 post 하위 폴더가 들어 있습니다. 작업의 각 단계에서 pre 폴더의 모든 스크립트가 알파벳 순으로 실행되고 이어서 enact 폴더, post 폴더 순으로 실행됩니다.

인스턴스가 시작되면 Elastic Beanstalk가 preinit, appdeploy, postinit를 순서대로 실행합니다. 인스턴스 실행에 이은 후속 배포에서 Elastic Beanstalk는 appdeploy 후크를 실행합니다. configdeploy 후크는 사용자가 인스턴스 소프트웨어 구성 설정을 업데이트하면 실행됩니다. restartappserver 후크는 사용자가 애플리케이션 서버 재시작을 초기화했을 때만 실행됩니다.
오류 발생 시 스크립트는 0이 아닌 상태로 종료한 후 stderr에 기록해 작업을 장애 조치할 수 있습니다. stderr에 기록한 메시지는 작업이 실패했을 때 출력되는 이벤트에서 나타납니다. Elastic Beanstalk는 또한 이 정보를 로그 파일 /var/log/eb-activity.log에 캡처합니다. 작업을 장애 조치하지 않으려면 0을 반환하십시오. stderr 또는 stdout에 기록한 메시지가 배포 로그에 표시되며 작업이 실패했을 때만 이벤트 스트림에 나타납니다.
```     


appdeploy여기 넣어줘야하는데 이게 또 안에서 갈린다. 


일단 여기 한번 가보자. 

`eb ssh`
`cd  /opt/elasticbeanstalk/hooks/appdeploy/post`   
  
  
가보면 이미 두개 script있다.    

```
rwxr-xr-x 1 root root 1386  9월 19 19:40 00_clean_imgs.sh     ( 도커 배포위해서 이미지 가져온것을 배포 끝난뒤 지우는것.)   
-rwxr-xr-x 1 root root  397  9월 19 19:40 01_monitor_pids.sh ( 뭔가 모니터링하는 툴을 실행)  
```   

여기에 sh 집어넣으면 여기의 파일들 처럼 deploy끝난뒤 실행된다. (순서는 여기 파일 명 대로)   
   
배포가 됬을때 여기에 파일이 있었으면 좋겠다.   
(여기는 docker밖이니까.) 인스턴스안에 파일 만들어야 한다.( Linux 서버에서 소프트웨어 사용자 지정. 에서 본 파일 키를 사용해서 만들자. )  
  
그 파일에 어떤 명령 실행하도록 해야한다.    

일단 이렇게 주석처리 하고 
**01_commands.config**  
```
#container_commands:
# 01_collectstatic:
# command: "sudo docker exec $(sudo docker ps -q) python /srv/project/app/manage.py collectstatic --noinput"
```   
  
**.ebextensions/01_files.config** 만듬  
```
files:
"/opt/elasticbeanstalk/hooks/appdeploy/post/01_collectstatic.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py collectstatic --noinput
```    

배포 다시 해봄.    
  
`git add -A`    
`./deploy.sh`    
`eb open`      

이번에는 admin 의 static파일들 잘 보인다.       












