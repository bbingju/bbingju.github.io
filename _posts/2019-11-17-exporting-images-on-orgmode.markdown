---
layout: post
title:  "Org-mode에서 생성한 이미지 결과물이 잘 보이게 하기"
date:   2019-11-17
categories: emacs org-mode
---

이 포스팅은 이맥스 org-mode 관련 간단한 팁이다.


# 시작

텍스트로 그려진 다이어그램을 이미지로 변환해주는
[프로그램](http://ditaa.sourceforge.net/)이 있다. 아래와 같이 텍스트로
다이어그램을 그렸다면,

```
    /------------------\                                     +--------+ 
    | Smart PEG Board  |                                     | Apps   | 
    | c1AB             |    Classic Bluetooth SPP Profile    |        |
    |                  | <---------------------------------> |        | 
    |                  |                                     |        |
    |                  |                                     |        |
    \------------------/                                     +--------+
```

다음과 같이 변환해준다.

![변환된 다이어그램](https://github.com/bbingju/smart-peg/raw/master/doc/pic/smart-peg-system.png)


이 프로그램은 오래되긴 했지만, 간단한 다이어그램이 필요할 때 유용하게
쓸 수 있다. 뭐가 좋냐고? Plain text 기반 문서 작업을 많이 하다보면 알
수 있다. 우선 간단한 에디터에서도 다이어그램이 어떻게 생겼는지 문맥
흐름을 해치지 않고도 알 수 있고, 무엇보다
[WYSIWYG](https://en.wikipedia.org/wiki/WYSIWYG) 툴로 그릴 때
고려해야할 도형 크기나 위치 같은 자잘한 것들에 대해 신경쓰지 않아도
된다.

그런데 이 툴과 emacs org-mode와는 무슨 관계일까?


# Org-mode Exports

Org-mode는 기본적으로
[Babel](https://orgmode.org/worg/org-contrib/babel/) 기능을
지원한다. 쉽게 말해 문서 내에 삽입된 소스 코드를 실행할 수 있게 해주는
강력한 기능이다[^1]. 물론 세상의 모든 프로그래밍 언어를 지원하는 것은
아니지만, ditaa도 기본적으로 지원하기 때문에 그냥 쓸 수 있다.

예를 들어, Org 문서 내에서 Python 코드를 실행하고자 한다면 대략 다음과
같이 쓰면 된다.

```
#+BEGIN_SRC python :results output :exports both
  print("Hello Org Babel")
#+END_SRC
```

그리고는 `C-c C-c (org-babel-execute-src-block)` 같은 명령을 실행하면
그 결과가 문서에 아래와 같이 기록된다.

```
#+RESULTS:
: Hello Org Babel
```

# Github README

Github은 프로젝트 루트에 README 문서를 두면 그것을 자동으로 html
포맷으로 변환하여 보여준다.  `README`는 전통적으로 plain text이지만
github에서는 markdown 이나 org 포맷 등도 지원하므로 편의에 따라 쓸 수
있다.

나는 orgmode를 선호하므로 `README.org`를 항상 사용한다. 그런데 최근에
올린 프로젝트 README 파일의 ditaa babel 결과물이 github 프로젝트
페이지에서 제대로 표시되지 않아서 약간 껄적지근 했었다. 해결 방법이
없는지 검색해도 내 부족한 영어실력 때문인지 해결책을 찾을 수 없었다.

무슨 문제냐면,

```
  #+BEGIN_SRC ditaa :file doc/pic/smart-peg-system.png :cmdline -E :results file

    /------------------\                                     +--------+ 
    | Smart PEG Board  |                                     | Apps   | 
    | c1AB             |    Classic Bluetooth SPP Profile    |        |
    |                  | <---------------------------------> |        | 
    |                  |                                     |        |
    |                  |                                     |        |
    \------------------/                                     +--------+

  #+END_SRC
```

위와 같이 코드를 집어 넣으면 변환된 이미지가 github 페이지 문서에
포함되어 보여야하는데 아래와 같이 원소스 그대로 html에 출력되었다.

![원하지 않는 결과](/assets/orgmode-exports-unwanted-result.png)

[여기](https://github.com/bbingju/smart-peg/blob/master/doc/pic/smart-peg-system.png)에서
보듯이 파일 변환이 안된 건 아니다. 변환된 이미지가 html 문서에 포함되지 않은 것이다.

그러다가 우연히
[메뉴얼](https://orgmode.org/manual/Specific-header-arguments.html#Specific-header-arguments)을
보게 되었다. 게중에 `:exports`라는 헤더 인자가 있네. 그리고 그 값으로 `code`, `results`, `both`, `none` 이 있는 걸 확인하고,
위 헤더를 다음과 같이 고쳤더니,

```
  #+BEGIN_SRC ditaa :file doc/pic/smart-peg-system.png :cmdline -E :results file :exports results
```

아래와 같이 제대로 나왔다!

![원하는 결과](/assets/orgmode-exports-wanted-results.png)

그러니까 `:exports results`를 더 추가한 것인데, 그동안 `:exports` 헤더 인자의
기본값이 `code`로 되어 있어서 계속 텍스트로만 변환되었던 것이다.


# 결론

메뉴얼을 잘 읽어보자.

끝.

[^1]: Python 쪽에서는 [Jupyter Notebook](https://jupyter.org/) 같은 것들도 있다.
