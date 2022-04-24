# Git Internals

## Plumbing and Porcelain

현재 git은 `checkout`, `branch`, `remote` 와 같이 편한 명령어를 제공하지만, git은 원래 low-level command를 제공했다.
이거를 "plumbing" 명령어라고 하고, 유저 친화적인 명령어를 "porcelain" 명령어라고 한다.

`git init`을 실행하게 되면 git은 `.git` 폴더를 생성한다. 여기에 git 관련한 모든 파일이 저장된다.
레포지토리를 백업이나 복사하고 싶다면, **이 폴더만 복사해가면 된다**

`.git` 폴더를 처음 생성하면 보통 이렇게 생겼다.

```text
config : 프로젝트에 관한 설정이 들어있는 파일
description : GitWeb 프로그램에서만 사용하는 파일
HEAD : 현재 작업중인 브랜치를 가리킴
index : staging area 정보를 저장함.
hooks/ : client나 server side git hook 스크립트들이 저장되어 있다.
info/ : .gitignore 파일에서 정한 패턴을 저장하기 위한 exclude 파일이 저장되어 있음.
objects/ : 데이터베이스에 대한 모든 컨텐츠가 저장되어 있는 폴더.
refs/ : 커밋 오브젝트에 대한 포인터가 저장되어 있음 (branch, remote, tag...)
```

## Git Objects

깃은 content-addressable 파일시스템이다.
깃의 핵심은 key-value 데이터 저정소로, 어떤 자료를 깃 레포지토리에 저장하던지 간에 깃은 그 자료에 접근할 수 있는 unique 키를 반환한다.

`git init` 직후에 `objects/` 디렉토리를 보면 아직 일반 파일이 없는 상태다.

```text
.git/objects
.git/objects/info
.git/objects/pack
```

이 상태에서 `git hash-object` 를 통해 데이터 오브젝트를 만들어서 깃 데이터베이스에 저장해보자.

```bash
$ echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

- `-w` : 해당 오브젝트를 깃 데이터베이스에 저장하도록 하는 옵션

위 명령어의 결과로 반환된 키값이 깃 DB에 저장된 컨텐츠의 키값이다. 해당 키값은 40자의 SHA-1 체크섬 해쉬이다.
이 상태에서 `object` 폴더 내부를 보면 키값의 앞 2글자를 딴 폴더 내부에 나머지 38글자를 파일명으로 하는 파일이 생성되어 있는 것을 확인할 수 있다.

```bash
$ find .git/objects -type f
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

생성된 깃 파일을 다시 확인하려면 `git cat-file` 명령어를 이용한다. `-p` 옵션과 함께 사용하면 파일 타입을 확인하고 그에 맞게 출력하도록 할 수 있다.

```bash
$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
test content
```

이 상태에서 해당 파일에 대해 변경을 하고, `git hash-object` 를 실행하면 새로운 버전에 대한 정보가 깃 데이터베이스에 저장된다. (기존 파일에 대한 정보도 그대로 유지되어 있다.)

