---
title: GitLab 배포사항 자동 기록 개선하기
date: 2024-06-23 21:20:00 +09:00
categories: [Deveploment, GitLab]
tags: [aws, lambda, gitlab, ci, changelog] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 돌아보기
<img src="/assets/img/capture/gitlab-changelog-13.png" alt="CHANGELOG.md 일부 이미지" /> <br />
위 이미지는 이전에 구현한 CHANGELOG.md 자동 업데이트가 동작한 결과 중 일부 이미지다. 해당 기능에 대한 자세한 설명은 [GitLab 배포 사항 자동 기록하기](https://wslee94.github.io/posts/GitLabChangeLog/){:target="_blank"} 여기를 참고하자! 꾸준히 잘 사용하고 있었지만 개선하면 더 좋을 것 같은 점들이 있어 추가로 작업했다. 작업하면서 기록으로 남기고 싶은 부분이 있어 게시글을 작성한다.

# 개선하고 싶은 부분

## 1. 점차 많아지는 프로젝트 → GitLab 저장소 증가
<img src="/assets/img/capture/gitlab-changelog-14.png" alt="CHANGELOG.md를 업데이트하는 자바스크립트 위치 이미지" /> <br />
위 이미지를 보면 CHANGELOG.md 를 업데이트하는 자바스크립트(`generate_changelog.js`)가 GitLab 저장소 내부에 위치한 것을 확인할 수 있다. 메이플스토리 월드 관련 프로젝트가 많아짐에 따라 스크립트가 여러 저장소에 중복해 존재하는 문제점이 있다. 이런 문제로 인해 `generate_changelog.js`에서 수정사항이 발생할 경우 동일한 작업을 저장소 개수만큼 반복해야 한다.

## 2. 중복으로 발생하는 CHANGELOG.md 업데이트 커밋
<img src="/assets/img/capture/gitlab-changelog-15.png" alt="CHANGELOG.md를 업데이트 커밋 중복 이미지" /> <br />
`master` 타겟 브랜치로 MR이 발생하는 순간 GitLab CI에서 CHANGELOG.md 업데이트하는 Job을 실행시키는데, 문제는 등록한 MR이 머지되기 전에 소스 브랜치에 새로운 커밋(ex. 충돌 해결 커밋)이 생기면 Job이 반복적으로 실행되어 master 브랜치 커밋 히스토리가 지저분해진다.

# 개선하기

## 1. AWS Lambda 함수로 자바스크립트 옮기기
> **AWS Lambda?** <br />
> AWS Lambda는 서버 구축 없이 AWS가 제공하는 환경(FaaS)에 코드를 배포하여 실행할 수 있는 서버리스 컴퓨팅 서비스이다.
{: .prompt-tip }
AWS Lambda를 이용해 CAHNGELOG.md를 업데이트하는 자바스크립트(`generate_changelog.js`)가 프로젝트 저장소 외부에 위치하도록 변경했다.

### 난관 봉착
AWS Lambda 함수 내부에서 사내 GitLab API를 사용하려면 네트워크 허용 처리가 필요한데, 네트워크 허용은 출발지 IP와 목적지 IP가 필요하다.

- 출발지 IP: Lambda 함수 실행 환경 IP
- 목적지 IP: 사내 GitLab 서버 IP

문제는 기본적으로 Lambda 함수의 경우 고정된 Public IP를 갖지 않는다. Lambda 함수는 요청에 따라 동적으로 할당되는 컨테이너에서 실행된다. 따라서 실행마다 다른 물리적 위치에서 실행될 수 있다.

### 난관 해결
결론을 미리 얘기하면, Lambda에 고정 IP(=Source IP)를 부여하기 위해 VPC + NAT 게이트웨이의 조합을 사용할 수 있다.

설명에 앞서 용어 정리를 해보자!
- **VPC**: AWS 환경에서 논리적으로 격리된 네트워크 공간을 의미
- **Subnet**: VPC 내부 인스턴스의 IP 범위
  - **Public subnet**: Internet과 연결되어 인바운드, 아웃바운드 트래픽을 주고받을 수 있음
  - **Private subnet**: 외부에 노출되지 않아 인터넷에 접근할 수 없음
- **Routing table**: 서브넷 내부 인스턴스에서 발생하는 네트워크 트래픽을 어디로 보낼지 나타내는 이정표
- **Internet gateway**: VPC 리소스와 인터넷을 연결하는 역할을 하며 VPC당 1개만 존재
- **NAT gateway**: Private subnet에서 발생하는 네트워크 요청을 받은 후 Internet gateway에 전달하여 인터넷 통신을 할 수 있게 함. 아웃바운드 트래픽만 담당하며 인바운드 트래픽은 허용하지 않음<br />
즉, NAT gateway를 통해 Private subnet에서 Internet으로 트래픽을 내보낼 수 있지만, 반대로 Internet에서 Private subnet으로 접근할 수 없음 

<img src="/assets/img/capture/gitlab-changelog-16.png" alt="AWS Lambda 인프라 구성도" /> <br />
위 구성도에서 주목할 점은 Lambda 함수가 VPC 내부 Private subnet에 위치한 점과 Private subnet 내부 아웃바운드 트래픽이 NAT gateway로 중개되는 것을 주목하자!

위 구성도가 어떻게 동작하는지 살펴보면

1. Lambda 함수에서 아웃바운드 트래픽(=GitLab API 호출)이 발생하면 Router가 Private subnet routing table을 참조해 NAT gateway로 트래픽을 중개한다.
2. NAT gateway는 Private subnet 트래픽의 인스턴스 IP(=Lambda)를 자신에게 할당된 IP로 변경한다.
3. Router가 Public subnet routing table을 참조해 Internet gateway로 트래픽을 내보낸다.

> **0.0.0.0/0?** <br />
> 모든 IP를 의미하지만, 위의 경우 Routing table에 정의되지 않은 모든 트래픽을 의미한다.
{: .prompt-tip }

## 2. gitlab-ci.yml 변경

### 변경 전
```yml
stages:
  - generate_changelog
  - build
  - deploy

generate_changelog:
  stage: generate_changelog
  tags:
    - kubu-autoscale-shared-runner-linux
  image: node:14
  script:
    - npm install axios
    # 프로젝트 저장소 내부 자바스크립트 실행
    - node script/generate_changelog.js $PROJECT_ID $GITLAB_API_TOKEN $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event' && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^release.*$/  && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == 'master'
```

 기존에 `$CI_PIPELINE_SOURCE` 환경 변수값이 `merge_request_event`일 때, 즉 MR 이벤트가 발생할 때 CHANGELOG.md 업데이트 Job을 실행한다.

### 변경 후
```yml
stages:
  - docker-build
  - update-changelog

update-changelog:
  stage: update-changelog
  # 중략...
  before_script:
    - VERSION=$(cat package.json | grep version | head -1 | awk -F= "{ print $2 }" | sed 's/[version:,\",]//g' | tr -d '[[:space:]]')
    - BRANCH_NAME=release/v$VERSION
  script:
    # AWS Lambda 함수 호출
    - >
      curl -X POST https://kr0uynqcl3.execute-api.ap-northeast-2.amazonaws.com/changelog
      -H "Content-Type: application/json"
      -d '{
          "projectID": "'$CI_PROJECT_ID'",
          "apiToken": "'$GITLAB_API_TOKEN'",
          "branchName": "'$BRANCH_NAME'",
          "version": "'$VERSION'"
         }'
  when: always
  only:
    - master
```
`stages`를 수정해 `build` Job이 완료된 후 실행하도록 변경했다.


위의 1, 2번을 통해 CHANGELOG.md를 업데이트하는 자바스크립트를 저장소에서 분리해 독립된 환경에서 실행할 수 있고, CHANGELOG.md 업데이트 커밋이 중복되는 이슈를 해결할 수 있다.

