---
layout: post
title:  "React & Immutable"
date:   2017-08-16 21:22:48 +0900
categories: log
---

[React](https://facebook.github.io/react/)에서 말하는 Immutable은 이
한 문장으로 요약된다. 

> Whenever your object would be mutated, don’t do it. 
> Instead, create a changed copy of it.

모든 UI library 혹은 framework가 그렇듯 화면
업데이트를 가능한한 최소화해야 성능을 극대화 할 수 있다. React도
마찬가지인데 DOM의 복사본이라 할 수 있는 VirtualDOM을 만들어 그것이
바뀌는 시점에만 화면을 업데이트하겠다는 것이 컨셉인 것이다. 문제는
VirtualDOM이 바뀌었다는 것을 어떻게 체크하냐는 것인데, 어떤 시점의 전
후 상태를 비교할 때 바뀐 값을 모두 비교하는 것은 비효율적이다. 하지만
객체가 한번 생성되면 바뀌지 않는다는 원칙을 정하고(immutable), 만약
객체가 바뀌어야 한다면 새로운 객체로 만들어(기존 것을 복사하고)
변경점을 저장하게 한다면, 레퍼런스(주소) 값만의 비교로 변경여부를
확인할 수 있다. 자세히 파 봐야하겠지만, React의 기본 개념은 이것인 것
같다. 올레~

Filesystem에서 사용하는 [COW
(Copy-on-Write)](https://en.wikipedia.org/wiki/Copy-on-write)
매커니즘이나, Git의 변경점 저장단위인 blob도 비슷한 원리이다.