```bash
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30

$ echo 'version 2' > test.txt
$ git hash-object -w test.txt
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a

$ find .git/objects -type f
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

이렇게 저장된 깃 오브젝트의 타입을 `blob` 이라고 한다. 오브젝트 타입은 `git cat-file -t` 로 확인 가능하다.

```bash
$ git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
blob
```

### Tree Object

다음으로 알아볼 깃 오브젝트 타입은 `tree` 타입이다. 트리는 파일명을 저장하고, 여러 파일들을 그룹으로 저장하는 문제를 해결한다.

깃에서는 UNIX 파일시스템과 비슷하지만, 조금 더 단순화된 형태로 데이터를 저장한다.

모든 자료는 `tree` 나 `blob` 으로 저장된다. 여기서 `tree`가 디렉토리를, `blob`이 파일을 나타낸다고 생각하면 된다.
각각의 `tree` 오브젝트는 1개 이상의 entry를 갖는데, 이 엔트리는 blob이나 subtree의 SHA-1 해시값이다.

```bash
$ git cat-file -p master^{tree}
100644 blob a906cb2a4a904a152e80877d4088654daad0c859      README
100644 blob 8f94139338f9404f26296befa88755fc2598c289      Rakefile
040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0      lib
```

- `master^{tree}` : `master` 브랜치에서의 가장 최근 커밋이 가리키는 트리 오브젝트를 가리킴.
  위의 예시를 보면 `lib` 디렉토리는 blob이 아니라 다른 tree에 대한 포인터라는거를 알 수 있다.
  해당 해시값을 `cat-file`해보면

```bash
$ git cat-file -p 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0
100644 blob 47c6340d6459e05787f644c2447d2595f5d3a54b      simplegit.rb
```

> 어떤 쉘을 쓰느냐에 따라서 master^{tree}를 사용하는 문법이 좀 다를 수 있다.
> [링크](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) 참고

해당 트리 안의 오브젝트를 볼 수 있다.

위의 예시에서 깃이 저장하고 있는 데이터는 다음과 같다.
![깃 데이터 모델](./img/data-model-1.png)

`git update-index` : index를 생성하는 명령어 (staging 하는 명령어)
`git write-tree` : 현재 staging area를 tree object로 쓰는(?) 명령어 (현재 스테이지 영역의 상태의 스냅샷을 트리 오브젝트로 추출한다?고 생각하면 될듯)
`git read-tree` : 지정한 트리 오브젝트를 스테이징 영역에 추가할 수 있다

### Commit Object

트리로 레포지토리의 스냅샷을 저장할 수 있다는 것은 알았다.
하지만 이 스냅샷을 읽어오려면 해당 트리에 대한 SHA-1 해시값을 알고 있어야 한다는 문제가 있다.
그리고 그 스냅샷이 언제, 어디서, 누구에 의해 생성되었는지 알 수 가 없다.

이런 내용을 저장할 수 있는게 commit object라고 보면 된다.

`git commit-tree 트리의_SHA-1_해시` : 해당 트리에 대한 커밋 오브젝트를 생성하고, 해당하는 SHA-1 해시를 반환한다.

```bash
$ echo 'First commit' | git commit-tree d8329f
fdf4fc3344e67ab068f836878b6c4951e3b15f3d
```

이 커밋 오브젝트를 `git cat-file` 명령어로 확인해보면

```bash
$ git cat-file -p fdf4fc3
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579 // 해당 커밋의 파일에 대한 스냅샷
author Scott Chacon <schacon@gmail.com> 1243040974 -0700 // 작성자
committer Scott Chacon <schacon@gmail.com> 1243040974 -0700 // 커밋 넣은사람

First commit
```

과거 커밋 이력을 확인하고 싶다면 `git log` 커맨드를 사용하면 된다. 가장 최근 커밋에 대해서 이 명령어를 사용하면

```bash
$ git log --stat 1a410e
commit 1a410efbd13591db07496601ebc7a059dd55cfe9
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:15:24 2009 -0700

 Third commit

 bak/test.txt | 1 +
 1 file changed, 1 insertion(+)

commit cac0cab538b970a37ea1e769cbbde608743bc96d
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:14:29 2009 -0700

 Second commit

 new.txt  | 1 +
 test.txt | 2 +-
 2 files changed, 2 insertions(+), 1 deletion(-)

commit fdf4fc3344e67ab068f836878b6c4951e3b15f3d
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:09:34 2009 -0700

    First commit

 test.txt | 1 +
 1 file changed, 1 insertion(+)
