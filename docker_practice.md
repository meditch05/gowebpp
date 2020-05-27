도커실습가이드

sudo -i

모든 작업은 root 로 하세요!! - 실제 업무환경에서는 root 허용을 보안상의 이유로 허용하지 않지만 테스트환경에서 sudo 권한의 작업을 쉽게 하기 위해서 root 계정으로 작업을 수행합니다. 

실습파일 복사하기

```
1. windows 에서 파일관리자를 열어서 폴더생성
   c:\share
2. gowebapp.tar.gz 파일을 c:\share 에 복사하고 나서 
   오른쪽 마우스 -> 속성 -> 공유 -> 공유버튼을 눌러서 everyone 을 추가시킨다.
3. ubuntu app 으로 와서
   mkdir /mnt/share
   mount -t drvfs '\\윈도우즈컴퓨터IP\share' /mnt/share
   mount -a
   mount
   ls /mnt/ls -> 로 gowebapp.tar.gz 을 확인해보기
   
   tar -zxvf /mnt/share/gowebapp.tar.gz  
   
   ls 
   gowebapp    -> 디렉토리 확인하기
```



도커설치하기

```
sudo -i
password: student

apt-get update 
apt install apt-transport-https ca-certificates curl software-properties-common 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" 
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io

docker [엔터]
docker version
dockr info


```

1. 실습파일 확인해보기 

   홈디렉토리 아래에 있는 gowebapp.tar.gz  파일의 압축을 푸세요

   ```
   tar -zxvf gowebapp.tar.gz 
   ```

2. 프론트엔드 어플리케이션의 이미지를 생성하기 위한 도커파일을 만드세요

   ```
   cd $HOME/gowebapp/gowebapp
   vim Dockerfile
   
   FROM ubuntu
   
   COPY ./code /opt/gowebapp
   COPY ./config /opt/gowebapp/config
   
   EXPOSE 80
   
   WORKDIR /opt/gowebapp/
   ENTRYPOINT ["/opt/gowebapp/gowebapp"]
   
   
   ```

3. gowebapp 이미지 생성하기

   ``` 
   cd $HOME/gowebapp/gowebapp
   docker build -t gowebapp:v1 .
   ```

4. 백엔드 어플리케이션이미지 생성을 위한 도커파일만들기

   ```
   cd $HOME/gowebapp/gowebapp-mysql
   vim Dockerfile
   
   FROM mysql:5.6
   COPY gowebapp.sql /docker-entrypoint-initdb.d/
   ```

5. gowebapp-mysql 도커이미지 생성하기

   ```
   cd $HOME/gowebapp/gowebapp-mysql
   docker build -t gowebapp-mysql:v1 .
   ```

   

6. 로컬레파지토리에 생성된 이미지 확인해보기 

   ```
   docker image ls
   docker history gowebapp:v1
   docker history gowebapp-mysql:v1
   ```

7. 백엔드와 프론트엔드 컨테이너를 연결하기 위한 네트웍생성

   ```
   docker network create gowebapp
   ```

   

8. 백엔드 와 프론트엔드 컨테이너 실행해보기 

   ```
   docker run --net gowebapp -p 43306:3306 --name gowebapp-mysql --hostname gowebapp-mysql -d -e MYSQL_ROOT_PASSWORD=mypassword gowebapp-mysql:v1
   ```

   1분정도 기다렸다가 프론트엔드컨테이너를 실행한다.

   ```
   docker run --net gowebapp  -p 9000:80 -d --name gowebapp --hostname gowebapp gowebapp:v1 
   ```

9. 브라우저를 띄우고 gowebapp 어플리케이션에 접속해보자

   http://node1_ip:9000

   계정을 임의로 생성하여 로그인하고  원하시는 텍스트를 입력하고 save 해보자. save 가 정상적으로 동작하면 프론트엔드 gowebapp 이 백엔드 gowebapp-mysql 과 정상적으로 연동되고 있다는 것이다. 

9. gowebapp-mysql 컨테이너에 접속하여 위에서 생성한 계정정보와 텍스트가 데이타베이스에 정상적으로 저장되었는지 확인해보자.

   ```
   docker exec -it gowebapp-mysql mysql -u root -pmypassword gowebapp
   
   SHOW DATABASES;
   USE gowebapp;
   SHOW tables;
   select * from user;
   select * from note;
   ```

10. 컨테이너에 대한 정보 확인해보기

    ```
    docker inspect gowebapp
    docker diff gowebapp-mysql
    ```

11. 생성한 컨테이너 삭제하기

    ```
    docker rm -f gowebapp gowebapp-mysql
    docker ps 
    ```

12. 로컬레파지토리에 저장된 로컬이미지를 docker hub 레파지토리에  push  하자.

    http://hub.docker.com
    
    ```
    docker login 
username: gklab
    password: global123
    ```
    
    ```
docker tag gowebapp:v1 gklab/gowebapp:v1
    docker tag gowebapp-mysql:v1 gklab/gowebapp-mysql:v1 
    ```
    
    ```
    docker push gklab/gowebapp:v1
docker push gklab/gowebapp-mysql:v1
    ```
    
    
    
    

