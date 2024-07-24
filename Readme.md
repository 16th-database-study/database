# 1. 파일 변경 관련 규칙
## 1.1 Github 내 프로젝트 클론

메인 프로젝트 fork!
![img.png](img/readme_main/clone_image.png)

fork 생성! Owner를 본인으로 놓고 생성하면 됨!
![img.png](img/readme_main/fork_image.png)

## 1.2 인텔리제이 IDE 접속 및 커맨드

따로 fork해서 dev나 뭐 아무 이름 브랜치 생성해서 PR 보내고 카톡 남기면 됨. 개인적으로 확인하고 PR 내용 보고 병합하겠음.
```shell
데이터 가져오기
> git clone [복사URL]
> git checkout -b dev
> git pull
글 작성 후
> git add -A
하단 커밋 메시지 규칙: [작성자] : 변경 챕터, 간단 이유
> git commit -m [커밋 메시지]
> git push origin
```