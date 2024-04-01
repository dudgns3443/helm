## 개요

현재 운영 리소스들을 공통으로 관리되어있지 않아 팀업무의 효율성 및 시스템에 전반적인 이해에 불편한 점이 많음

코드를 git이라는 레포로 관리하는 것 처럼 운영도 git으로 공통관리하는 GitOps를 실천하기 위해 도입 및 시작

## ST Unitas 클러스터구조 

클러스터 - 공통리소스들(ingress-controller, ingress, agent, configmap, serviceAccount 등) - 여러서비스들(기술단기, 공단기, 노무단기 등)
으로 구성되어 있다

클러스터당 공통 리소스를 먼저배포 후 각 서비스를 배포해야함

## datadog 배포

dependancy로 추가하려 했지만 현재 실패 그래서 따로 따로 배포해야함

서비스 배포전에 먼저 해주어야 한다. (나중에 해도 되지만 agent먼저 배포하는게 편하다)
```
helm repo add datadog https://helm.datadoghq.com
```
repo등록후
```
helm install datadog-agent -f ./values/common/datadog-values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey={api키} datadog/datadog
```

## 배포 명령어

클러스터당 공통리소스들을 먼저 배포한 뒤에 각 서비스들을 배포하는 형태로 구성해야한다.

1. 공통 리소스 배포

_현재 dependancy들(ingress controller, datadog agent 등)은 미완성이며 옮겨질 클러스터들은 이미 다 provisioning되어있음 ingress만 선택적으로 설치하는 것을 권장_

ingress 첫 배포시에는 host url을 test용 url로 작성하는 것을 권장 예) gong-test.conects.com 등
configmap이 필요한 클러스터의 경우 configmap.enabled=true 추가

예)
```
helm install conects-gong-config -f values\services\ingress-conects-gong-values.yaml -f values\common\values-common-prod.yaml --set ingress.enabled=true charts/prod --dry-run
```

2. 서비스 배포

datadog client를 설치해야하는 서비스 , php.ini가 설정되어야하는 서비스는 volumes-common-prod.yaml 변수 파일 추가 및 volumes 변수를 추가해준다
추가될 볼륨이 없을 경우는 변수파일을 적용안하면 됨
실제 적용전 dryrun 및 debug를 이용해 테스트해본다

예)
```
helm install {서비스이름} -f {환경 공통 변수 파일} --set {원하는 옵션 true} ./charts/{env}
# 실제 적용전 테스트적용 dryrun 및 debug
helm install conects-gong -f .\values\common\values-common-prod.yaml -f .\values\common\volumes-common-prod.yaml --set nodeSelector.service=gong --set volumes.datadogClient=true --set volumes.iniEnabled=true --set autoscaling.enabled
=true --set configmap.enabled=true .\charts\prod\ --dry-run --debug
# 
helm upgrade conects-gong -f .\values\common\values-common-prod.yaml -f .\values\common\volumes-common-prod.yaml --set nodeSelector.service=gong  --set autoscaling.enabled=true .\charts\prod\ --dry-run --debug


# 실제 적용 volums 변수들은 선택적으로 넣는다.
helm install conects-gong -f .\values\common\values-common-prod.yaml -f .\values\common\volumes-common-prod.yaml --set volumes.datadogClient=true --set volumes.iniEnabled=true --set autoscaling.enabled
=true .\charts\prod\ 

helm upgrade conects-gong -f .\values\common\values-common-prod.yaml -f .\values\common\volumes-common-prod.yaml --set volumes.datadogClient=true --set volumes.iniEnabled=true --set autoscaling.enabled
=true .\charts\prod\ 
```


## setting 필요한 옵션

기본적으로 모두 다 false이다. 선택적으로 업데이트/생성하고 싶은 것만 --set 이용해 enabled=true로 바꾸어 주자
```
dependancys:
    aws-load-balancer-controller.enabled
    serviceAccount.enabled
    datadog.enabled

services:
    ingress.enabled            : ingress 배포여부
    autoscaling.enabled        : service, deployment, hpa 배포 여부
    configmap.enabled          : php.ini configmap 적용 여부
    nodeSelector.service       : 스케줄링할 노드 지정 (클러스터 통합 시)
#HPA min, max 파드값 기본 값은 2 to 10 이지만 각 서비스마다 다를 수가 있기에(예: gong, my) 따로 설정해 준다
    autoscaling.minReplicas : 기본 2
    autoscaling.maxReplicas : 기본 10
# datadog apm 설치 여부 및 php.ini 설정 여부에 따라 true로 선택
    volumes.datadogClient   : 기본 false
    volumes.iniEnabled      : 기본 false
```

## helm으로 k8s 리소스 설치 이후 각 서비스 cicd 변경 작업

각 서비스 gitlab 레포에서 테스트용 브랜치 생성
staging/Dockerfile.staging에 FROM 이미지 534420079206.dkr.ecr.ap-northeast-2.amazonaws.com/pre_source:php7.4-alpine3.16 로 변경


Dockerfile에 
FROM image 01_conects_[서비스이름]:staging 으로변경
datadog-php-client 설치 및 환경변수 설정 넣기 (datadog 설치대상 서비스만)

ECR에 01_conects_[서비스이름] 으로 컨테이너 저장소 생성 예) 01_conects_gong 

gitlab-ci 수정

staging 컨테이너의 주소를 바꿨으므로 staging 에서 가져오는 컨테이너 이미지 주소도 바꿔줘야한다
web-deploment.staging.yaml 수정

        image: 534420079206.dkr.ecr.ap-northeast-2.amazonaws.com/01_conects_[서비스이름]:staging

route53 에서 ingress에서만든 test url과 새로운 클러스터의 ingress 로드밸런서 연결 후 테스트
ingress에서 test url(gong-test.conects.com) -> 운영 url(gong.conects.com)로 변경 WAF에서 운영 url의 타겟 로드밸런서를 새로만든 클러스터의 로드밸런서로 수정
staging 및 production 브랜치에 수정사항 merge 및 cherry-pick

## 클러스터 통합

01-conects-gong 클러스터로 통합함에 따라 각 서비스들의 카테고리를 nodeGroup 및 namespace로 분리
01-conects-gong 에서 nodeGroup 생성시 *서브넷*과 *레이블* 설정에 주의 
```
kubectl create namespace [카테고리명] 예) exam, lang, gong, my 등
```
ingress-[서비스명]-values.yaml 생성시 
```
alb.ingress.kubernetes.io/group.name: conects
```
어노테이션추가

helm install 시 네임스페이스 지정

```
helm install conects-gong -n gong ..이하생략 .\charts\prod\ 
```

각 repo 에서 gitlab-ci.yaml 수정
```
aws eks update-kubeconfig --name 01-conects-gong 
kubectl rollout restart deployments/conects-[서비스명] -n [카테고리명]
```
## 개선점 

- ArgoCD를 사용한 pull방식 배포 고려(각 서비스들의 컨테이너 버저닝이 선행되어야함)