```

이렇게 3가지의 오브젝트를 이용해서 깃이 작동한다.

이 깃 레포의 참조관계까지 그림으로 나타내보면 다음처럼 된다.

![깃 레포 현황](img/data-model-3.png)

## Git References

깃 히스토리를 조회하고 싶을때 `git log 1a410e` 같은 명령어를 실행해서 조회할 수 있다.
여기서 1a410e가 조회하고 싶은 커밋에 대한 해쉬인데, 이걸 매번 기억하는것보다는 이거를 참조하는 파일을 만들어 놓는게 편하지 않을까?

를 가능하게 하는게 git ref 이다.

`.git/refs` 디렉토리를 뜯어보면 다음과 같다.

```bash
$ find .git/refs
.git/refs
.git/refs/heads
.git/refs/tags
```

여기서 ./git/refs/heads 아래에 내가 저장하고 싶은 SHA-1 해쉬를 담은 파일을 저장해두면
해당 파일이름을 커밋 해시처럼 쓸 수 있다.

```bash
$ git update-ref refs/heads/master 1a410efbd13591db07496601ebc7a059dd55cfe9

$ git log --pretty=oneline master
1a410efbd13591db07496601ebc7a059dd55cfe9 Third commit
cac0cab538b970a37ea1e769cbbde608743bc96d Second commit
fdf4fc3344e67ab068f836878b6c4951e3b15f3d First commit
```

`git branch <branch>` 를 실행하면 깃은 이 `update-ref` 명령어를 이용해서 현재 브랜치의 마지막 커밋을 새로운 브랜치에 대한 ref로 지정해준다.

앞으로는 깃에서 사용하는 3가지 reference 종류를 알아볼 것이다.

### HEAD

그러면, `git branch <branch>` 를 실행했을 때 깃이 어떻게 현재 브랜치의 마지막 커밋이 뭔지 알 수 있는가가 관건인데, 여기서 HEAD 파일이 등장한다.

일반적으로 HEAD 파일은 현재 브랜치에 대한 symlink이다.
하지만 특정 상황에서는 HEAD파일이 git object에 대한 SHA-1 해시값을 저장하기도 한다.
그 특정 상황은 레포지토리를 "detached HEAD" 상태로 저장하는 상황인데, 태그를 생성하거나 원격 브랜치를 생성하는 상황 등이 있다.

```bash
$ cat .git/HEAD
ref: refs/heads/master

$ git checkout test # test 브랜치로 이동하는 명령어

$ cat .git/HEAD
ref: refs/heads/test # HEAD가 가리키는 위치가 바뀐걸 볼 수 있다
```

`git symbolic-ref` : HEAD가 가리키는 값을 읽을 수도 있고, 바꿀 수도 있다.

### Tags

앞에서 깃의 3가지 오브젝트 타입 (**blob**, **tree**, **commit**)에 대해 알아봤는데, 사실 하나 더 있다.

**tag** 오브젝트는 커밋이랑 비슷한데, 트리가 아니라 특정 커밋을 가리키는 오브젝트라는 점에서 다르다.
브랜치 Ref와 비슷하지만, 태그는 특정 커밋 하나만을 가리키며, 변하지 않는다.

```bash
git update-ref refs/tags/v1.0 cac0cab538b970a37ea1e769cbbde608743bc96d # lightweight tag 생성
git tag -a v1.1 1a410efbd13591db07496601ebc7a059dd55cfe9 -m 'Test tag' # annotated tag 생성
```

### Remotes

3번째 reference type이다. remote를 하나 지정해 두고 거기다가 push를 하면 깃은 `refs/remotes` 디렉토리에 해당 remote에 마지막으로 push 한 값을 저장한다.

```bash
# origin/master 브랜치에 푸시하는 예시
$ git remote add origin git@github.com:schacon/simplegit-progit.git
$ git push origin master
Counting objects: 11, done.
Compressing objects: 100% (5/5), done.
Writing objects: 100% (7/7), 716 bytes, done.
Total 7 (delta 2), reused 4 (delta 1)
To git@github.com:schacon/simplegit-progit.git
  a11bef0..ca82a6d  master -> master
