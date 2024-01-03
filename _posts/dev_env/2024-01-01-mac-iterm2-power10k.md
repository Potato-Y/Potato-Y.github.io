---
title: "Mac iterm2, oh-my-zhs/Powerlevel10k 설치하기"

toc: true
toc_sticky: true

categories:
    - dev_env
tags:
    - Mac
---

# Mac 터미널 셋팅하기

## iterm2 설치

`brew`를 통해 `iterm2`를 설치한다.

```bash
$ brew install --cask iterm2
```

iterm2를 실행한다.

## oh my zsh 설치

아래의 세 가지 방법 중 하나를 선택하면 된다.

- curl
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

- wget 
```bash
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

- fetch
```bash
sh -c "$(fetch -o - https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### 기본 쉘 설정

현재 설정된 쉘을 확인한다.
```bash
echo $SHELL
```

만약 `zsh`가 아니라면 다음의 방법으로 변경한다.

우선 zsh가 설치된 위치를 확인한다.
```bash
which zsh
```

쉘 정보가 있는 파일에 위에서 찾은 `zsh`의 위치가 명시되어 있는지 확인한다.
```bash
vim /etc/shells
```

확인 후 변경한다.
```bash
chsh -s `which zsh`
```

## Powerlevel10k 설치

다음의 명령어로 Powerlevel10k를 다운로드 한다.

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

`zsh` 설정 파일을 연다.
```bash
vim ~/.zshrc
```

아래와 같이 수정한다.
```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

### power10k 설정

터미널을 재실행한다.

폰트를 설치할 것인지 묻는다. 당연히 설치한다.
```bash
   This is Powerlevel10k configuration wizard. You are seeing it because you
 haven't defined any Powerlevel10k configuration options. It will ask you a few
                      questions and configure your prompt.

                            Install Meslo Nerd Font?

(y)  Yes (recommended).

(n)  No. Use the current font.

(q)  Quit and do nothing.

Choice [ynq]:y
```

이후 폰트 적용을 위해 터미널을 재실행한다. 그 후 특수문자가 정상적으로 표시되는지 확인하고, 개인 선호에 맞게 설정을 마치면 된다.
