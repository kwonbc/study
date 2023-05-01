# Planck Keyboard - vial 지원

tap dance를 설정하려면 qmk 컴파일 아니면 vial을 사용
- qmk 사용하려면 컴파일을 해서 업로드를 해야 하기 때문에 실시간으로 설정이 어려움
- vial을 사용하면 설정 즉시 확인이 가능하나 vial을 지원해야 함

현재 planck v6 drop 버전은 vial을 지원하지 않고 누가 지원하게 만든 fork 버전을 사용해야 함
- fork 버전 컴파일에 대한 방법이 문서로 잘 정리되어 있지 않아 내가 나중에 알 수 있는 수준으로 문서화 남김

## 1. qmk 개발 환경 설정

qmk msys 설치 -윈도우 환경에서 개발 도구를 한방에 설치

```
https://msys.qmk.fm/
```

완료되면 QMK MSYS와 QMK TOOLBOX가 설치됨
- QMK MSYS는 개발 관련 작업을 하는 커맨드 라인 쉘
- QMK TOOLBOX는 컴파일된 바이너리 파일을 실제로 키보드에 flashing하는 도구

## 2. fork된 소스 가져오기

git을 이용하여 vial 지원 소스를 내려받음
- QMK MSYS 실행

```
git clone https://github.com/vial-kb/vial-qmk
```

## 3. 컴파일 환경 구성

가져온 소스를 컴파일 할 수 있도록 설정

```
cd vial-qmk
make git-submodule
qmk setup
```

## 4. 소스 복사

참고로, 대부분 내려받은 소스를 default 옵션을 이용하여 그대로 사용하는 걸로 설명되어 있으나 
내 경우에는 그렇게 하면 인식이 안되어 keyboards 경로 내 소스를 vial이라는 이름으로 복사해서 사용

```
cd keyboards\planck\rev6_drop keyboards\planck\vial
```

내 키보드는 planck v6 DROP 버전이라 이것을 선택, 각자 자기 키보드에 맞는 소스를 복사

## 5. 컴파일

여기서부터는 일반적인 qmk 컴파일 과정과 동일

- 자기 키보드에 맞는 옵션을 사용해야 함

```
make planck/rev6_drop:vial
```

## 6. 플래싱

현재 경로(\vial-qmk)에 컴파일된 바이너리 (planck_rev6_drop_vial.bin)가 생성되어 있음
QMK TOOLBOX를 실행시키고 바이너리를 플래싱

끝!