```

이렇게 하고 나면 `refs/remotes/origin/master` 파일에서 마지막으로 origin 의 master 브랜치에 푸시한 내용을 확인할 수 있다.

```bash
$ cat .git/refs/remotes/origin/master
ca82a6dff817ec66f44342007202690a93763949
```

Remote ref와 branch(`refs/heads`) 간 차이는 remote ref는 읽기 전용이라는 거다.
여기로 checkout을 할 수는 있지만 커밋을 할 수는 없다.

## Packfiles

```bash
$ find .git/objects -type f
.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
.git/objects/95/85191f37f7b0fb9444f35a9bf50de191beadc2 # tag
.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1
```

깃을 사용하다 보면 한 파일에 대해서 조그만 변경사항이 있는 여러가지 버전을 저장하고 있게 된다.
각 커밋에 대해서 이런 파일을 매번 새로 저장한다고 하면 저장공간을 많이 잡아먹으므로 이거를 효율적으로 저장하기 위해서 git은 packfile이라는 걸 만든다.

packfile은 깃 오브젝트들을 하나의 바이너리로 pack한 것을 뜻한다. packfile 생성은 `git gc` 명령어를 이용해서 할 수 있다. 사실 깃이 자동으로 팩을 하는데 수동으로 해주고 싶을때 사용한다.

```bash
$ git gc
Counting objects: 18, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (14/14), done.
Writing objects: 100% (18/18), done.
Total 18 (delta 3), reused 0 (delta 0)
```

깃이 오브젝트를 pack 할때 깃은 이름이나 사이즈가 비슷한 파일들을 찾아서 해당 파일들의 delta만 저장한다.
gc를 하고 나면 기존의 object 대신에 packfile이 생성되어 있는것을 확인할 수 있다

```bash
$ find .git/objects -type f
.git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
.git/objects/info/packs
.git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.idx # index 파일 : 팩파일에 대한 offset을 저장해서 특정 오브젝트를 쉽게 접근할 수 있게 해줌
.git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.pack # packfile : 해당하는 오브젝트들의 내용을 저장
```

오브젝트로 남아있는 파일들은 어떤 커밋에도 저장되어있지 않은 파일들이다.

`git verify-pack` : 어떤 파일들이 pack되었는지 확인할 수 있는 명령어.

```bash
$ git verify-pack -v .git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.idx
2431da676938450a4d72e260db3bf7b0f587bbc1 commit 223 155 12
69bcdaff5328278ab1c0812ce0e07fa7d26a96d7 commit 214 152 167
80d02664cb23ed55b226516648c7ad5d0a3deb90 commit 214 145 319
43168a18b7613d1281e5560855a83eb8fde3d687 commit 213 146 464
092917823486a802e94d727c820a9024e14a1fc2 commit 214 146 610
702470739ce72005e2edff522fde85d52a65df9b commit 165 118 756
d368d0ac0678cbe6cce505be58126d3526706e54 tag    130 122 874
fe879577cb8cffcdf25441725141e310dd7d239b tree   136 136 996
d8329fc1cc938780ffdd9f94e0d364e0ea74f579 tree   36 46 1132
deef2e1b793907545e50a2ea2ddb5ba6c58c4506 tree   136 136 1178
d982c7cb2c2a972ee391a85da481fc1f9127a01d tree   6 17 1314 1 \
  deef2e1b793907545e50a2ea2ddb5ba6c58c4506
3c4e9cd789d88d8d89c1073707c3585e41b0e614 tree   8 19 1331 1 \
  deef2e1b793907545e50a2ea2ddb5ba6c58c4506
