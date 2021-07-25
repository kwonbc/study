저장소 생성
```
git init
````

정보 설정
```
git config -g user.name <name>
```
```
git config -g user.email <email>
```

상태 확인
```
git status
```

파일 추가
```
git add -A
```

커밋
```
git commit -m "<comment>"
```

로그 확인
```
git log
```

특정 시점으로 되돌리기 (복구 불가)
```
git reset <gitno> --hard
```

특정 시점으로 돌아가기
```
git revert <gitno>
```

브랜치 생성
```
git branch <branch>
```

브랜치 확인
```
git branch
```

브랜치 이동
```
git checkout <branch>
```

브랜치 생성 후 이동
```
git checkout -b <branch>
```

브랜치 합침
```
git merge <branch>
```

브랜치 합친 후 정리
```
git rebase <branch>
````

브랜치 삭제
```
git branch -D <branch>
```

리포지토리 추가
```
git remote add <remote> <rep_url>
```

리포지토리 확인
```
git remote
```

리포지토리 커밋
```
git push -u <remote> <branch>
```

리포지토리 내려받기
```
git clone <remote>
```

리포지토리 상태 확인
```
git fetch
```

리포지토리 상태 적용
```
git pull <remote> <branch>
```

리모트 브랜치 확인
```
git branch -a
```

새 브랜치 생성 후 리모트 브랜치 가져오기
```
git checkout -b <new_branch> <remote>/<branch>
```

리모트 브랜치 삭제
```
git push -D <remote> <branch>
```
