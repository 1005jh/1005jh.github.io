---
title: Next.js 14버전 AWS Amplify 배포
date: "2023-12-18T12:36:37.121Z"
template: "post"
draft: false
category: "nextjs"
tags:
  - "nextjs"
  - "amplify"

description: "Next.js 14버전 AWS Amplify 배포"
---

11월 초 next.js 14버전의 aws amplify 배포를 하며 겪었던 문제이다.
aliy 프로젝트에서 nextjs의 배포를 aws amplify를 이용하기로 했었다. 그러나 nextjs 14버전은 빌드 시 node 버전이 18버전 이상이여야 하는 이슈가 있었다.
aws amplify cli에서 제공하는 node 버전은 17버전까지가 끝이었다.
```
## Install AWS Amplify CLI for all node versions
RUN /bin/bash -c ". ~/.nvm/nvm.sh && nvm use ${VERSION_NODE_8} && \
    npm config set user 0 && npm config set unsafe-perm true && \
	npm install -g @aws-amplify/cli@${VERSION_AMPLIFY}"
RUN /bin/bash -c ". ~/.nvm/nvm.sh && nvm use ${VERSION_NODE_10} && \
    npm config set user 0 && npm config set unsafe-perm true && \
	npm install -g @aws-amplify/cli@${VERSION_AMPLIFY}"
RUN /bin/bash -c ". ~/.nvm/nvm.sh && nvm use ${VERSION_NODE_12} && \
    npm config set user 0 && npm config set unsafe-perm true && \
	npm install -g @aws-amplify/cli@${VERSION_AMPLIFY}"
RUN /bin/bash -c ". ~/.nvm/nvm.sh && nvm use ${VERSION_NODE_14} && \
    npm config set user 0 && npm config set unsafe-perm true && \
	npm install -g @aws-amplify/cli@${VERSION_AMPLIFY}"
RUN /bin/bash -c ". ~/.nvm/nvm.sh && nvm use ${VERSION_NODE_16} && \
    npm config set user 0 && npm config set unsafe-perm true && \
	npm install -g @aws-amplify/cli@${VERSION_AMPLIFY}"
RUN /bin/bash -c ". ~/.nvm/nvm.sh && nvm use ${VERSION_NODE_17}  && \
    npm config set user 0 && npm config set unsafe-perm true && \
	npm install -g @aws-amplify/cli@${VERSION_AMPLIFY}"
```
빌드를 하는 과정에서 node 버전에 대한 이슈로 에러가 생겼고, nextjs의 14버전을 aws apmlify에 배포하기 위해선 node의 버전을 바꿔주는 추가적인 작업이 필요했다.

amplify의 배포설정에서 빌드 이미지 세팅에 node 버전을 바꿔주고 해결할 수 있었는데
node version을 선택하고 `public.ecr.aws/docker/library/node:18.18.2` 을 입력해 node 버전의 문제를 해결할 수 있었다.

참고
<a href="https://github.com/aws-amplify/amplify-hosting/issues/3773">https://github.com/aws-amplify/amplify-hosting/issues/3773</a>