0155eb4229851634a0f03eb265b69f5a2d56f341 tree   71 76 1350
83baae61804e65cc73a7201a7252750c76066a30 blob   10 19 1426
fa49b077972391ad58037050f2a75f74e3671e92 blob   9 18 1445
b042a60ef7dff760008df33cee372b945b6e884e blob   22054 5799 1463 # 이게 나중 버전임
033b4468fa6b2a9547a70d88d1bbe8bf3f9ed0d5 blob   9 20 7262 1 \
  b042a60ef7dff760008df33cee372b945b6e884e #  033b44가 b042 blob을 참조하고 있는거임. 그래서 033b44를 보면 9byte만 먹고 있는걸 볼 수 있음. 이는 b042에 대한 delta만 저장하고 있기 때문이다.
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a blob   10 19 7282
non delta: 15 objects
chain length = 1: 3 objects
.git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.pack: ok
```

흥미로운거는 `b042`가 나중 버전이고 `033b44`가 이전 버전인데, `b042`를 온전히 저장하고 있고 `033b44`에는 `b042`에 대한 delta(변경사항) 만 저장하고 있다.
이는 일반적으로 나중 버전에 더 많이 접근하므로 더 나중 버전에 대한 접근 속도를 향상하기 위해서라는 거.

## Refspec

은 일단 패스

## Transfer Protocols

레포지토리 간 데이터 transfer는 "dumb" or "smart" protocol로 이루어진다.

### Dumb Protocol

레포지토리를 읽기 전용으로 HTTP를 통해 제공하려면 이거 쓰면 된다.
dumb인 이유는 서버 사이드에서 Git-specific한 코드를 사용할 필요가 없기 때문임.
fetch process는 여러 HTTP GET 요청의 연속이다.

```bash
git clone <원격_레포지토리_주소>
```

를 실행하면

1. `info/refs` 파일을 받아온다.

```text
GET info/refs
ca82a6dff817ec66f44342007202690a93763949     refs/heads/master
```

이를 통해서 remote ref와 SHA-1 값들을 가져올 수 있다.

2. HEAD가 어디를 가리키고 있는지 받아온다

```text
=> GET HEAD
ref: refs/heads/master
```

이제 작업이 끝난 후 어디로 checkout 할지 알 수 있다.
이 경우 master로 가면 된다.

3. master에 해당하는 ref가 가리키는 object 받아오기

```text
=> GET objects/ca/82a6dff817ec66f44342007202690a93763949
(179 bytes of binary data)
```

이 커밋 오브젝트를 뜯어보면 다음과 같을거고, 그 커밋 오브젝트를 구성하는 오브젝트들을 또 fetch 해온다.

```bash
$ git cat-file -p ca82a6dff817ec66f44342007202690a93763949
tree cfda3bf379e4f8dba8717dee55aab78aef7f4daf
parent 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
author Scott Chacon <schacon@gmail.com> 1205815931 -0700
committer Scott Chacon <schacon@gmail.com> 1240030591 -0700

Change version number
```

그런데 만약에 이 파일이 loose format으로 저장되어 있지 않다면 어떡할까(packfile로 저장되어 있다던가, alternate repository에 저장되어 있다던가)

그럴 경우 404를 돌려보내고, 우선 alternate가 목록에 적혀있는 지 확인해본다.

```bash
=> GET objects/info/http-alternates
(empty file)
```

만약에 alternate URL 목록이 온다면 git은 거기서 찾아본다. (보통 특정 프로젝트의 fork이거나 할때 이용)
아무것도 없다면 packfile을 찾아본다. packfile 관련 정보는 `objects/info/packs` 파일에 담겨 있다.

```bash
# 먼저 팩파일 이름을 받아오고
=> GET objects/info/packs
P pack-816a9b2334da9953e530f27bcac22082a9f5b835.pack

# 내가 찾는 오브젝트가 팩파일 안에 있는지 확인하기 위해서 인덱스파일을 받아오고 (팩파일 내 오브젝트의 SHA-1값과 offset이 들어있음)
=> GET objects/pack/pack-816a9b2334da9953e530f27bcac22082a9f5b835.idx
(4k of binary data)

# packfile내에 오브젝트가 있는걸 확인했다면 packfile을 받아온다.
=> GET objects/pack/pack-816a9b2334da9953e530f27bcac22082a9f5b835.pack
(13k of binary data)
```

### SMART Protocol

Dumb Protocol은 단순하지만 클라이언트에서 서버로 쓰기 작업을 할 수 없다는 단점이 있다.
Smart Protocol이 좀 더 좋지만, 서버가 깃을 잘 지원해야 한다. (로컬 데이터를 읽고, 클라이언트에 어떤 데이터가 있고 어떤 데이터를 필요로 하는 지 알아야 하고, 이를 위한 커스텀 packfile을 생성할 수 있어야 한다.)

스마트 프로토콜에는 데이터 업로드를 위한 프로세스, 데이터 다운로드를 위한 프로세스 이렇게 2가지 셋의 프로세스가 있다.

#### Uploading Data

##### HTTPS

`git push origin master` 을 치면, 깃은 `send-pack` 프로세스를 시작한다. 이 때 깃은 다음과 같은 HTTP 요청을 보낸다

```bash
=> GET http://server/simplegit-progit.git/info/refs?service=git-receive-pack
001f# service=git-receive-pack
00ab6c5f0e45abd7832bf23074a333f739977c9e8188 refs/heads/master□report-status \
	delete-refs side-band-64k quiet ofs-delta \
	agent=git/2:2.1.1~vmg-bitmaps-bugaloo-608-g116744e
