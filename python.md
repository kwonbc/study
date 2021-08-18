# Apple Silicon M1 Python 설치 정보

## Miniforge 설치

- 현재 Python 패키지 중에서 M1을 지원하는 것은 Miniforge 밖에 없음
- 나중에 Anaconda로 변경 시 최소 변경으로 적용 가능 예상

```zsh
brew install miniforge
```

## 작업 환경 구성

작업 경로에 의존하므로 작업 경로로 가서

```zsh
conda create --name <가상환경명> python==3.9.6
```

가상 환경 목록 보기

```zsh
conda env list
```

가상 환경 적용

```zsh
conda activate <가상환경명>
```

가상 환경 종료

```zsh
conda deactivate
```

패키지 설치 방법

```zsh
conda install <패키지명>
```