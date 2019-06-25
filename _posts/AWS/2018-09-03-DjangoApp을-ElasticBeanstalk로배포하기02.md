---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기02"
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
date: "2018-09-03 18:50"  
---  


## 1.EB CLI(Elastic Beanstalk 명령줄 인터페이스) 구성  

코드를 업로드하기만 하면 Elastic Beanstalk가 용량 프로비저닝, 로드 밸런싱, 자동 크기 조정부터 시작하여 애플리케이션 상태 모니터링에 이르기까지 배포를 자동으로 처리한다. 이뿐만 아니라 애플리케이션을 실행하는 데 필요한 AWS 리소스를 완벽하게 제어할 수 있으며 언제든지 기본 리소스에 액세스할 수 있다.        

[EB CLI를 설치](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/eb-cli3-install.html)한 후 eb init를 실행하면 프로젝트 디렉터리 및 EB CLI를 구성할 준비가 완료된다.   

다음은 eb init한 후 터미널의 모습이다.  
```
(eb-docker-deploy-fpWdLAkh) ➜ eb-docker-deploy git:(master) eb init --profile fc-8th-eb
 
Select a default region
8) ap-southeast-2 : Asia Pacific (Sydney)
9) ap-northeast-1 : Asia Pacific (Tokyo)
10) ap-northeast-2 : Asia Pacific (Seoul)

(default is 3): 10
 
Enter Application Name
(default is "eb-docker-deploy"): EB Docker Deploy
Application EB Docker Deploy has been created.
 
It appears you are using Python. Is this correct?
(Y/n): n
 
Select a platform.
7) Docker
(default is 1): 7
 
Select a platform version.
1) Docker 18.03.1-ce
2) Docker 17.12.1-ce

(default is 1): 1
Note: Elastic Beanstalk now supports AWS CodeCommit; a fully-managed source control service. To learn more, see Docs: https://aws.amazon.com/codecommit/
Do you wish to continue with CodeCommit? (y/N) (default is n): n
Do you want to set up SSH for your instances?
(Y/n):
 
Select a keypair.
1) fc-8th
2) [ Create new KeyPair ]
(default is 1): 1
```  

이제 elastic beanstalk 초기화가 된것.   
. elasticbeanstalk 폴더 생김.-> <config.yml 가보면 아까 입력한 것 볼 수 있거 자동으로 .gitignore 에 추가됬다.-->잃어버리지 말고 잘 관리하면된다.  


eb  init시 다음과 같은 오류 뜨면    
```
ERROR: UndefinedModelAttributeError - "serviceId" not defined in the metadata of the model: ServiceModel(elasticbeanstalk)
```
`pip install awsebcli --upgrad` 먼저 해준다.   
 
 ---  
 
 
 
## 2.Elastic Beanstalk 환경을생성  

[EB CLI를 설치](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/eb-cli3-install.html)하고 [프로젝트 디렉터리를 구성](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/eb-cli3-configuration.html)하면 EB CLI를 사용하여 Elastic Beanstalk 환경을 생성하고, 소스 및 구성 업데이트를 배포하고, 로그와 이벤트를 가져올 준비가 된다.  

AWS 의 Elasticbeanstalk텝에 접속해 보면    
어플리케이션 생김. 어플리케이션- 우리가 보통 말해온 project 하나라 생각하면 된다.  
우리가 만든 프로젝트가 반드시 하나의 서버에서만 돌아가지 않는다. 서버를 두개 틀 수 있다.  
( 예를 들어 깃으로 관리시 브랜치 dev, master, topic 브랜치별로 나눠서 관리했다. 비슷하게 브랜치별로 사용할 수 있도록 쉽게 연동이 되어있다. 서버를 두개틀 수 있는것. 운영서버 틀어놓고 개발서버 틀어놓으면각각을 환경이라 한다. 개발환경, 운영환경.)   
하나의 엡 안에 환경 여러개 있을 수 있고 각환경마다 계속 코드를 업데이트해서 새로운 배포를 실행해 볼 수 있다.   
dev 서버에서 뭔가 해봐서 완벽하면 그글 master환경에 올리는 등. 온라인에 서버를 놓되 분리를 하는것.   
환경하나에 ec2하나 과금.--->우리는 한개로 작업을 하겠다... 그 내용이 elasticbeanstalk /config.yml에 나와있다.

EB CLI로 환경을 생성하려면 서비스 역할이 필요하다. Elastic Beanstalk 콘솔에서 환경을 만들어 서비스 역할을 만들 수 있다.  
서비스 역할이 없는 경우 사용자가 eb create를 실행할 때 EB CLI가 역할 생성을 시도한다.  
첫 번째 환경을 생성하려면 eb create를 실행하고 프롬프트를 따르면된다.  
프로젝트 디렉터리 안에 소스 코드가 있는 경우 EB CLI는 이를 번들링한 후 환경에 배포한다.  
그렇지 않은 경우 샘플 애플리케이션이 사용된다.   

