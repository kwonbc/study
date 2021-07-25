# macOS 세팅 정보

## Apple Silicon M1 Homebrew

### 설치

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 경로 추가

```zsh
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/<USER_ID>/.zprofile
```
```zsh
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 설치 확인

```zsh
which brew
```
```zsh
brew --version
```
