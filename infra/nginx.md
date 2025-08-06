# 💽 Nginx 포트 설정



AWS EC2나 GCP의 Compute engine 등을 사용해 배포 후 인스턴스의 ip로 접속을 해보면 접속이 안된다.

**WHY?**\
여러가지 이유가 있을 수 있겠지만, 포트를 설정 해준 경우에는 해야 할 것이 더 있다.

**http는 기본적으로 80포트로 통신한다.**\
그 말인 즉슨, 인스턴스의 외부 ip로 접속하면 자동으로 80포트로 접속이 된다는 것이다.\
3000번 포트나 톰캣의 기본 포트 8080, 혹은 도커 컨테이너에 설정한 포트로는 접속이 안되는 것이다.

\
\


> Nginx 설정

**인스턴스에 접속 후 실행 할 명령어**

```null
sudo apt-get install nginx
sudo service nginx start
```

* Nginx 설치 및 시작

```null
sudo vi /etc/nginx/sites-available/project.conf
```

* /etc/nginx/sites-available 경로에 conf파일 생성(conf파일 이름은 프로젝트 이름 등 자유롭게)\


```null
server {
    listen 80;
    server_name 인스턴스ip;

    location / {
        proxy_pass http://인스턴스ip:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

* 커스텀 도메인이 없는경우 server\_name에 인스턴스 ip 주소를넣어주기

```null
server {
    listen 80;
    server_name 도메인.site www.도메인.site;

    location / {
        proxy_pass http://인스턴스ip:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

* 커스텀 도메인이 있는경우 ip주소 대신 도메인을 넣어주기
* 이렇게 하면 기본 80포트로의 접속 혹은 도메인 접속을 8080포트로 리다이렉트 해준다.
* 그다음 저장! vi editor는 esc누른 후 :w 를 입력하면 저장 :q 를 입력하면 나가진다.\


\


**sites-enabled에 심볼릭 링크 만들기**

```null
sudo ln -s /etc/nginx/sites-available/project.conf /etc/nginx/sites-enabled
```

* 심볼릭 링크 만든 후

```null
cd /etc/nginx/sites-enabled
ls -l
```

* 잘 생성 되었나 확인해보자

```null
sudo rm -rf default
```

default 파일이 존재할탠데, 삭제해주자. 삭제안하면 nginx의 기본 참조가 default파일 이므로 설정이 반영 안될수도\
(Nginx의 메인 파일은 /etc/nginx/nginx.conf)

**!!마지막!!**

```null
sudo nginx -t
```

* Nginx 설정 유효성 검사 테스트 후 이상이 없다면

```null
sudo service nginx restart
```

* Nginx 재시작

AWS 기준 퍼블릭ip, GCP 기준 외부ip 혹은 커스텀 도메인으로 접속해보면 잘 될것이다.

\
\
\


> (추가) 심볼릭 링크?

파일을 단순히 복사하는 것과 심볼릭 링크를 생성하는 것의 차이점은 심볼릭 링크는 원본 파일을 가리키는 참조 포인터이다.\
즉, 원본 파일의 데이터를 포함하는 복사와 달리 원본 파일의 경로를 저장하는것.

**중간에 sites-available에 생성한 conf 파일의 심볼릭 링크를 sites-enabled에 생성해 줬었다. 그렇다면 그 이유는?**

sites-available : 가능한 모든 사이트의 설정 파일을 저장하는 곳\
sites-enabled : 실제로 Nginx에 의해 읽히고 사용되는 사이트 설정 파일의 심볼릭 링크를 저장하는 곳

* 모든 설정 파일을 sites-available에 보관함으로써, 어떤 설정이 가능한지 쉽게 파악할 수 있고, sites-enabled에 있는 심볼릭 링크를 통해 현재 활성화된 설정을 쉽게 확인할 수 있다.
* 실수로 중요한 설정 파일을 삭제하거나 잘못 수정하는 것을 방지할 수 있다. 심볼릭 링크를 제거하거나 추가하는 것은 원본 파일에 영향을 주지 않으므로, 설정을 안전하게 관리할 수 있다.
