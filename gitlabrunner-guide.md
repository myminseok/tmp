
[Gitlab runner등록 방법]

## gitlab runner vm ssh 로그인
- IP :  
- ID : ubuntu


```
cd /srv
root@gitlab-runner:/srv# ls -al
total 56
drwxr-xr-x  5 root root 4096  3월  2 16:46 .
drwxr-xr-x 25 root root 4096  2월 26 16:01 ..
drwxr-xr-x  2 root root 4096  2월 24 16:51 docker_images
drwxr-xr-x  6 root root 4096  3월  2 16:46 gitlab-runner
drwxr-xr-x  4 root root 4096  2월 24 16:28 maven-repo
-rwxr-xr-x  1 root root  341  3월  2 16:34 remove_runner.sh
-rwxr-xr-x  1 root root  929  3월  2 16:43 run_regi_total.sh
```

## gitlab 프로젝트용 gitlab runner 등록
- run_regi_total.sh 를 실행 step별 실행 하는것을 한번에 적용한다.

실행형식은 아래와 같다
 ./run_regi_total.sh        git_project_name      registration-token                             tag  
./run_regi_total.sh         purchase-item          bB59gtzLN65NUbsZQqVv               maven

git_project_name : gitlab 프로젝트 명으로 gitlab runner의 도커 프로세스 이름이 됨, unique해야함
registration-token : gitlab runner 가 등록될 gitlab프로젝트 settings - CI/CD - runners - expand 해서 등록 된 token 내용 확인
tag : maven 사용


## 등록 상태 확인
### gitlab runner docker프로세스
-등록시 사용한 gitlab 프로세스 명으로 도커 프로세스가 있는지 확인

```
root@gitlab-runner:/srv# docker ps |grep purchase-item
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS                               NAMES
a4be7aa0c06e        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   4 seconds ago       Up 3 seconds                    purchase-item

```

### gitlab UI 확인
- gitlab runner 가 등록될 gitlab프로젝트 settings - CI/CD - runners - expand 해서 등록 된 gitlab runner tag확인(maven)


##등록 / 삭제
### gitlab UI에서 삭제
- gitlab runner 가 등록될 gitlab프로젝트 settings - CI/CD - runners - expand 해서 등록 된 gitlab runner의 remove icon클릭하여 삭제

### gitlab runner VM서버 접속 후  docker 프로세스 삭제
```
remove_runner.sh   purchase-item 
```

-docker ps -a 수행시 결과가 공란 인지 확인
```
root@gitlab-runner:/srv# docker ps |grep purchase-item
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS                               NAMES

```





# 참고 - 내부 동작순서

## 1. 도커에 컨테이너를올리기
/srv/run-docker-test-runner.sh

root@gitlab-runner:/srv# vi run-docker-test-runner.sh
docker run -d --name test-runner --restart always \
          -v /srv/gitlab-runner/config_test:/etc/gitlab-runner \
            -v /var/run/docker.sock:/var/run/docker.sock \
              gitlab/gitlab-runner:latest

./run-docker-test-runner.sh실행

root@gitlab-runner:/srv# ./run-docker-test-runner.sh
a4be7aa0c06e298a39181cbb5e02c4864396b9e5b0a3fdac7a713f847d84ed06

root@gitlab-runner:/srv# docker ps
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS               NAMES
a4be7aa0c06e        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   4 seconds ago       Up 3 seconds                            test-runner
6ea77873c08a        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   3 days ago          Up 3 days                               npm-runner
b85b2d64ed0b        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   4 days ago          Up 4 days  


## 2. test-runner 컨테이너가 실행 됨
a4be7aa0c06e        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   4 seconds ago       Up 3 seconds                            test-runner


## 3. docker 레지스터등록하기(gitlab -runner사이트를 참조하여 등록진행)
--------------------------------------------------------------------------------------------------------------------
[gitlab -runner사이트 정보]
CD/CD 등록  : settings - CI/CD - runners - expand 해서 등록 된 내용 확인 가능
URL 정보, 토큰 확인 가능

gitlab 주소 : http://10.6.220.89:8070/khpark/msa_pilot_sample  접

프로젝트 정보 : msa_pilot_sample
purchase-item-front : 프론트엔드 
purchase-item :백엔드 
-----------------------------------------------------------------------------------------------------------------------
root@gitlab-runner:/srv# ./register-test.sh
Runtime platform                                    arch=amd64 os=linux pid=6 revision=1b659122 version=12.8.0
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://10.6.220.89:8070/
Please enter the gitlab-ci token for this runner: (토큰 입력)
bB59gtzLN65NUbsZQqVv
Please enter the gitlab-ci description for this runner: (러너 이름)
[49b2216ab906]: test
Please enter the gitlab-ci tags for this runner (comma separated):( 태그,별칭)
test
Registering runner... succeeded                     runner=bB59gtzL
Please enter the executor: parallels, shell, docker-ssh, ssh, virtualbox, docker+machine, docker-ssh+machine, kubernetes, custom, docker:
docker
Please enter the default Docker image (e.g. ruby:2.6): 도커 이미지 이름 등
maven:3.6.3-jdk-8
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

docker run --rm -t -i -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest register \
        --docker-privileged \
        --non-interactive \
        --executor "docker" \
        --docker-image maven:3.6.3-jdk-8 \
        --url "http://10.6.220.89:8070/" \
        --registration-token "bB59gtzLN65NUbsZQqVv" \
        --description "test-runner" \
        --tag-list "maven" \
        --run-untagged \
        --locked="false"

아래 스크립트를 이용할 경우  register가 한번에 실행됨
root@gitlab-runner:/srv# vi register-test.sh
docker run --rm -t -i -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest register \
        --docker-privileged \
        --non-interactive \
        --executor "docker" \
        --docker-image maven:3.6.3-jdk-8 \
        --url "http://10.6.220.89:8070/" \
        --registration-token "bB59gtzLN65NUbsZQqVv" \
        --description "test-runner" \
        --tag-list "maven" \
        --run-untagged \
        --locked="true"

## 4. 도커 상태 확인

root@gitlab-runner:/srv# docker ps
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS               NAMES
6fdb01e59c0a        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   6 minutes ago       Up 6 minutes                            test-runner
6ea77873c08a        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   3 days ago          Up 3 days                               npm-runner
b85b2d64ed0b        gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   5 days ago          Up 5 days                               gitlab-runner
root@gitlab-runner:/srv#

## 5. git lab runner에 등록 정보 확인 하기
http://10.6.220.89:8070/hkmc-msa-pilot/msa-pilot/purchase-item/purchase-item-front/-/settings/ci_cd
 
Runners activated for this project
 6E1XLtsw  #31
test-runner
maven
값을 확인
--------------------------------------------------------------------------------------------------------------------

CD/CD 등록  : settings - CI/CD - runners - expand 해서 등록 된 내용 확인 가능
URL 정보, 토큰 확인 가능

gitlab 주소 : http://10.6.220.89:8070/khpark/msa_pilot_sample  접속

프로젝트 정보 : msa_pilot_sample
purchase-item-front : 프론트엔드 
purchase-item :백엔드 
-----------------------------------------------------------------------------------------------------------------------


