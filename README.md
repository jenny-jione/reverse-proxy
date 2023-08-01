# Docker Compose로 localhost Nginx 리버스 프록시 구성

Hi! I'm your first Markdown file in **StackEdit**. If you want to learn about StackEdit, you can read me. If you want to play with Markdown, you can edit me. Once you have finished with me, you can create new files by opening the **file explorer** on the left corner of the navigation bar.
내부에 Docker로 2개의 컨테이너를 올린다. 각각 Nginx(reverse proxy)와 Nginx(서버) 컨테이너를 올린다.

reverse proxy 서버는 80번 포트를 open하고 뒷단의 nginx 서버는 8080 포트를 오픈한다. 크롬에서 localhost를 입력하면 reverse proxy 서버가 뒷단의 nginx의 index.html를 보여주는 것이 목표이다.


## 1. 디렉토리 구조

```
├─ docker-compose.yml
├─ nginx
│	└─ default.conf
├─ proxy
│	└─ nginx.conf
└─ source
	└─ index.html
```

## 2. docker-compose.yml

```
version: '3'
  services:
    proxy:
      image: nginx:latest
      ports:
        - "80:80"
      volumes:
        - ./proxy/nginx.conf:/etc/nginx/nginx.conf
    web:
      image: nginx:latest
      expose:
        - "8080"
      volumes:
        - ./source:/source
        - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
```
**Proxy**: Reverse Proxy 서버 역할을 할 Nginx Container.
80번 포트를 host와 공유하고 있다. volumes에서는 host pc의 proxy/nginx.conf 파일을 container의 /etc/nginx/nginx.conf로 공유하고 있다.

**web**: 실제 html의 결과를 보여줄 웹서버. expose 8080 옵션으로 다른 container와 8080 포트를 통해 통신할 수 있다. volumes에서는 source 디렉토리를 공유하고 있다.

## 3. nginx.conf
reverse proxy 서버 역할을 할 nginx의 nginx.conf이다.
```
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {                     
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    upstream docker-nginx {
        server web:8080;
    }
    server {
        listen 80;
        server_name localhost;
        location / {
            proxy_pass         http://docker-nginx;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;          
        }
    }
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
                                                
    sendfile        on;                                                                         
    keepalive_timeout  65;                                                                      
    include /etc/nginx/conf.d/*.conf;           
}
```

proxy 설정을 위해 굵게 표시한 부분
```
upstream docker-nginx {
	server web:8080;
}
```
docker-nginx라는 이름을 가진 upstream을 정의한다. server web:8080으로 되어 있는데 여기서 web은 docker-compose.yml에서 정의한 웹서버 nginx의 이름이다. 만약 docker-compose.yml에서 이름을 web이 아닌 nginx로 줬다면 server nginx:8080; 으로 명시해야 한다.

```
server {
        listen 80;
        server_name localhost;
        location / {
            proxy_pass         http://docker-nginx;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;          
        }
}
```
Reverse Proxy 서버는 80번 포트를 listen하고 server_name은 localhost로 지정했다. **proxy_pass** 설정을 보면 / 로 들어올 경우 위에서 정의한 upstream docker-nginx(web이라는 이름을 가진 container의 8080 포트)로 proxy한다.


## 4. default.conf
웹서버 역할을 할 Nginx의 default.conf 내용이다.
```
server {
    listen 8080;
 
    server_name example.com www.example.com;
    root /source;
 
    error_log /var/log/nginx/api_error.log;
    access_log /var/log/nginx/api_access.log;
}
```
주요한 설정만 보면 nginx가 8080 포트에서 listen하고 있으며 root 디렉토리는 /source를 사용하고 있다.