```

(eb-docker-deploy-fpWdLAkh) ➜ eb-docker-deploy git:(master) ✗ eb create --profile fc-8th-eb
Enter Environment Name
(default is EBDockerDeploy-dev): Fastcampus8th-EBDockerDeploy
Enter DNS CNAME prefix
(default is Fastcampus8th-EBDockerDeploy): -maro (주소만들때 이내용쓰니까 겹치면 안된다.)
CNAME must be 4 to 63 characters in length. It can only contain letters, numbers, and hyphens. It can not start or end with a hyphen
Enter DNS CNAME prefix
(default is Fastcampus8th-EBDockerDeploy): Fastcampus8th-EBDockerDeploy-maro
Select a load balancer type
1) classic
2) application
3) network
(default is 1): 2
```  

CNAME: Fastcampus8th-EBDockerDeploy-maro.ap-northeast-2.elasticbeanstalk.com 이 url로 이제 접속 가능하게됨. 여기에 배됨.   

 뭔가 많이 세팅하고 있다.   
 ec2도 만들고,  시큐리티 그룹 지알아서 만들고.  
이상태가 배포중인 상태. 소스코드 압축한후 s3에 올린다.  s3에 올린 코드를 서버 세팅이 끝난 후 그 코드 받아와서 배포 완료를 함.  
당연히 이대로는 안되고 추가 설정이 필요하다.  

환경을 만드는데 성공하지만 에러가 뜰것이다.  
이유는 도커 어플리케이션 실행에 필요한 파일이 없기때문.  

-->해결법 
도커 파일을 만들어야함.   
베포를 위해서는 Dockerfile.production을 실행하면 된다는 것을 알 고 있다.    
이것은 FROM      eb-docker:base- > 에서 시작된다.    
기존의 eb-docker:base 이미지는 웹에없고 내컴퓨터에있다.
이상태서 배포 했을때 그안에서 docekrfile   (Dockerfile.production내용으로 )실행하면 실행안될것.   


---  


## 3.Docker 컨테이너에서 Elastic Beanstalk 에플리케이션 배포.(Dockerfile작성.)  

[aws 에서 docker로작업](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/create_deploy_docker.html) 문서를 보고 해보겠다.   
>Elastic Beanstalk는 Docker 컨테이너에서 웹 애플리케이션의 배포를 지원합니다. Docker 컨테이너로 실행 시간 환경을 사용자 지정할 수 있습니다. 다른 플랫폼에서 지원되지 않는 기타 애플리케이션 종속성(패키지 관리자 또는 도구 등), 프로그래밍 언어, 사용자 지정 플랫폼을 선택할 수 있습니다. Docker 컨테이너는 독립형으로 실행되며 웹 애플리케이션을 실행하는 데 필요한 소프트웨어와 모든 구성 정보를 포함합니다.    


[단일컨테이너도커](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/single-container-docker.html)  
>단일 컨테이너 Docker 환경은 Dockerfile(빌드할 이미지 설명), Dockerrun.aws.json 파일(사용할 이미지와 추가 Elastic Beanstalk 구성 옵션 지정) 또는 둘 다에서 시작할 수 있습니다.(둘중하나만 있어도 되고 둘 다 있어도된다.**Dockerfile = 도커 이미지를 구성하는 파일. aws에서 빌드요청 했을때 알아서 dockerfile의 내용 실행해서 배포를 완료해준다. 그내용을 우리가 이제 정해야 한다. )**  이러한 구성 파일은 소스 코드로 번들링하여 ZIP 파일로 배포할 수 있습니다.     


Dockerfile에   
기존의 Dockerfile.base 내용 복사   
Dockerfile.production. MAINTAINER 이하 냬용 복사해서 붙여 넣기함.     

**Dockerfile**  
```
# slim- 최소한 필요한것만 깔림.
FROM python:3.6.5-slim
MAINTAINER nadcdc4@gmail.com

# uWSGI는 Pipfile에 기록
RUN apt -y update && apt -y dist-upgrade
RUN apt -y install build-essential
RUN apt -y install nginx supervisor

# 로컬의 requirements.txt파일을 /srv에 복사후 pip install 실행
# (build하는 환경에 requirements.txt.가 있어야함!)
COPY ./requirements.txt /srv/
RUN pip install -r /srv/requirements.txt



#이하 production에서 복사.

ENV PROJECT_DIR /srv/project
ENV BUILD_MODE production

#nginx ,supervisor install
ENV DJANGO_SETTINGS_MODULE config.settings.${BUILD_MODE}


# Copy projects files
COPY . ${PROJECT_DIR}
#WORKDIR ${PROJECT_DIR}


# Ngnix config
RUN cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/nginx.conf \
/etc/nginx/nginx.conf && \

# available에 nginx_app.conf파일 복사
cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/nginx_app.conf \
/etc/nginx/sites-available/ && \

# 이미 sites-enabled에 있던 모든 내용 삭제
rm -f /etc/nginx/sites-enabled/* && \

# available에 있는 nginx_app.conf를 enabled로 링크.
ln -sf /etc/nginx/sites-available/nginx_app.conf \
/etc/nginx/sites-enabled

# Supervisor 설정복사
RUN cp -f ${PROJECT_DIR}/.config/${BUILD_MODE}/supervisor.conf \
/etc/supervisor/conf.d

# RUN supervisor
CMD supervisord -n
```  


