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

## iTerm2 + Powerline

### iTerm2 설치

```
https://iterm2.com
```

### Oh My Zsh 설치

```zsh
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

### Oh My Zsh 테마 변경

```zsh
git clone https://github.com/romkatv/powerlevel10k.git $ZSH_CUSTOM/themes/powerlevel10k
````

```zsh
vi ~/.zshrc
```

다음 값을 변경

```
ZSH_THEME="powerlevel10k/powerlevel10k"
```

터미널 재시작 후 설정 필요

### Powerline font 설치

```zsh
git clone https://github.com/powerline/fonts.git --depth=1
```

```zsh
cd fonts
```

```zsh
./install.sh
```

```zsh
cd ..
```

```zsh
rm -rf fonts
```

### iTerm2 설정
