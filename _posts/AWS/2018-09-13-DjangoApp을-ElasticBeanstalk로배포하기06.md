---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기06"
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

## 1. EB와 Loadbalancer구조와 ec2 leader    
앞서서 leader옵션이 적용된 ec2하나에서만 createsuperuser가 생성되도록 설정해 주었다.     
우리가 만든 loadbalancer의 구조를 한번 살펴보자.    
```
EB  (Application)
        (Env1)
            Load Balancer
                    EC2 (Leader)
                    EC2
                    EC2

        (Env2)
            Load Balancer
                    EC2 (Leader)
                    EC2
                    EC2
```  

EC2 시피유 점유율이 90프로 넘으면 ec2를 로드벨런서가 늘린다.   
그중 한대가 리더가 된다. 어디가 될지는 모른다.   
eb(appication) 안에   
        env여러개 있고    
                그밑에 로드 벨런서가 있고 이것이ec2    인스턴스 여러개가 있다.    

지금 우리같은 경우에는   

**config.yml**보면  
```
branch-defaults:
master:
environment: Fastcampus8th-EBDockerDeploy-review
group_suffix: null  
```
기본 eb 설정이 master브랜치 기준으로 되어있다. 
마스터 브랜치가 Fastcampus8th-EBDockerDeploy-review 이 환경 가지고 있는데      
다른 브렌치가 또 다른 환경 만들면 거기에 loadbalancer생기고 거기에 ec2들이 붙는다.   

## 2. leader_ec2 구분 위해서 만든  collectstatic, migrate, createsu 등의 파일 지워줌   
leader_ec2,일반 ec2 구분 위해서 commands.config에서 만들어 줫던 tmp/ collectstatic, migrate, createsu 등의 파일 지워줘야한다.     
**commands.config**  
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
rm /tmp/collectstatic
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
rm /tmp/migrate
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py migrate --noinput
fi


"/opt/elasticbeanstalk/hooks/appdeploy/post/03_createsu.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
if [ -f /tmp/createsu ] # 파일이 있으면 이하를 실행.
then
rm /tmp/createsu
sudo docker exec `sudo docker ps -q` python /srv/project/app/manage.py createsu
fi
```   
rm라인 세개가 이번에  추가해준것이다.   

만약 배포중 오류가 뜬다면 파일 이름 변경한 것들이 eb안의 /opt/elasticbeanstalk/hooks/appedeploy/post/에 남아있기 때문일 것이다.      


이와 같이 /opt/elasticbeanstalk/hooks/appdeploy/post/01_collectstatic.sh 와같은 파일이 이미 남아있으면 오류를 불러 일으키는 경우  
대비해서 극단적으로 는 오토 스크립트 돌려서  기존환경 다 없에고 ebcreate를 하루에 한번 하기도 한다. (우리같은 경우는 설정한 환경에서 새로운 배포만 하는데...) 

환경이 이미 있으면 swap하는것도 가능하다.    

이것을 **cnameswap**혹은 domainname swaping이라 한다.

## 3.cnameswap이란 ?   
이번에는 구현하지않겠지만 설명을적고 넘어가겠다.   

```
EB (Application)
    Env1
        EC2 -> eb create, deploy
    
    Env2
        EC2
```  
다음과같이 환경 Env1 하나 만든상태에서    
eb deploy,create해서 만들어 지는 것이 ec2.   
이 Env1 안에서 ec2가 계속 유지되는 중이다.    

이때   
Env2 하나더 만들고 eb create등 다시하면 
새로운 EC2가 또 생성된다.   
그뒤 Env1에 연결되 있던 외부의 연결을 Env2로 전부다 옮기는것이다.    

[http://console.aws.amazon.com/](http://console.aws.amazon.com/)  접속해서  elasticbeanstlak 가보자.   

환경별 회색, 빨간색 박스 클릭 해본다..   
우리가 eb-create할때 c-name물어봤던것   
그 cname이 url 앞에있는 이부분이다.   
maro.ap-northeast-2.elasticbeanstalk.com    
여기서는 maro. 까지가 cname    
환경별로  cname을 따로 갖는다.   
새로운 환경 만들면 다른 도메인 가지고 접근 가능해지는데   
이 두개를 서로 바꿔주는것.   
하나는 아예 종료하고 기존에 환경에 오던 요청을 새로운 환경에 보내버리는 것.   
매일매일 이 과정을 자동으로 이뤄지도록.   

이렇게 할 경우 서버 안정성이 대폭 올라간다.   
(프로그램 같은 경우는 변수가 참조하는곳 없으면 메모리가 자동으로 해제되는데 
코드 중 가끔 외부 라이브러리 등경우 메모리 해제 안되서 메모리 조금씩 누수되서 사용못하는 메모리 늘어나는 경우 잇을 수 있다. 
하루에 한먼 배포한다면 그런 걱정을 할 필요가 없다. )   


## 4. script 만들어서 appdeploy 안에 파일들(커스텀 셀 스크립트) 배포할때마다 모두 한번 지우도록함   

먼저 eb ssh 해서    opt/elasticbeanstalk/hooks/appdeploy/post들어가서   
쓰래기 파일 몇개 만들어 놓고 다시 배포해새 서 삭제 되는지 보자.   
`toutch 04_asdasd.sh`
`sudo chmod 755 04_asdasd.sh`   

**.ebextensions/01_files.config**  에 수정
```
"/opt/elasticbeanstalk/hooks/appdeploy/post/maro_01_collectstatic.sh":...
"/opt/elasticbeanstalk/hooks/appdeploy/post/maro_02_migrate.sh":...
"/opt/elasticbeanstalk/hooks/appdeploy/post/maro_03_createsu.sh":...
"/opt/elasticbeanstalk/hooks/appdeploy/post/maro_9999_remove_scripts.sh":
mode: "000755"
owner: root
group: root
content: |
#!/usr/bin/env bash
rm -f /opt/elasticbeanstalk/hooks/appdeploy/post/maro*.sh
rm -f /opt/elasticbeanstalk/hooks/appdeploy/post/maro*.sh.bak
```    
아예 만들어주는 sh 파일 앞에 닉네임(maro) 붙여서 그거 가진것들만 지워주자.     

## 5. dockerhub에 올라간 base이미지와 deploy.sh에서 requriement저장하는부분 중복되서 deploy.sh에서 삭제함.   

deploy.sh 에 쓸때없이 requirements.txt 만들어주는데    
생각해보니 그런 과정 Dockerfile.base에 다 있는 내용이고.(우리는 도커헙에서 다 가져다쓰는중.)  
Dockerfile에서 추가적으로 requriements만들고 install해주는 과정이 없다 .  

**deploy.sh** 에서  아래 주석 표시한 부분을 제거해주자.
```
#!/usr/bin/env bash

# requirements 만들기
#pipenv lock --requirements > requirements.txt   여기 

# .secrets와 requirements를 staging area에 추가
git add -f .secrets/ requirements.txt

# eb deploy 실행
eb deploy --profile fc-8th-eb --staged

# .secrets와 requirements를 staging area에서 제거
git reset HEAD .secrets/ # requirements.txt  여기 

# requirements.txt 삭제
#rm requirements.txt  여기 
```     





