
본 가이드는 다음 환경을 기준으로 설명합니다.
- 소스코드 저장소: gitlab
- CI/CD파이프라인 플랫폼: gitlab-runner
- App 배포/실행 플랫폼: Pivotal Application Service


### 구성흐름
 gitlab(project) -> gitlab-runner(docker) -> PAS(org/space)

## 사전구성사항
- gitlab계정:
- gitlab runner 구동환경 준비(VM)
- PAS계정,  배포공간: 애플리케이션이 배포될 공간  (org/space)
  

## 작업 순서

### git에 소스코드 업로드
 - http://10.6.220.89:8070/khpark/msa_pilot_sample
 
### gitlab runner등록
gitlab project마다 CI/CD작업을 실행할 gitlabrunner를 등록한다. =>  [gitlab-runner등록가이드 참조](/wiki url/)

### 수동 배포 테스트
CI/CD로 작성하기 전에 개발환경에서 수동으로 애플리케이션을 PAS에 배포해본다.


#### 배포파일 작성

```manifest.yml
---
applications:
- name: purchaseitem
  memory: 1024M
  instances: 1
  path: ./item-boot/build/libs/xxxx.jar"
  buildpack: java_buildpack_offline
  services:
  - service-registry
```

#### 배포
  1. cf7가이드: https://docs.pivotal.io/platform/application-service/2-8/cf-cli/v7.html
  1. docker container환경으로 내려받기: gitlab 이나 artifactory에  올려놓고  wget으로 내려받아 실행
    - download repo: https://github.com/cloudfoundry/cli/releases
   linux에서는 release다운로드하지 말고 아래 스크립트를 실행해서 설치해야함.
    ```
wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
sudo apt-get update
sudo apt-get install cf7-cli
```

  1. 배포
```
cf7 login -a https://api.sys.pas.hvzpilot.com -u test -p test1! --skip-ssl-validation -o msa-pilot -s dev-space
cf7 push  -f manifest.yml
curl -k https://purchaseitem.pas.hvzpilot.com
```




### CI/CD 파이프라인 작성
git  프로젝트의 root에  .gitlab-ci.yml파일을 생성한다.
 예) http://10.6.220.89:8070/msa-pilot/purchaseitem/.gitlab-ci.yml

java + maven build 기준 작성가이드: https://docs.gitlab.com/ee/ci/examples/artifactory_and_gitlab/index.html
springboot + cf push : https://docs.gitlab.com/ee/ci/examples/deploy_spring_boot_to_cloud_foundry/index.html

#### purchaseitem/.gitlab-ci.yml의 예시

```
stages:
  - build_stage
  - deploy_stage

build_job:
  stage: build_stage
 #  image: maven:3.6.3-jdk-8   ==> 선택적으로 docker image지정. 미지정시 gitlab-runner에 지정된 이미지를 로딩

  artifacts:                            ==> 이 파이프라인의 다른 job에 파일을 전달해야할 때 목록 지정
    paths:
      - "item-boot/build/libs/*.jar"
  tags:
    - gradle                           ==> gitlab-runner를 등록할 때 부여한 태크. gitlab 프로젝트> 설정> CI/CD> runner에서 표시된 태그와 매칭됨. 

  script:                                ==>  bash스크립트를 자유롭게 기술.
    - echo "Building project "
    - export NEXUS_ID=hkmc-dev
    - export NEXUS_PASSWORD=hkmc123
    - gradle clean build assemble


deploy_job:
  stage: deploy_stage

  tags:
    - gradle

  script:
    - echo "Building project with deploy"
    - wget http://10.6.220.89:8070/khpark/msa-platform-repo/raw/master/cf7
    - chmod +x ./cf7
    - ./cf7 login -a https://api.sys.pas.hvzpilot.com -u test -p test1! --skip-ssl-validation -o msa-pilot -s dev-space
    - ./cf7 push  -f manifest.yml --strategy=rolling

```

- docker container에 cf cli설치방법 
  1. cf7가이드: https://docs.pivotal.io/platform/application-service/2-8/cf-cli/v7.html
  1. docker container환경으로 내려받기: gitlab 이나 artifactory에  올려놓고  wget으로 내려받아 실행
  

#### purchaseitem-front/.gitlab-ci.yml의 예시

```
stages:
  - build_deploy_stage

build_deploy_job:
  stage: build_deploy_stage
  image: node:latest

  tags:
    - npm

  script:
    - npm install
    - npm run build_app
    - mkdir -p ./push_bin/public/ui
    - rm -rf ./push_bin/public/ && cp -r ./public  ./push_bin/
    - rm -rf ./push_bin/public/ui/purchaseitem && mv dist ./push_bin/public/ui/purchaseitem
    - rm -rf ./push_bin/public/index.html && ./push_bin/public/ui/purchaseitem/index.html ./push_bin/public/index.html
    - cp ./nginx.conf ./push_bin/
    -  cp ./mime.types ./push_bin/
    - cp ./buildpack.yml ./push_bin/
    - echo "Building project with deploy"
    - wget http://10.6.220.89:8070/khpark/msa-platform-repo/raw/master/cf7
    - chmod +x ./cf7
    - ./cf7 login -a https://api.sys.pas.hvzpilot.com -u test -p test1! --skip-ssl-validation -o msa-pilot -s dev-space
    - ./cf7 push  -f manifest.yml --strategy=rolling
```


### PAS에서 배포된 애플리케이션 확인하기
- https://api.sys.pas.hvzpilot.com에 로그인
- msa-pilot 조직,  dev-space 공간에 배포된 애플리케이션 확인.

