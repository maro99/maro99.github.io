---
layout: "post"
title: "DjangoApp을 ElasticBeanstalk로 배포하기07"
categories:  
- AWS  
tags:  
- Python    
- Docker      
- ElasticBeanstalk    
- deploy    
- route53
- ebextensions   
comments : true    
date: "2018-09-09 18:50"  
---      

## 1.Route53으로 Domain 붙여보자.   
route53이 하는일은 ?    
```
Amazon Route 53은 DNS(Domain Name System) 서비스입니다.
 DNS는 사람이 읽을 수 있는 도메인 이름 (example.com)을 IP 주소(192.0.2.0)로 변환하는 시스템입니다.
 전 세계 데이터 센터에 권위 있는 이름 서버를 갖춘 53번 국도는 안정적이고 확장 가능하며 빠릅니다.
(route 53 이용해서 굉장히 쉽게  aws에 도메인을 붙일 수 있다.)
```   

[AWS Route53](https://console.aws.amazon.com/route53) 에 들어가서      
create hosted zone 클릭해라. -> 또클릭.  
( 우리가 쓸 도메인 자체에 대한 설정 )  
  
오른쪽 보면 이렇게 적으라고 되있다.    
```
Domain Name:
Comment:
Type:
```  
Domain Name에 자신이 가진 도메인 네임 그대로 적는다.내경우 nssmr.com
나머지 두개는 안적고 create누른다.    

![1](https://imgur.com/bGvGjet)   


type ns에 value 4개 있는데    
이것을 자신이 도메인 산곳  https://www.hosting.kr 들어가서   
나의서비스 --> 도메인 관리   
nssmr.com 체크    
도메인 관리 --->네임서버주소 변경   
신청하기 누른다.   

![2](https://imgur.com/wNshXJq)     



1,2,3,4차 넣는부분 에다가   
aws의 nss -type 의 valie의 값들을 그대로 넣는다.   
호스팅 사이트의 ip check눌러준다. 하나씩   
aws상에서는 이제 건들것 없다.   
변경하기 누른다.    


네임서버란 도메인에 요청이 왔을때 그것을 어디로 보낼지 정해주는 서버. 
그 주소를 우리가 바꾼것이다.       
이것이 실제로 적용 되기위해 전세계 네임서버들은  이것이 바뀐다음에 여기저기 알려준다.      
그게 전세계로 퍼질때 까지 기다린다.    

우리 서비스에 이것을 연결시켜야 한다.     
Domain -> IP Address    보통은 도메인 주소란것은 ip에 붙는다. (어떤 도메인으로 왔을때 어떤 ip로 가느냐.)     

문제는 우리가 다음과 같이 eb를 쓴다는 것이다.   
```
EB
    Load Balancer
        EC2 ( IP)
        EC2 (IP)
```  

각각의 ec2들이 각각의 ip갖는다.     
그래서 도메인을 어디 붙일줄 모르니까 load balancer에 붙여준다.   
(load balancer가 어떤 특정한 ip갖는다고 하기는 조금 애매하다. 그런식의 동작과는 다르다.AWS에서는 어떤 도메인이 있으면 그걸 이 loadbalncer에 붙인다는 설정을 route53으로 할 수 있다.  )     


다시 AWS로 돌아와서 create Record Set  클릭한다.    
create hosted zone은 우리가 쓸 도메인 자체에 대한 설정.   
create Record Set 은 도메인으로 온 요청이 어디로 갈지에 대한 설정.     

![3](https://imgur.com/BcNh0jF)  

name에 아무것도 안붙이면 그냥 그 도메인 자체에 접근할때에 대한설정이다.  
**type**은 IPv4
**alias**는  아마존에서 제공하는 서비스에 도메인 붙일건지에 대한 설정. -->yes
**alias target*** 누르면    
    elb application loadbalancer와   
    elb enverionment 두개 뜬다.   

둘중 하나 아무나 연결해도 되지만   
**loadbalacner**에 하는것이 나중에 https설정으로 인증서 붙일때 쉽다.   

create하면 왼쪽에 이렇게 A type의 새로운 칸이 생겼다.  
![4](https://imgur.com/E7aXLHO)  


type들이 여러개 있는데    
A type이 기본적인 ipv4  아이피 주소 방식이다.(바로 ip로 연결)    
근데 alias target으로 설정한 dualstack.awseb-awseb-1wp%%%%kryu9-13%%%%449.ap-northeast-2.elb.amazonaws.com. 는 ipv4가 아닌데 ? 
이것을  aws안에서 alias로 자동으로연결 해준다.    
만약에 dns시스템을 aws말고 다른회사 것 을 쓴다면 ?    
이런식으로 loadbalacner에 붙이는것은 안된다.     

IPV4말고 다른  type살펴보면 이런것들이 있다.     
```
CNAME   ---> 다른 도메인으로 연결하는것. (이것 같은 경우는  도메인 -->도메인 -->ip 이렇게 두단계 거쳐가는것.) 
MX  -- > 메일 프로토콜 관련 
AAAA ---> ipv6관련 
```  

아까 create recordset다시 가서    
www. 을 name으로 붙인거 하나더 생성 하겠다.  나머지는 아까와 같게 해준다.     

## 2. HTTPS인증서 연결    
일단 https인증서가 뭔지 살펴보자.   
구글 접속시 자물쇠 표시 클릭 -- >인증서 보기 
globlsigin 이라해서 root인증   기관에서 받은인증서 유효하다고 뜬다.  

이 내용을 증명하는과정에는 공개키 기법을 사용된다.   
인증받은 기관 서버에 공개키를 넣어놓고 
우리 개인키로 접속 여부를 확인 
이것을 인터넷 의 컴퓨터 끼리 주고 받는것이다.        

[lets encrpyt](https://letsencrypt.org/) 이라는 무료 서비스도 있지만 aws에서는 편하게 자체 서비스를 이용한다.
**aws console  ----> acm**    
(사이트의 이름을 제공하고 ID를 만들면 나머지는 ACM이 처리합니다.    
ACM은 Amazon 또는 사설 인증 기관에서 발행한 SSL/TLS 인증서 갱신을 관리합니다.)   
인증서 프로비저닝 --->시작하기   
인증서 요청    
도메인이름 추가  (아까 www.  www없는것 두개다 적어줘야한다.)

DNS 검증          ----------> 우리가 어떤 도메인에 대해 인증서 요청했으니 그것의 주인이 나라는것을 인증해야한다. 
이메일 검증       -------------> host kr가입시 입력한 메일로 인증메일이 온다. ---->확인및요청 ---> 메일가서 확인 해준다. 

다시 aws certificate manager돌아오면 발급 완료라고 뜰 것이다.  

이 도메인 주인이라는것이 인증 된것이고 
이제 ssl 인증서를 받을 수 있다. (SSL 이라는건 공계키 암호  기법을 http에 붙였다 생각하면 된다.)

nssmr.com에 접속해 보면    
Bad Request (400) 뜬다.    
   
연결은 됬는데 nginx설정에서 못밭는 것일 수도 있다.    

배포코드를 를 다시한번 해보자.      

allowed host에 nssmr.com에 대한 내용 없으니 다음과 같이 secret에 추가해준다.   

**.secret/prodiction. json**
```
{
"ALLOWED_HOSTS" : [
".elasticbeanstalk.com",
"nssmr.com",
"www.nssmr.com"
],
```   
또한 nginx설정도 바꿔준다.   
**.config/nginx_app.conf**      
```
server_name localhost; -> server_name *.elasticbeanstalk.com www.nssmr.com nssmr.com; 이렇게 변경     
```   

다시 배포한다.    

이제 www.nssmr.com/admin접근하면 잘 된다.        
하지만 위에 안전하지 않음 이라는 웹사이트가 뜬다. (Not secure)    

aws ec2 console의 로드벨런서  가본다. 
eb에서 만든  로드벨런서 한개가 존재하는 상태 
아래 리스너 있다.       
![5](https://imgur.com/6Z0RZ4F)        


로드벨런서가    
외부로부터 받는 요청이 알아서 필터링되게 되있다.   
리스너가 기본적으로 HTTP80이다.   
80으로 받은걸 오른쪽 eb로 전달하고 있다.   
   
우리가 받아야할 포트가 https쓰니까 443포트도    추가해야한다.    
리스너 추가 누른다.  
   
프로토콜 : 포트   
HTTPS : 443    

전달대상    
awseb -AWSEB-1%%6~~~이거하나 뜨는데 선택한다.    
  
보안정책 그대로 두고   
기본SSL인증서 --->방금 만든거 선택.    
  
저장.   
  
리스너 생성했다고 뜬다.      
뒤로 가보면 추가 HTTPS : 443 추가 됬는데 문제는    
HTTP: 443 옆에 경고 표시가 있다.     
443 포트로 받아서 eb로 넘기는건 같게 표시되어있다.     
![6](https://imgur.com/7Gxhbjs)   

아까 eb생성할때 security그룹이 두개 생겼는데   
하나는 eb로 만들어진 environment대한것   
다른하나는 loadbalancer의 보안그룹.  

loadbalancer의 보안그룹 가보자.    
 인바운드 보면 80번 밖에 허용 안하고 있다.       
이것이 443도 허용하도록 하고 소스는 전체  0.0.0.0 받도록 하자.   
HTTPS  tcp   tcp 443 위치무관  0.0.000         

이제 앞으로 https로 접근해도 해당 사이트에 접근 가능해 진다.    
https://nssmr.com   
접근 해보면 이제 안전하지 않은 사이트 안뜬다.     
자물쇠 눌러보면    
아마존에서 인증해준 사이트라고하면서 인증서 뜬다. 보안 위협도 안뜬다.      

## 3.http요청시 https로 redirect설정   


https를 붙이지 않아도 nssmr.com접근하도록 해보자.    
http로 접근해도 자동으로 https붙도록 설정 하고 싶다.  ---------->nginx에서 설정 해준다.    
  
eb ssh해보자.   
여기에 지금 nginx설정 있는데 거기로 가보자.   
`cd /etc/nginx/site-available   `   `elasticbeanstalk -nginx-docker-proxy.conf`(nginx에서 docker로 proxy연결을 만들어준다. )
이것은 ec2의 설정되어있는 nginx이다. 80번 포트에 대해서 요청을 받을 것이다.    
cat명령어로 출력해보면    
 ```
    map $http_upgrade $connection_upgrade {
        default        "upgrade";
        ""            "";
    }
    
    server {
        listen 80;            80번 포트로 요청을 받았을때 

        gzip on;
        gzip_comp_level 4;
        gzip_types text/html text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

        if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2})") {
            set $year $1;
            set $month $2;
            set $day $3;
            set $hour $4;
        }
        access_log /var/log/nginx/healthd/application.log.$year-$month-$day-$hour healthd;

        access_log    /var/log/nginx/access.log;
    
        location / {                                                                    그 요청을 여기로 보낸다는것. 
            proxy_pass            http://docker;
            proxy_http_version    1.1;
    
            proxy_set_header    Connection            $connection_upgrade;
            proxy_set_header    Upgrade                $http_upgrade;
            proxy_set_header    Host                $host;
            proxy_set_header    X-Real-IP            $remote_addr;
            proxy_set_header    X-Forwarded-For        $proxy_add_x_forwarded_for;
        }
    }
```     

우리가 하고싶은것은 443으로 요청을 받았을때에 대해 조작하고 싶다. 

이것 수정하고 싶다. 
vim으로 수정해 보자. 

`sudo vim elasticbeanstalk-nginx-docker-proxy.conf`

```
    map $http_upgrade $connection_upgrade {
        default        "upgrade";
        ""            "";
    }
    
    server {
        listen 80;

        gzip on;
        gzip_comp_level 4;
        gzip_types text/html text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

        if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2})") {
            set $year $1;
            set $month $2;
            set $day $3;
            set $hour $4;
        }
        access_log /var/log/nginx/healthd/application.log.$year-$month-$day-$hour healthd;

        access_log    /var/log/nginx/access.log;
    
        location / {
            proxy_pass            http://docker;
            proxy_http_version    1.1;
    
            proxy_set_header    Connection            $connection_upgrade;
            proxy_set_header    Upgrade                $http_upgrade;
            proxy_set_header    Host                $host;
            proxy_set_header    X-Real-IP            $remote_addr;
            proxy_set_header    X-Forwarded-For        $proxy_add_x_forwarded_for;
        }
        if ($http_x_forwarded_proto = 'http') {
             return 301 https://$host$request_uri;
         }
    }
```

nginx설정에서 이 $http_x_forwarded_proto 변수 값이 오는데         
프로토콜의 스킴이다. 이것이 http로 왔는지, https로 왔는지에 대해한 설정이 오는것. 여기에 할당된다.   
이 변수가 만약 http로 온다면 https로 redirect로 해주는것.   
즉 http로 요청이오면 요청왓던 주소에  https를 앞에 붙여서 이쪽으로 돌아가라 요청을 보내준다.    
   
nginx 껐다가 켜야한다.      

`sudo service nginx restart`   
ok 두번 뜨면 된다.    
  
도메인으로 그냥 접근해본다.  
nssmr.com  ----->자동으로 https로 바뀐다.    

**이걸 아예  .ebextensions에 파일로 추가해 버리자.**   

**/.ebextensions/02_nginx_https_redirect.config**  
```
files:
"/etc/nginx/sites-available/elasticbeanstalk-nginx-docker-proxy.conf":
mode: "000644"
owner: root
group: root
content: |
map $http_upgrade $connection_upgrade {
default "upgrade";
"" "";
}

server {
listen 80;

gzip on;
gzip_comp_level 4;
gzip_types text/html text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2})") {
set $year $1;
set $month $2;
set $day $3;
set $hour $4;
}
access_log /var/log/nginx/healthd/application.log.$year-$month-$day-$hour healthd;

access_log /var/log/nginx/access.log;

location / {
proxy_pass http://docker;
proxy_http_version 1.1;

proxy_set_header Connection $connection_upgrade;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

if ($http_x_forwarded_proto = 'http') {
return 301 https://$host$request_uri;
}
}
```   
배포가 이뤄지기 전에 파일이 덮어지기 때문에    
마지막에 리다이렉트 설정 덮어진 상태로 배포 이뤄지는것.    
배포 다시 해라.   
./deploy.sh    




