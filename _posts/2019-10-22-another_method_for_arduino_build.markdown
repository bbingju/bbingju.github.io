---
layout: post
title:  "아두이노 앱을 빌드하는 또다른 방법"
date:   2019-10-22
categories: embedded programming
---

# 들어가며

최근에 Arduio Pro IDE preview 버전
[발표](https://blog.arduino.cc/2019/10/18/arduino-pro-ide-alpha-preview-with-advanced-features/)가
있었다. [기존 IDE](https://www.arduino.cc/en/Main/Software)를 대신할
도구를 내놓을 준비를 하고 있는 것이다.

물론 나는 Arduino Pro IDE가 나와도 쓰지 않을 것이다. 이미 만족하며
쓰고 있는 조합(emacs + Makefile)이 있기 때문이다. 이번 기회에 내가
사용하고 있는 방법에 대하여 간단히 설명해보려 한다.

참고로, 이 글은 AVR 기반 아두이노에 대해서만 다룬다.


# 기존 IDE의 문제점들

기존 IDE는 간단한 프로그램을 작성하기엔 충분하지만, 전문적인
소프트웨어 엔지니어라면 아무도 쓰지 않을 도구이다. 에디터는 윈도우
노트패드에 syntax highting만 넣어둔 수준이고, 키 바인딩을 바꾸거나
소스 간 점프를 지원하지도 않기 때문이다.

게다가 라이브러리 매니저가 포함되어 있고 자동 업데이트도 되지만,
펌웨어에 포함될 모든 코드를 한 곳에 모아두기가 어려운 구조를 가지고
있다. 소스가 대략 3가지(Arduino Core, libraries, Main code)로
구성되는데 한 소스트리 아래에 모여있지 않다. 임베디드 환경에서
이런 상황은 많은 문제를 일으킨다. 

구체적인 예로 설명하면, 내가 작성한 펌웨어 코드를 다른 이에게 전달할
때, 코드에서 사용한 라이브러리를 직접 설치할 수 있도록 따로 설명을
덧붙여야 하고 버전도 맞추어야 하는 일이 벌어진다. 그럼에도 빌드가 되니
안되니 하는 말들을 주고 받아야할 가능성이 크다. 코드와 빌드스크립트에
이런 모든 것들이 기술되어 있어 간단한 명령으로 내가 만들었던 펌웨어와
똑같은 펌웨어를 만들어낼 수 있으면 좋지않겠는가?  이렇게 하면 비단
상대방을 위해서 뿐만 아니라 자신의 개발 히스토리를 정리하는데도 훨씬
좋다.


# 내가 사용하는 방식

생각한 소스트리 구성은 대략 다음과 같다.

```
my-awesome-firmware/
|-- build/
|-- core/
|-- lib/
|-- main.ino
`-- Makefile
```

- `build/`에는 빌드한 결과물들이 들어간다.
- `core/`는 Arduino Core 소스가 들어간다. 아두이노가 구동되는 다양한
  AVR 칩들이 있어서 Core 소스가 몇가지 된다. 기본적으로는
  [ArduinoCore-avr](https://github.com/arduino/ArduinoCore-avr) 정도로
  모두 대응이 되겠지만, 칩에 따라서는
  [MegaCore](https://github.com/MCUdude/MegaCore) 같은 걸 써야할
  경우도 있다.
- `lib/`에는 사용할 아두이노 라이브러리들이 들어간다. github를
  검색하면 엄청나게 많은 라이브러리들이 있다. 혹은 내가 직접
  라이브러리를 만들어 넣어도 된다.
- `main.ino`는 애플리케이션 코드가 들어간다(`setup()` 과 `loop()`가 여기 포함됨).
- `Makefile`은 makefile이다.

대략 위와 같은 소스트리를 가지고, git을 통해 관리한다. core와 lib는
대부분 github에서 구할 수 있으므로 해당 repo를 git submodule 형식으로
포함시키면 깔끔해진다.

결국 아두이노도 C/C++ 코드 덩어리이므로 Makefile 혹은 cmake 같은 빌드
스크립트로 간단히 구성할 수 있지 않겠냐는 생각이 들었고, 찾아보니
[적당한 것](https://github.com/sudar/Arduino-Makefile)이
있었다. 코드도 있지만 운영체제별로 패키징가 이미 제공되고
있었다. 우분투 혹은 데비안이면,

```sh
sudo apt install arduino-mk
```

로 avr toolchain과 arduino makefile을 설치하면 된다. 그리고 맥에서는,

```sh
brew tap sudar/arduino-mk
brew install arduino-mk
```

로 수행하여 패키지를 설치하면 된다.

그리고 Makefile의 예는 대략 다음과 같다.

```Makefile
PROJECT_DIR = $(PWD)

ARDUINO_DIR   = $(PROJECT_DIR)/core/MegaCore/avr
ALTERNATE_CORE_PATH = $(PROJECT_DIR)/core/MegaCore/avr

# on the Linux
ARDMK_DIR     = /usr/share/arduino
AVR_TOOLS_DIR = /usr
ARDUINO_PORT = /dev/ttyUSB0
AVRDUDE_CONF  = $(ALTERNATE_CORE_PATH)/avrdude.conf

# on the Mac
# ARDMK_DIR   = /usr/local/opt/arduino-mk
# AVR_TOOLS_DIR = /usr/local
# AVRDUDE_CONF  = $(ALTERNATE_CORE_PATH)/avrdude.conf
# ARDUINO_PORT = /dev/tty.SLAB_USBtoUART

AVRDUDE_ARD_BAUDRATE = 38400
BOARD_TAG = 64
F_CPU = 8000000L

USER_LIB_PATH += $(PROJECT_DIR)/lib
ARDUINO_LIBS  += Wire \
	SoftwareSerial

CURRENT_DIR       = $(shell basename $(CURDIR))
OBJDIR  = $(PROJECT_DIR)/build/$(BOARD_TAG)/$(CURRENT_DIR)

include $(ARDMK_DIR)/Arduino.mk
```

위 스크립트에는 다음과 같은 것들이 기술되어 있다.

- 사용할 arduino-mk와 avr toolchain의 경로
- flashing과 monitoring을 위한 장치파일 및 flashing software 설정 파일 경로
- 보드 혹은 칩 타입
- 사용할 라이브러리 폴더 이름

이렇게 하고 소스 루트에서 `make` 명령을 내리면 빌드가 되고, `make
upload` 명령을 내리면 빌드된 펌웨어가 보드로 플래싱될 것이다.

참고로, 실제 이런 방식으로 구현된 예제는
[여기](https://github.com/bbingju/kappa-digital-clock)에서 확인해볼 수
있다.

끝.

