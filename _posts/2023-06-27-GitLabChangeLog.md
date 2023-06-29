---
title: GitLab 배포사항 자동 기록하기
date: 2023-06-27 18:30:00 +09:00
categories: [Deveploment, GitLab]
tags: [gitlab, ci, changelog] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 개요
팀에서 커피챗 도중 배포 사항을 수동으로 기록하는 것이 불편해 자동으로 기록하는 것이 있으면 좋을 것 같다는 의견이 있어 알아보게 되었다. 

**들어가기전에!** <br />
우리 팀의 브랜치 전략과 커밋 컨벤션 기준으로 작성된 문서이므로 이 부분은 인지하고 읽으면 좋을 것 같다!

# GitLab API 활용하기
GitLab API를 활용하면, 원격 저장소의 커밋 이력을 가져오거나, 커밋을 하는 등의 일을 할 수 있다. GitLab API 중 CHANGELOG.md 파일을 업데이트하는 API가 있었으나, 사내 GitLab 버전이 낮아 사용할 수 없었다. ㅠ-ㅠ 아래 표시한 API 목록이 이번 기능을 개발하기 위해 필요했다.

- **태그 리스트 조회 API**
- **커밋 리스트 조회 API**
- **파일 조회 API**
- **커밋 생성 API**

각 API 명세는 [여기](https://docs.gitlab.com/ee/api/rest/){:target="_blank"}서 확인할 수 있다.

# GitLab CI/CD
GitLab CI/CD에서 Job을 정의하고 특정 조건에 해당 Job을 실행할 수 있다. 아래 코드를 살펴보자!
```yml
image: registry-gitlab.nexon.com/webuidev/ops-scripts:latest
stages:
  - generate_changelog

generate_changelog:
  stage: generate_changelog
  tags:
    - autoscale-shared-runner-linux
  image: public.ecr.aws/bitnami/node:14
  script:
    - npm install axios
    - node script/generate_changelog.js $PROJECT_ID $GITLAB_API_TOKEN $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event' && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^release.*$/  && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == 'master'
```

### 코드 설명
- **`generate_changelog:`**<br />Job을 정의

- **`node script/generate_changelog.js $PROJECT_ID $GITLAB_API_TOKEN $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`**<br />Job에서 실행할 스크립트

- **`if: $CI_PIPELINE_SOURCE == 'merge_request_event' && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^release.*$/ && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == 'master'`**<br />MR을 등록했을 때 Source Branch 이름이 release로 시작하고 Target Branch 이름이 master일 때만 해당 Job을 실행

### 변수 설명
- **`$PROJECT_ID:`**<br />GitLab Repository Project ID, Project Overview 메뉴에서 확인할 수 있다. <br />
<img src="/assets/img/capture/gitlab-changelog-1.png" alt="project id" style="width:250px;" />

- **`$GITLAB_API_TOKEN:`**<br />GitLab API 호출 시 필요한 토큰으로 Setting → Access Token 메뉴에서 발급받을 수 있다.
<img src="/assets/img/capture/gitlab-changelog-2.png" alt="api token" style="width:500px;" />


- **`$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME:`**<br />사전에 정의된 변수로 MR Source Branch 이름을 저장하고 있다.Source Branch 이름은 release/vN.N.N 형태로 구성되어 있고, 버전을 추출하기 위해 스크립트 파라미터로 전달했다.

- **`$CI_PIPELINE_SOURCE:`**<br />사전에 정의된 변수로 파이프라인이 트리거 된 원인을 저장하고 있다.

`$PROJECT_ID, $GITLAB_API_TOKEN은` GitLab Setting → CI/CD Variable 섹션에서 정의했다. (아래 이미지 참고) <br />
<img src="/assets/img/capture/gitlab-changelog-3.png" alt="gitlab variable" />

GitLab CI에서 사전에 정의된 변수 목록은 [여기](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)서 확인하자!

# generate_changelog.js 살펴보기
```javascript
const axios = require("axios");

// 상수 정의
const PROJECT_ID = process.argv[2];
const GITLAB_API_TOKEN = process.argv[3];
const BRANCH_NAME = process.argv[4];
const BASE_URL = `https://gitlab.nexon.com/api/v4/projects/${PROJECT_ID}`;
const VERSION = BRANCH_NAME.split("/")[1];
const FILE_PATH = `changelogs/${VERSION}.md`;
const ESCAPED_FILE_PATH = `changelogs%2F${VERSION}.md`;

const api = axios.create({
  baseURL: BASE_URL,
  headers: { "PRIVATE-TOKEN": GITLAB_API_TOKEN },
});

const regexVersion = /^v(\d+\.)?(\d+\.)?(\*|\d+)$/;
const regexCommit =
  /^(feat|fix|docs|style|refactor|perf|build|ci|chore|revert).*$/;
const regexFeature = /^feat.*$/;
const regexFix = /^fix.*$/;
const regexStyle = /^style.*$/;
const regexRefactor = /^refactor.*$/;

// 메인 함수
(async function () {
  try {
    // 최근에 배포한 태그를 가져온다. (e.g. v1.0.0)
    const latestTag = await fetchLatestTag(); 

    // 최근 배포한 태그의 태깅된 커밋 일시를 가져온다.
    const sinceDate = latestTag ? latestTag.commit.created_at : undefined; 

    // MR의 Source Branch인 release 브랜치에서 sinceDate 이후에 커밋된 이력을 가져온다. 
    const commits = await fetchCommits(sinceDate); 

    // 커밋 컨벤션이 적용된 커밋들만 필터링한다.
    const filteredCommits = commits.filter((commit) =>
      regexCommit.test(commit.title) 
    );

    // 커밋들을 기록하기 좋게 분류한다.
    const classifiedCommits = classifyCommits(filteredCommits); 

    // 분류된 커밋을 기반으로 마크다운 형식의 컨텐츠를 생성한다.
    const changeLogContents = createChangeLogContent(classifiedCommits); 
    
    // 해당 버전의 파일이 이미 존재하는지 확인하는 변수
    const isExist = await hasAleadyChangeLogFile(); 
    const postChangeLogActions = isExist
      ? [
          {
            action: "delete",
            file_path: FILE_PATH,
          },
          {
            action: "create",
            file_path: FILE_PATH,
            content: changeLogContents,
          },
        ]
      : [
          {
            action: "create",
            file_path: FILE_PATH,
            content: changeLogContents,
          },
        ];
    // 파일을 커밋한다.
    await postChangeLog(changeLogContents, postChangeLogActions); 
  } catch (error) {
    console.error(error);
  }
})();

// 가장 최근에 태깅한 버전 태그 가져오기, 태그명은 vX.X.X 규칙을 가지고 있어야함
async function fetchLatestTag() {
  const res = await api.get("/repository/tags");
  const versionTags = res.data.filter((tag) => regexVersion.test(tag.name));
  return versionTags[0];
}

// relase/vX.X.X에서 최신 버전 태그 커밋의 커밋일 이후 모든 커밋이력 가져오기 (= 아직 배포하지 않은 커밋 이력들)
async function fetchCommits(since) {
  const res = await api.get(`/repository/commits?ref_name=${BRANCH_NAME}`, {
    params: { since },
  });
  return res.data;
}

// 원격 저장소에 이미 해당 버전의 change log 파일이 있는 지 여부 가져오기
async function hasAleadyChangeLogFile() {
  try {
    await api.get(`/repository/files/${ESCAPED_FILE_PATH}`, {
      params: { ref: "master" },
    });
    return true;
  } catch (error) {
    if (error.response.status === 404) {
      return false;
    } else {
      throw error;
    }
  }
}

// 원격 저장소에 해당 버전의 change log 파일 생성
async function postChangeLog(changeLogContents, actions) {
  await api.post("/repository/commits", {
    branch: "master",
    commit_message: `docs: ${VERSION} change logs 생성`,
    actions,
  });
}

// 커밋 type 별 분류하기
function classifyCommits(commits) {
  const result = {
    feat: [],
    fix: [],
    style: [],
    refactor: [],
    etc: [],
  };

  for (const commit of commits) {
    if (regexFeature.test(commit.title)) {
      result.feat.push(commit);
    } else if (regexFix.test(commit.title)) {
      result.fix.push(commit);
    } else if (regexStyle.test(commit.title)) {
      result.style.push(commit);
    } else if (regexRefactor.test(commit.title)) {
      result.refactor.push(commit);
    } else {
      result.etc.push(commit);
    }
  }

  return result;
}

// chage log 파일 콘텐츠 생성하기, 마크다운 형식
function createChangeLogContent(classifiedCommits) {
  let content = `# Change Log\nAll notable changes to this project will be documented in this file.\n\n`;
  content = content.concat(`## ${VERSION}\n\n`);

  const prefixKeys = Object.keys(classifiedCommits);

  for (const prefixKey of prefixKeys) {
    const commits = classifiedCommits[prefixKey];
    if (commits.length === 0) {
      continue;
    }

    content = content.concat(`### ${prefixKey}\n`);
    for (const commit of commits) {
      content = content.concat(`- [${commit.title}](${commit.web_url})\n`);
    }
    content = content.concat("\n");
  }

  return content;
}
```
위 코드를 요약하면,
1. **가장 최근에 배포된 버전의 태그를 가져오고, 해당 태그가 태깅된 커밋 일시를 가져온다.**
2. **MR Source Branch인 release 브랜치에서 위 일시 이후에 커밋된 이력을 가져온다. (배포 내역 가져오는 작업)**
3. **커밋 이력 중 커밋 컨벤션 지키는 커밋만 필터링하고 분류한다.**
4. **위 커밋 이력을 기반으로 파일 콘텐츠를 생성하고 커밋한다.**

# 실습

### release/v1.0.0 커밋 내역 확인 (대부분 지라 티켓 단위로 개발사항이 축적)
<img src="/assets/img/capture/gitlab-changelog-4.png" alt="release branch commits" />

### release/v1.0.0 → master MR 등록 (배포를 위한 MR 등록)
<img src="/assets/img/capture/gitlab-changelog-5.png" alt="MR" style="width:500px" />

### MR 등록 시 Gitlab Job 자동 실행
<img src="/assets/img/capture/gitlab-changelog-6.png" alt="run job" />

### 배포사항을 기록한 파일 확인
<img src="/assets/img/capture/gitlab-changelog-7.png" alt="check file" />

### MR Merge 완료 후 배포 버전 태그 생성
<img src="/assets/img/capture/gitlab-changelog-8.png" alt="tagging" /> <br />
배포 시 버전 태그를 기록한다. 다음 버전의 커밋 이력을 가져오기 위한 기준이 된다.

### 한 사이클이 끝났다. 다음 버전(v1.0.1) 배포도 진행해보자!

### release/v1.0.1 커밋 내역 확인 (대부분 지라 티켓 단위로 개발사항이 축적)
<img src="/assets/img/capture/gitlab-changelog-9.png" alt="release branch commits" /> <br />
표시한 부분의 내역만 파일로 잘 기록되는지 확인해보자!

### 배포사항을 기록한 파일 확인
<img src="/assets/img/capture/gitlab-changelog-10.png" alt="check file" /> <br />
음! 잘 기록되었다!

### 두 번의 배포가 진행된 후 master 브랜치 커밋 트리
<img src="/assets/img/capture/gitlab-changelog-11.png" alt="master commit tree" /> <br />

<br />
<br />
<br />
<hr />

# 개선사항
**위에서는 배포 버전 별로 마크다운 파일이 생성되는데 하나의 CHNAGELOG.md 파일에 버전 별로 내용이 기록되도록 하는게 좋을 것 같다. (그 이유는 검색이 편해서!?)**
```javascript
const axios = require("axios");

// 상수 정의
const PROJECT_ID = process.argv[2];
const GITLAB_API_TOKEN = process.argv[3];
const BRANCH_NAME = process.argv[4];
const BASE_URL = `https://gitlab.nexon.com/api/v4/projects/${PROJECT_ID}`;
const VERSION = BRANCH_NAME.split("/")[1];
const VERSION_ARR = BRANCH_NAME.split("/v")[1].split(".");
const FILE_PATH = `CHANGELOG.md`;

const api = axios.create({
  baseURL: BASE_URL,
  headers: { "PRIVATE-TOKEN": GITLAB_API_TOKEN },
});

const regexVersion = /^v(\d+\.)?(\d+\.)?(\*|\d+)$/;
const regexCommit =
  /^(feat|fix|docs|style|refactor|perf|build|ci|chore|revert).*$/;
const regexFeature = /^feat.*$/;
const regexFix = /^fix.*$/;
const regexStyle = /^style.*$/;
const regexRefactor = /^refactor.*$/;

// 메인 함수
(async function () {
  try {
    // 최근에 배포한 태그를 가져온다. (e.g. v1.0.0)
    const latestTag = await fetchLatestTag();

    // 최근 배포한 태그의 태깅된 커밋 일시를 가져온다.
    const sinceDate = latestTag ? latestTag.commit.created_at : undefined; 

    // MR의 Source Branch인 release 브랜치에서 sinceDate 이후에 커밋된 이력을 가져온다.
    const commits = await fetchCommits(sinceDate);

    // 커밋 컨벤션이 적용된 커밋들만 필터링한다.
    const filteredCommits = commits.filter(
      (commit) => regexCommit.test(commit.title) 
    );

    // 커밋들을 기록하기 좋게 분류한다.
    const classifiedCommits = classifyCommits(filteredCommits);

    // 분류된 커밋을 기반으로 마크다운 형식의 컨텐츠를 생성한다.
    const changeLogContent = makeChangeLogContent(classifiedCommits); 

    // 원격 저장소에 CHANGELOG.md 파일을 조회한다.
    const changeLogFile = await fetchChangeLog();

    // 파일이 없으면 파일을 생성하기 위한 플래그 변수
    const isCreateFile = !changeLogFile; 

    const changeLogContents = isCreateFile
      ? createChangeLogContent(changeLogContent)
      : updateChangeLogContent(changeLogContent, changeLogFile);

    const postChangeLogActions = isCreateFile
      ? [
          {
            action: "create",
            file_path: FILE_PATH,
            content: changeLogContents,
          },
        ]
      : [
          {
            action: "delete",
            file_path: FILE_PATH,
          },
          {
            action: "create",
            file_path: FILE_PATH,
            content: changeLogContents,
          },
        ];

    // 파일을 커밋한다.
    await postChangeLog(postChangeLogActions); 
  } catch (error) {
    console.error(error);
  }
})();

// 가장 최근에 태깅한 버전 태그 가져오기, 태그명은 vX.X.X 규칙을 가지고 있어야함
async function fetchLatestTag() { /* 전과 동일 */ }

// relase/vX.X.X에서 최신 버전 태그 커밋의 커밋일 이후 모든 커밋이력 가져오기 (= 아직 배포하지 않은 커밋 이력들)
async function fetchCommits(since) { /* 전과 동일 */ }

// 원격 저장소에 CHANGELOG.md 파일 가져오기 (API가 변경됨)
async function fetchChangeLog() {
  try {
    const res = await api.get(`/repository/files/${FILE_PATH}/raw`, {
      params: { ref: "master" },
    });
    return res.data;
  } catch (error) {
    if (error.response.status === 404) {
      return null;
    } else {
      throw error;
    }
  }
}

// 원격 저장소에 해당 버전의 change log 파일 생성
async function postChangeLog(actions) { /* 전과 동일 */ }

// 커밋 type 별 분류하기
function classifyCommits(commits) { /* 전과 동일 */ }

// chagelog 콘텐츠 생성하기
function makeChangeLogContent(classifiedCommits) {
  let content = `## [${VERSION}]\n\n`;
  const prefixKeys = Object.keys(classifiedCommits);

  for (const prefixKey of prefixKeys) {
    const commits = classifiedCommits[prefixKey];
    if (commits.length === 0) {
      continue;
    }

    content = content.concat(`### ${prefixKey}\n`);
    for (const commit of commits) {
      content = content.concat(`- [${commit.title}](${commit.web_url})\n`);
    }
    content = content.concat(`\n\n`);
  }

  return content;
}

// chage log 파일 콘텐츠 생성하기, 마크다운 형식
function createChangeLogContent(changeLogContent) {
  return `# Change Log\n\n${changeLogContent}`;
}

// chage log 파일 콘텐츠 수정하기, 마크다운 형식
function updateChangeLogContent(changeLogContent, changeLogFile) {
  let result;
  const [major, minor, patch] = VERSION_ARR;
  const versionRegex = new RegExp(`## \\[v${major}\\.${minor}\\.${patch}\\]`);
  const isExistVersion = versionRegex.test(changeLogFile);

  if (!isExistVersion) {
    const regex = new RegExp(`# Change Log\n\n`);
    result = changeLogFile.replace(
      regex,
      `# Change Log\n\n${changeLogContent}`
    );
  } else {
    /**
     * ## [vX.X.X] 를 시작으로
     * 개행이 연속적으로 세번 나오는 구간까지 정규식을 통해 검색한다.
     * 검색된 구간을 새롭게 작성한 배포 내역으로 대체한다.
     */
    const regex = new RegExp(
      `## \\[v${major}\\.${minor}\\.${patch}\\]((.|\\n)*?)(\\n\\n\\n)`
    );
    result = changeLogFile.replace(regex, changeLogContent);
  }

  return result;
}
```
<img src="/assets/img/capture/gitlab-changelog-7.png" alt="check file" />

음! 이 방법이 더 좋은 것 같다!

<br />
<br />
<br />
<hr />

# 참고사항
- 배포 후 태깅을 잘하자!
- 배포 내역을 기록한 파일을 굳이 추적할 필요가 없다면 .gitignore에 등록하자!
