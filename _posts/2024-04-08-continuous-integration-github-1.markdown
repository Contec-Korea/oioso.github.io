---
layout: post
title: 응소팀 CI 첫걸음
author: liz
---

어디에서 필요해요?
===
> 자바기반의 웹 어플리케이션을 주로 개발하고 있는 응소팀은 프론트/백앤드 개발자들이 개발한 소스를 커밋하면 자동으로 Test > Build 과정을 실행하고 싶었습니다.
> CI 적용 전에는 로컬에서 테스트와 빌드 절차를 진행한 후 문제가 없으면 배포를 하는 프로세스였습니다.

무슨 문제가 있었어요?
===
> On-demand 기반이긴 하나, 다행히 Bitbucket을 이용한 Git을 통한 소스관리가 되어 있었습니다.
> 이제 CI를 자동으로 하기 위한 솔루션이나 서비스를 무엇을 사용할 지 리서치를 하게 됩니다.
> 후보군으로는 Jetbrain Space와 Github였습니다. 당시 Space 선보인지 얼마 안되어서 레퍼런스도 많지 않았고 
> 무엇보다도 속도가 너무 느려서 Github쪽으로 선호했습니다.

대안이 있었어요?
===
> 아직 많은 서비스를 운영하는 것이 아니었기 때문에 배포관리자들은 CI/CD의 중요성을 모르는 상태였고,
> 회사의 규모도 큰 편이 아니어서 적극적으로 지원하지는 않는 상황이었습니다. 열정있는 사람이 있으면 해봐라 하는 정도였죠.
> 그래서, 대안으로 생각한 것은 현재 상태대로 그대로 가느냐 아니면 Github를 활용해서 프로세스를 정립해 보느냐 두가지 선택지가 있었습니다.
> 
> 운이 좋았는지 Github Free버전을 사용하고 있었으나, 우리팀은 Team 버전을 사용하는 예산을 확보하게 되었습니다.
> * Free 버전의 경우 Packages storage 한도 500MB, CI/CD 실행시간 2,000분/월
> * Team 버전의 경우 Packages storage 한도 2GB, CI/CD 실행시간 3,000분/월 
> 
> 더이상 고민할 이유가 없게 되었습니다.

Github Actions에 대하여..
===
> 소프트웨어 Workflow를 자동화할 수 있도록 도와주는 도구입니다. 간단히 설명하면 아래 순서로 CI 자동화를 구현하였습니다.
> 1. 개발자들은 브랜치를 develop 브랜치에 pull request해서 머지가 되는 순간(Event)에 Maven build 라이프사이클을 실행한다. 
> 2. 빌드 라이프사이클에 Test단계가 포함되어 있고 만약 테스트는 로컬에서 실행한다면 이 단계는 건너뛸 수도 있다.
> 3. 빌드가 성공하면 필요한 파일들, e.g) jar/yml/shell script 등을 Zip 압축하여 S3에 업로드시킨다.
> 4. 또한, 해당 스텝의 성공/실패 결과를 Slack을 통해 알려준다.

Pre-requisition
===
> 1. Github Personal access tokens를 생성해 둔다.
> 2. AWS S3를 이용한다면, 미리 설정이 필요하다. 버킷생성과 사용할 Folder 생성하고, Access key, Secret key등을 Github Secret에 등록해 둔다.
> 3. Slack을 이용한다면, Slack WebHook URL도 Github Secret에 등록해 둔다.

Details
===
> 아래 action 파일은 github repository/.github/workflows/ 위치해 있습니다.
> 

<pre>
<code>
name: Build and Slack

# develop 브랜치에 pull request한 변경사항이 Merged 되는 시점에 해당 workflow가 실행된다.
on:
  pull_request:
    branches: [ develop ]
    types: [ "closed" ]

# 두개의 Job을 생성 : 빌드, 슬랙알림
jobs:
  build:
    if: github.event.pull_request.merged == true
    # OS 선택 가능, linux, window, macos
    runs-on: ubuntu-latest

    # GITHUB_TOKEN 권한
    permissions:
      contents: read

    steps:
      # 환경설정, JDK 버전 설정, Maven 사용예고, 소스 checkout
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      # Maven 테스트 후 Jar 빌드
      - name: Build and Test with Maven
        run: mvn -B package --file ./pom.xml
        env:
          GITHUB_TOKEN: ${{ secrets.ACTION_PACKAGES_TOKEN }}

      # 압축
      - name: Make zip file
        run: zip test-api.zip target/*.jar start.sh appspec.yml

      # AWS 인증
      - name: AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
          aws-region: ap-northeast-2

      # S3에 빌드된 파일 업로드
      - name: Upload to AWS S3
        run: |
          aws s3 cp --region ap-northeast-2 test-api.zip s3://deploy-test/snapshot/
          
  # Slack 알림 관련 Job
  # 더 자세한 내용을 다른 포스트로 설명하겠습니다.
  slack-notification:
    if: ${{ always() }}
    runs-on: ubuntu-latest
    needs: [ build ]

</code>
</pre>

Pros and Cons
===

> 장점
> * git을 이용한 소스관리를 하고 있다면 Workflow 생성 로직을 이용해 간단한 Continuous Integration을 구현할 수 있다.
> * 다양한 템플릿을 제공하여 쉽게 구현 가능하다.
> * 누군가 만들어둔 action을 활용할 수 있다.
> * Secrets에 등록된 암호화된 키를 사용하고 있어 안전하다.
> * Github 서비스인 Packages를 활용하여 artifact를 upload/download 용이하다.
> 
> 단점
> * Github Action Core 개념 이해가 필요하다. Workflow, Event, Job, Step, Action, Runner등이 있다. 
> * 항상 Event를 발생시켜야 하기 때문에 Action 생성 후 테스트가 힘들다.
> * 로그를 실행시 실시간으로 화면에 제공하긴 하나 계속 올라가기 때문에 보기가 힘들다. 실행이 끝난 후 처음부터 따라가면서 살펴보아야 한다.