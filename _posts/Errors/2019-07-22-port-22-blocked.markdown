---
layout: "post"
title: "Public WIFI 에서 ssh 접속시  port 22 blocked"
categories:
- Errors

tags:
- git
comments : true
date: "2018-08-23 18:50"
---


**[상황]**
보통 카페에서는 git push, eb ssh 등 명령어실행이 잘 되는데 간혹 비밀번호가 없는 wifi를 쓸때는 ssh명령아가 동작하지 않음.

**[애러코드]** 
```
ssh: connect to host github.com port 22: Connection timed out
fatal: Could not read from remote repository.

Please make sure you have the correct access rights

and the repository exists.

```

이것은 방화벽이나 ISP가 설정한 방화벽이 포트 22에서 ssh 연결하는것을 차단하는경>우 자주 발생한다고 함.


[여기](https://stackoverflow.com/questions/7953806/github-ssh-via-public-wifi-port-22-blocked/8081292#8081292)찾아보니 `~/.ssh/config` 에 다음을 추가하라 함.
```
Host github.com
  Hostname ssh.github.com
  Port 443
```
 

추가 해주고 git push 해보니 다음 뜨면서 push 됬다.   
```

The authenticity of host '[ssh.github.com]:443 ([192.xx.xxx.xxx]:443)' can't be established.
RSA key fingerprint is SHA256:.
Are you sure you want to continue connecting (yes/no)? yes

```
