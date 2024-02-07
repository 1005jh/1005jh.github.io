---
title: aws deploy, s3, ec2 cicd 분기처리
date: "2023-10-20T12:36:37.121Z"
template: "post"
draft: false
category: "devops"
tags:
  - "devops"

description: "aws deploy, s3, ec2 cicd 분기처리"
---

진행중인 aliy 프로젝트에서 aws s3와 codedeploy를 사용해 ec2에 자동배포를 했었다.
dev와 main 둘 다 같은 방식으로 진행을 했던터라 codedeploy 과정에서 애플리케이션을 실행하는 방식이 달랐다.

이 문제를 해결하기 위해 어떤 방식이 있을지 고민했던 것들을 적어보겠다.

### 1. workflow에서 해결

우선적으로 생각을 했던게 브랜치가 서로 다르기 때문에 workflow 단계에서 할 수 있지 않을까? 라는 생각이었다.
그래서 workflow에서 github contexts를 이용해 보기로 했다.

github contexts로 브랜치에 따라 다른 스크립트 파일을 만들어 해결할 수 있을거라는 생각으로 시도를 했었다.

<a href='https://docs.github.com/en/actions/learn-github-actions/contexts'>링크</a>를 참조하여 브랜치에 따라 스크립트 파일을 만들어서 보내주는 방식으로 진행을 했었다.

첫번째로 권한에러가 나왔다. workflow에서 만든 스크립트 파일에 대해 ec2에서 실행권한이 없는 문제였다.
권한을 부여해주고 나서 다시 github action을 다시 시도해보았으나 권한 문제가 해결되지 않았다.

그래서 다른 방식으로 넘어갔다.

### 2. appspec 단계에서 해결

appspec에는 codedeploy에서 실행되는 hook을 설정할 수 있다.

```shell
# 예시
version: 0.0
os: linux
files:
  - source: Config/config.txt
    destination: /webapps/Config
  - source: source
    destination: /webapps/myApp
hooks:
  BeforeInstall:
    - location: Scripts/UnzipResourceBundle.sh
    - location: Scripts/UnzipDataBundle.sh
  AfterInstall:
    - location: Scripts/RunResourceTests.sh
      timeout: 180
  ApplicationStart:
    - location: Scripts/RunFunctionalTests.sh
      timeout: 3600
  ValidateService:
    - location: Scripts/MonitorService.sh
      timeout: 3600
      runas: codedeployuser
```

hook 단계에서 스크립트 파일의 경로를 설정해주니 해결할 수 있지 않을까 했지만 공식문서를 봐도 appspec에서 처리할 수 있는 방법에 대한 건 찾아볼 수 없었다.
그래서 스크립트 파일에서 처리를 해보기로 했다.

### 3. 스크립트 파일에서 해결

첫번째로 위의 appspec에서 두개의 스크립트를 넣고 스크립트 파일 내에서 실행여부를 판단하는 방식으로 접근했다.

```shell
hooks:
... 이전 단계
  AfterInstall:
    - location: 경로
      timeout: 180
      runas: ubuntu
  ApplicationStart:
    - location: 경로
      timeout: 180
      runas: ubuntu
    - location: 경로
      timeout: 180
      runas: ubuntu
```

위와 같은 방식으로 진행했었다.
애플리케이션을 시작하는 단계에서 두개의 스크립트가 실행이 되는 방식이고 dev와 main 서버에서 사용하는 env파일이 달랐기 때문에 이를 if문으로 관리를 했다.
위의 과정이 성공적으로 이뤄졌고, if문으로 성공을 했기 때문에 하나의 스크립트 파일로 컨트롤이 가능했다. 그래서 스크립트 파일을 하나만 설정해줬다.

```shell
# ...이전 과정
if [ -f ".env.dev" ]; then
  echo "Running script for dev branch"
  # dev 애플리케이션 실행
elif [ -f ".env.prod" ]; then
  echo "Running script for main branch"
  # main 애플리케이션 실행
fi
```

결국 하나의 스크립트로 해결할 수 있었다.

### 마치며

위 작업들을 하며 스크립트 파일을 통해 여러가지 작업을 할 수 있다는 걸 느낄 수 있었다. 물론 지금 해결한 방식보다 좋은 방식도 있을거라 생각하고 찾아보며 시도하고 적용해봐야겠다.
1번 workflow를 사용하는 과정에서 권한문제에 관한 건 무엇때문인지 왜 그런건지 조금 더 찾아봐야 할 것 같다.
