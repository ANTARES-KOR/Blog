# Git Server

일반적으로 원격 repository는 `bare repository`(작업중인 폴더가 없는 깃 레포지토리) 이다.
Collaboration Point로만 사용하기 때문에 Snapshot을 가지고 있을 필요가 없고, 단순히 Git 데이터일 뿐이다.
따라서 프로젝트의 `.git` 폴더 내의 내용만 담고 있다고 생각하면 된다.

## 프로토콜

깃은 데이터 전송을 위해 Local, HTTP, SSH, Git 이렇게 4가지 프로토콜을 사용할 수 있다.

### 1. Local Protocol

remote repository가 같은 host 내에 존재하는 경우 사용한다.
예를 들어 NFS와 같은 공유 파일시스템 같은 데에서 쓴다.

### 2. HTTP 프로토콜

두개의 다른 노드 간 HTTP로 통신이 가능하다. HTTP 프로토콜은 Git 1.6.6 버전 이전과 이후로 나뉘는데, 이전 버전을 `Dumb HTTP` 이후 버전을 `Smart HTTP`라고 부른다.

#### Smart HTTP - 아마 이걸 사용해야 할 것 같다

SSH나 Git Protocol과 비슷하게 동작하지만, 표준 HTTPS 포트를 사용하고, HTTP 인증 방식들을 사용할 수 있다.
그래서 SSH보다 편하게 사용할 수 있다. (SSH key 대신에 id/pw 인증을 사용할 수 있다)
현존하는 방식 중에 가장 인기가 많다. 그리고 git push/pull 용 URL과 저장소에 접근하는 URL을 동일하게 가져갈 수 있다는 장점이 있다.
예를 들어 Github을 보면 레포지토리를 online에서 조회할 때 사용하는 URL과 Clone할 때 이용하는 링크가 동일하다.

**장점**

- 모든 access에 대해서 single URL로 처리할 수 있다
- Authentication 메서드를 ID/PW 방식으로 처리할 수 있다.

**단점**

- 설정하기가 좀 어렵다 만 그래도 다른것보다 좋다...ㅋㅋ

### 3. SSH 프로토콜

일반적으로 대부분의 서버가 SSH 설정이 되어 있기 때문에 많이 쓰인다. 하지만 유저 입장에서 SSH key가 있어야 레포지토리에 접근할 수 있다는 단점이 있다.

### 4. Git 프로토콜

Git 자체에서 제공하는 데몬이다. 9418번 포트를 사용한다. SSH와 비슷하지만 Authentication이 없다.

## 서버에 깃 설치하기

Git Server를 처음으로 세팅하려면 일단 작업하고 있는 디렉토리를 새로운 bare directory로 export 해야한다.
이는 `--bare` 옵션을 이용해서 구현할 수 있다.

```bash
git clone --bare my_project my_project.git
```

### Bare repository를 서버에 올리기

이제 해야 할 일은 만들어놓은 bare repository를 서버에 올리고 프로토콜을 설정하는 일이다.
예를 들어서 서버를 `git.example.com` 에다가 올렸고, git repository들을 전부 `/srv/git` 에다가 저장하고 싶다면
`$ scp -r my_project.git user@git.example.com:/srv/git` 으로 bare repository를 복사하면 된다.

여기에 SSH 읽기 권한이 있는 유저라면
`git clone user@git.example.com:/srv/git/my_project.git/` 을 통해 클론이 가능하다.

이 폴더에 그룹 접근 권한을 주고 싶다면 해당 bare repository 폴더로 이동하여 `git init --bare --shared` 명령어를 사용하면 된다.
이거 사용한다고 commit, ref ... 등이 사라지지 않으니 안심하고 사용해도 된다.

> 이후 문서에서는 SSH, git Daemon 세팅 방법을 알려주는데, 우리는 HTTP 방식을 사용해야 하므로 정리 안했음

## 깃 서버에 SMART HTTP 설정하기

Smart HTTP 설정은 Git과 함께 제공되는 `git-http-backend`라고 하는 CGI 스크립트를 활성화하는 것이다.
이 CGI는 `git fetch` 나 `git push`를 통해서 HTTP URL로 전송되는 요청에서 path와 header를 읽는다.
이때 클라이언트의 깃 버전을 체크하고, Smart HTTP로 통신할 것인지, Dumb HTTP로 통신할 것인지 결정한다.

> CGI: Common Gateway Interface.
> 서버 프로그램과 외부 프로그램이 어떻게 연계할 지 정의되어 있는 인터페이스.

핵심은 CGI를 지원하는 웹 서버에서, `git-http-backend`를 활성화하면, 이 스크립트가 HTTP요청을 처리해준다는 거임.

<https://git-scm.com/docs/git-http-backend>

## 참고한 문서

<https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols>
