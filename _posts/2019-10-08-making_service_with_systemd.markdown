---
layout: single
title:  "리눅스 앱을 서비스로 만들기 위해 필요한 것들"
date:   2019-10-08
categories: [programming]
tags: [systemd, daemonize, linux]
---

# 들어가며

가끔은 끊임없이 실행되면서 주어진 작업을 수행하는 프로그램이 필요할
때가 있다. 프로그램이 끝나지 않고 끝없이 돌게 하려면 어떻게 해야할까?
간단하다. 무한루프를 만들고 그 안에다 우리가 원하는 일을 하도록 코드를
구현하면 된다.

```c
while (1) {
	// 원하는 작업 코드
}
```

이 구조가 흔히 서버나 서비스라고 일컫는 프로그램들의 기본
구조이다. 그러나 이 사실을 안다고 해서 만족할 만한 프로그램을
만들 수는 없다. 어떤 것들이 더 필요할까?


보통 첫번째로 접하는 어려움은 세션유지 문제일 것이다. 원격터미널로
접속하여 만든 프로그램을 실행시키고 그 터미널을 빠져나오면, 실행시켰던
프로그램도 같이 죽어버린다. 만든 프로그램의 프로세스는 원격터미널의 쉘
프로세스의 자식 프로세스이므로 부모가 죽으면(터미널을 빠져나가면) 따라
죽는 것이다.

이 문제를 해결하는 기본적이고도 확실한 방법은 모든 프로세스의
조상이면서 시스템이 살아있는 한 항상 존재하는 `init` 에게 내
프로그램을 양자로 보내는 것이다. 이것을 daemonize 라고 하는데 아래에서
다루겠다.

또 다른 방법은 `tmux` 같은 다중 터미널 관리기를 이용하는 것이다. :-)


# 간단한 방법

`tmux` 는 다중으로 가상터미널을 생성하고 그 세션을 계속 유지해주는
기능을 가진 프로그램이다. tmux daemon이 백그라운드로 하나 돌고 있고,
새로운 터미널 세션을 생성하면 tmux daemon의 자식으로 돌게 만들어서
로그인 했던 원래 세션을 끊어도 tmux를 통해 만든 터미널 세션들은 계속
살아있게 하는 방식이다. 한마디로 로컬 콘솔에서 로그인 한 채로 만든
프로그램을 구동시키고 로그아웃을 하지 않은 채로 그대로 돌아가게 두는
것과 같은 것이다.

![tmux 안에서 프로그램을 구통시킨 화면](/assets/img/running-app-within-tmux.png)

이 방법은 개발할 때 이용하면 여러모로 편할 뿐만 아니라, 쓰는 사람이 몇
안되는 소규모 서비스들은 이렇게 사용해도 충분한 경우가 많다.

하지만 tmux 만으로 서비스를 구동시키는 것 또한 문제를 안고 있다. 여러
이유로 프로그램이 의도치 않게 죽거나 시스템이 재부팅되었을 경우에
서비스를 수동으로 다시 실행시켜줘야 한다. 게다가 프로그램이나 시스템이
죽을 때 "나 죽는다" 하고 알려주지 않으니(물론 그렇게 구현할 수 있지만
공력이 많이 들어간다) 죽는 시점을 정확히 알기 어려워 자칫 서비스의
공백이 발생할 수도 있다.

그러면 좀 더 나은 방법은 무엇이 있을까?

# Daemonize

좀 더 근본적이고 확실한 방법은 앞에서도 잠깐 언급했듯이 내 프로그램의
프로세스를 데몬화(daemonize)하는 것이다. 데몬화를 하면 다음과 같은 일들이 일어난다.

1. `fork()` 를 통해 데몬으로 만들 프로세스를 생성 후, 부모 프로세스는
   프로세스를 죽이고 자식 프로세스만 살린다. 이 과정은 데몬의 부모
   프로세스를 죽임으로써 할아버지 프로세스가 더 이상 자식을 찾지 않게 만들기 위함이다.
2. 살아있는 자식 프로세스는 `setsid()` 시스템콜을 통해 새로운 프로세스 그룹과 세션을 부여받는다.
3. 현재 디렉토리도 `/` 로 바꾼다.
4. 열려있던 모든 file descriptor들 닫는다.
5. file descriptor stdin(0), stdout(1), stderror(2)를 생성 후 `/dev/null`로 모두 리다이렉팅 시킨다.

위 과정이 바로 C 표준 라이브러리에 있는 [daemon(3)](https://linux.die.net/man/3/daemon)이 작동하는 방식이다.

즉 C언어로 백그라운드로 돌아가는 프로그램을 만들려면 [이런식](https://gist.github.com/copyninja/1033862)으로
코딩해야하는 것이다. 좀 귀찮지 않은가? 더군다나 자바나 파이썬 같은 언어로 뭔가 만들려고 한다면?

물론 C가 아닌 언어들도 시스템 라이브러리에 데몬화해주는 API를 대부분
가지고 있을 것이지만(안찾아봐서 정확한 건 아님) 좀 더 쉬운 방법이 있다. 
바로 [systemd](https://www.freedesktop.org/wiki/Software/systemd/)가 제공하는 기능을 이용하는 것이다.


# systemd

`systemd`는 요즘 왠만한 리눅스 배포판은 다 사용하는 init
시스템이다. 과거에는
[SysVinit](https://wiki.archlinux.org/index.php/SysVinit)을 주로
사용하였으나, 많은 논란과 함께 대부분의 메이저 배포판들은 systemd로
일원화되었다.

그러면 systemd로 내 프로그램을 시스템 서비스로 만드는 방법에 대해 알아보자.

우선 systemd unit 파일을 만들어야 한다. `my-awsome-app` 이라는 프로그램을 서비스로 만들고 싶다고 가정하면 우선 다음과 같은 절차가 필요하다.

 `my-awsome-app.service` 파일을 아래와 같이 만든다.

```sh
# systemd unit file for My Awsome App

[Unit]
Description=My Awsome App

[Service]
ExecStart=/usr/local/bin/my-awsome-app

[Install]
WantedBy=default.target
```

이 파일을 `/etc/systemd/system/` 아래에 복사한다. `my-awsome-app` 파일도 `/usr/local/bin/` 아래에 복사한다.

service unit 파일을 systemd가 인식하고 있는지 확인하려면 다음과 같이 명령해 본다.

```sh
$ systemctl list-unit-files | grep my-awsome-app
my-awsome-app.service                    disabled
```

이제 서비스를 돌려보자.

```sh
$ sudo systemctl start my-awsome-app
```

시스템이 부팅될 때 자동으로 실행되게 하려면 다음과 같이 한다.

```sh
$ sudo systemctl enable my-awsome-app
```

암튼 이런 과정을 통해 서비스를 쉽게 구현할 수 있다. 또하나. systemd를
이용하면 프로그램상의 모든 표준출력, 표준에러로 출력하는 모든 메시지를
syslog로 기본 저장된다. 직접 로깅 로직을 만들 필요가 없는 것이다.

그리고 또하나. 내 서비스가 오류가 나서 죽었을 때 자동으로 실행하게
하려면 service 파일 명세에 `Restart=on-failure`를 `[Service]` 항목 아래에 추가해주면 된다.


# 마무리

이상 특정 사이트 페이지를 계속해서 읽어서 바뀐부분을 체크하는 [간단한
서비스](https://github.com/bbingju/lp7080-notifier)를 만들며 알아봤던
것을 정리해봤다.

끝.