**/requirements.txt** 를 만들어 주고 배포 해보자.   
(elasticbeanstalk 사용할때 말고 기존의 로컬+docker에서는 build.py에서 알아서 다해줬었음.) 

`pipenv lock --requirements > requirements.txt`  

다시 배포해본다.   
`eb deploy --profile fc-8th-eb`  

분명히 우리 프로젝트에 에 도커파일 있는데 없다고 애러 뜬다.   
```
ERROR: Dockerfile and Dockerrun.aws.json are both missing, abort deployment
```  

이유를 살펴보면    
**elasticbeanstalk/config.yml**가보면  
```
 branch-defaults:
master:
environment: Fastcampus8th-EBDockerDeploy
group_suffix: null
```  
이 environment가 aws사이트에서 elasticbenastalk 모든 어플리케이션에 보인다.   
환경이 하나 생긴것.   
이 environment가 branch-defaults의 master에 붙어있다는 것은 마스터 브랜치와 연동된다는것.  
eb-deploy하면 마스터 브랜치에 있는 내용만 올라간다.  
즉 깃에 포함된 내용만 올라가게되있다.  
커밋 안해서 도커파일 등 추가 안된상태.  ----> **배포위한  변경사항을 git에 add 먼저 해야한다.**  


---  



## 4. ebcli와 git사용.   
[eb cli와 git 사용 ](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/eb3-cli-git.html) 을 참고중이다.   

```
Git 브랜치와 Elastic Beanstalk 환경 연결

다양한 환경을 코드의 각 브랜치와 연결할 수 있습니다. 브랜치를 체크아웃하면 연결된 환경에 변경 사항이 배포됩니다. 예를 들어, 다음을 입력하여 프로덕션 환경을 마스터 브랜치와 연결하고 별도의 배포 환경을 배포 브랜치와 연결할 수 있습니다.  

기본적으로 EB CLI는 커밋 ID와 메시지를 각각 애플리케이션 버전 레이블과 설명으로 사용하여 현재브랜치에서 최신 커밋을 배포합니다. 커밋하지 않고 환경에 배포하려면 --staged 옵션을 사용하여 스테이징 영역에 추가된 변경 사항을 배포할 수 있습니다.
```   

commit해야할것은 commit하고, secret같이 하면 안되는 것들은 add 만 해주고 배포해보겠다.   
`git add requirements.txt`  
`git add -f .secrets`  
`eb deploy --profile fc-8th-eb --staged`  
  
 ---  
  
  
## 5.애러 수정: EXPOSE(Docker 컨테이너에 표시할 포트를 나열하는것) 추가     

EXPOSE   
```
(필수) Docker 컨테이너에 표시할 포트를 나열합니다. Elastic Beanstalk에서는 포트 값을 사용하여 호스트에서 실행 중인 역방향 프록시에 Docker 컨테이너를 연결합니다.
(도커를 실제로 ec2에서 쓸때 세팅을 해놓는것.)  

컨테이너 포트를 여러 개 지정할 수 있지만 Elastic Beanstalk에서는 첫 번째 포트만 사용해 호스트의 역방향 프록시에 컨테이너를 연결하고 퍼블릭 인터넷의 요청을 라우팅합니다. (뭔소리인지 아래서설명)
```  

eb로 배포한 ec2 -> django app까지 의 흐름이 다음과 같다. 
```
Browser -> ElasticBeanstalk
    Load Balancer
          ->EC2 -> Nginx(EB)
          (Reverse proxy) 80 -> 7000(Dockerfile EXPOSE)
         ->Docker(Container) :7000 -> Nginx (Docker Container):80-> uWSGI ->Django
```
위처럼 docker에서 7000번을 열어서 eb가 생성한 ec2바로 아래 있는 nginx의 80번 포트로 부터 오는 요청을 받겠다는것.   

**Dockerfile**에 다음을 추가   
```
# 7000번 포트 open
EXPOSE 7000
```     
`eb deploy --profile fc-8th-eb --staged`  
`eb open`       

 ----> 이번엔  502 bad gate 에러 가 맞이해준다.   
 
 
 
 ---  
 
 
 
 
 ## 7.애러수정: django프로젝트 앞단의 nginx 포트번호 불량.    
 
 **nginx_app.conf**     
 listen을 변경   
 ```
 listen 7000
 ```  
 다시 배포해 본다.   
`git add -A`
`git add -f .secrets`
`eb deploy --profile fc-8th-eb --staged`
`eb open`    

----> 이번엔 Bad Request(400) 애러    
500에러 아니므로서버 잘못한게 아니라 django app이 잘못한것이 있는거다.   


 




























 