0000
```

`git-receive-pack` 은 현재 갖고 있는 reference를 return 해준다.

reference를 서버에서 받은 클라이언트들은 서버의 상태를 이제 알기 때문에, `send-pack` 프로세스가 현재 클라이언트에 어떤 커밋이 있고, 서버에는 어떤 커밋이 없는지 체크한다.

`send-pack` 프로세스는 서버 측 `receive-pack` 프로세스에게 POST 요청을 보낸다. 이번 push가 업데이트할 모든 reference에 대한 데이터(`send-pack` output)와 서버에 없는 packfile 을 보내준다.

```text
=> POST http://server/simplegit-progit.git/git-receive-pack
```

클라이언트에서 깃은 각 reference 에 대해서 old SHA-1, new SHA-1, 그리고 업데이트 할 reference 정보를 보내준다.

예를 들어 master branch에 커밋을 넣고 experiment 브랜치가 추가된 상황이라면, `send-pack`의 response는 다음과 같다.

```text
0076ca82a6dff817ec66f44342007202690a93763949 15027957951b64cf874c3557a0f3547bd83b3ff6 \
	refs/heads/master report-status
006c0000000000000000000000000000000000000000 cdfdb42577e2506715f8cfeacdbabc092bf63e8d \
	refs/heads/experiment
0000
```

#### Downloading Data

다운로드 프로세스에서는 `fetch-pack` 과 `upload-pack` 프로세스가 사용된다.
클라이언트에서 `fetch-pack`을 해서 서버 측 `upload-pack` 프로세스와 연결하고, 어떤 데이터를 전송받을지 결정한다.

##### HTTP

fetch를 할 때는 2번의 HTTP Request를 한다.

첫번째는 GET 요청을 보낸다. 이에 대한 응답으로 현재 HEAD가 가리키는 위치를 보내준다.

```bash
=> GET $GIT_URL/info/refs?service=git-upload-pack
001e# service=git-upload-pack
00e7ca82a6dff817ec66f44342007202690a93763949 HEAD□multi_ack thin-pack \
	side-band side-band-64k ofs-delta shallow no-progress include-tag \
	multi_ack_detailed no-done symref=HEAD:refs/heads/master \
	agent=git/2:2.1.1+github-607-gfba4028
003fca82a6dff817ec66f44342007202690a93763949 refs/heads/master
0000
```

`fetch-pack`은 어떤 오브젝트가 있는지 확인하고, 자신이 가지고 있는 오브젝트에 대해선 'have', 서버로부터 받아와야 하는 오브젝트에 대해서는 'want'를 표시해서 서버에 POST 요청을 보낸다.

```text
=> POST $GIT_URL/git-upload-pack HTTP/1.0
0032want 0a53e9ddeaddad63ad106860237bbf53411d11a7
0032have 441b40d833fdfa93eb2908e52742248faf0ee993
0000
```

## Maintenance and Data Recovery

### Removing Objects

한가지 문제가 될 수 있는 상황은 `git clone`을 하면 프로젝트의 과거 history를 받게 되는데, 만약에 과거 히스토리에 엄청 큰 용량의 파일이 있었다면, clone 할 때마다 그 파일을 다운로드해야 하므로 문제가 될 수 있다.

> 이에 대한 해결방법이 적혀있긴 한데 이것까지 설명하기엔 좀 과한거같아서 있따 정도만 알아두고 넘어갑시다
