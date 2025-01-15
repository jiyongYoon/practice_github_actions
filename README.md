# practice_github_actions
github actions 연습 repo

---

## 세팅 환경

- 홈 네트워크 게이트 웨이(iptime 공유기를 라우터로 사용중) -> mini pc (server)

## 연결 방법

### 1. 비밀번호 사용방법
```
ssh {username}@{host} -p {port}
password: {입력}
```
- 평문으로 보내기 때문에 권장하지 않음

### 2. ssh key

- 서버에서 비대칭키를 생성하여 개인키를 github actions의 secret에 등록하여 사용

## 세팅 방법

### 1. 포트포워드 설정
- ssh 기본 포트는 22
### 2. 서버 ssh 키 생성 및 등록
1. 키 생성
    ```
    ssh-keygen -t rsa
    ```
  - 생성 위치: `~/.ssh/`
  - 생성 파일
    ```
    $ ls
    authorized_keys(서버에 공개키 등록하는 파일)  id_rsa(개인키)  id_rsa.pub(공개키)
    ```

2. 개인 키 내용 확인
    ```
    cat id_rsa
    ```
3. github actions의 secret에 추가 -> BEGIN ~ END 라인까지 다
4. 서버에 공개키 등록
   ```
   cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
   ```
5. 권한 변경
   ```
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

## 트러블 슈팅

### 1. 인바운드 규칙

자꾸 연결 시도 시 timeout이 났다.
```
Run appleboy/ssh-action@master
Run echo "$GITHUB_ACTION_PATH" >> $GITHUB_PATH
Run entrypoint.sh
Will download drone-ssh-1.8.0-linux-amd64 from https://github.com/appleboy/drone-ssh/releases/download/v1.8.0
======= CLI Version =======
Drone SSH version 1.8.0
===========================
2025/01/15 14:40:55 dial tcp ***:22: i/o timeout
Error: Process completed with exit code 1.
```

iptime에서는 인바운드 규칙을 [고급설정] - [보안기능] - [인터넷 사용제한] 에서 세팅할 수 있다.
내부 목적지는 서버로 사용하는 mini-pc의 내부 ip와 ssh port로 세팅했는데, 문제는 외부 ip였다.
전부 다 오픈하기에는 부담스러워서, github actions 동작을 하는 ip range를 찾아보려고 했다.

참고 1. [github actions ip range](https://docs.github.com/en/actions/using-github-hosted-runners/using-github-hosted-runners/about-github-hosted-runners#ip-addresses)
참고 2. [Get GitHub meta information](https://docs.github.com/en/rest/meta/meta?apiVersion=2022-11-28#get-github-meta-information)

요약하면, `meta information API`를 사용해서 정보를 가져가라는 것이다.

사용할 명령어는 아래와 같다.
```
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <YOUR-TOKEN>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/meta
```

토큰 세팅을 위해 [github 계정 프로필] - [좌측 사이드바: Developer settings] - [Personal access tokens] 에서 classic token을 생성했다.
meta API는 특별한 권한은 필요 없어서 권한 추가 없이 생성해서 사용했다.

웬걸, meta API에서 정보를 얻었는데, github actions에서 사용하는 ip가,,, 족히 200개는 넘는 것 같았다.
다 등록할 수 없었다...

다른 방법을 알아보니, 매 actions 동작마다 [ip 확인] -> [인바운드 규칙 추가] -> [CI/CD] -> [인바운드 규칙 삭제] 동작을 하게 actions script를 구성하는 방법도 있었다. [참고 블로그](https://makethree.tistory.com/19)

이런 레퍼런스들 모두 `aws`나 `gcp` 등에 배포할 때 cli를 사용하는 방법이었다...

로컬 서버에서는 해당 방법이 쉽지 않아서 일단 모든 포트를 여는 방법으로 해야 될 것 같다.
다만, 조금이라도 안전하게 하기위해 iptime에서 제공하는 국가별 접속 제한을 걸어두었다. 한국과 미국에서만 접속하도록 해두었다. github actions는 microsoft Azure 에서 동작한다는 사실을 누군가 파헤쳐두었기 때문이다. Azure에서 컨테이너로 동작한다고 한다. [참고 블로그](https://this-programmer.tistory.com/498)

세팅을 다 마치니 연결 및 동작에 성공했다.
```
Run appleboy/ssh-action@master
Run echo "$GITHUB_ACTION_PATH" >> $GITHUB_PATH
Run entrypoint.sh
Will download drone-ssh-1.8.0-linux-amd64 from https://github.com/appleboy/drone-ssh/releases/download/v1.8.0
======= CLI Version =======
Drone SSH version 1.8.0
===========================
===============================================
✅ Successfully executed commands to all hosts.
===============================================
```
