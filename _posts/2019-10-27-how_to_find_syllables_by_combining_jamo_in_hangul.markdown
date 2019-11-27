---
layout: single
title:  "유니코드에서 한글 자모 조합으로 음절을 찾는 방법"
date:   2019-10-27
categories: hangul programming
---

얼마 전에 한글입력기에서 한글 자모가 조합되어 음절 하나가 되는 과정이
궁금해져서 관련 코드를 찾아보는 꿈을 꾸었었다. 신기한 꿈이라 아직 기억을
하는데 오늘 실제로 그 부분을 찾아봤다.

우선 [최신 유니코드 표준 중 동아시아 장](https://www.unicode.org/versions/Unicode12.0.0/ch18.pdf "Unicode12.0.0 chapter18")에 한글이 어떻게 정의되어 있는지 봐야한다. 


# 유니코드

우선 자모(*jamo*)가 정의되어 있다. 알파벳이라 할 수도 있겠지만 한글은
독특하게도 초성, 중성, 종성이 모여 하나의 음절(*syllables*)를 이루기
때문에 그냥 jamo라 부르는 듯 하다.

한글 자모는 `U+1100`에서 `U+11FF`까지의 영역에 초성,중성,종성 순으로
정의되어 있다. 영역이 한 바이트이므로 최대 255개를 정의할 수 있는데,
이 정도면 옛한글 자모까지 포함해도 충분한 듯 한다. 우리가 두벌씩
자판을 써서 초성과 종성이 하나인 것처럼 여기고 있지만 유니코드
자모표에는 둘이 구분되어 있다는 점이 놀랍다. 구체적인 장표는 
[이 문서](https://www.unicode.org/charts/PDF/U1100.pdf)에서 확인할 수 있다.

그리고 음절(*syllables*)은 `U+AC00` 에서 `U+D7AF`까지 정의되어 있고
Syllables는 [이 문서](https://www.unicode.org/charts/PDF/UAC00.pdf)에서 확인할 수
있다. 엄청 방대한 분량이다. 아마 CJK 한자 다음으로 많은 영역을
차지하고 있을 꺼다.


# 세 자모(초성, 중성, 중성)로 음절을 찾는 방법

구현된 코드를 보는 것이 빠르다. 유닉스 영역에서 오래 전부터 사용해오던
C 라이브러리인 [libhangul](https://github.com/libhangul/libhangul)을
보자.

[이 소스](https://github.com/libhangul/libhangul/blob/master/hangul/hangulctype.c)를
보면 언급한 유니코드의 내용을 구현해 놓았다. 거기서 내가 궁금해하는
부분이 아래 함수 하나로 구현되어 있다.

```c
static const ucschar syllable_base  = 0xac00;
static const ucschar choseong_base  = 0x1100;
static const ucschar jungseong_base = 0x1161;
static const ucschar jongseong_base = 0x11a7;
static const int njungseong = 21;
static const int njongseong = 28;

/**
 * @ingroup hangulctype
 * @brief 자모 코드를 조합하여 한글 음절로 변환
 * @param choseong 초성이 될 UCS4 코드 값
 * @param jungseong 중성이 될 UCS4 코드 값
 * @param jongseong 종성이 될 UCS4 코드 값
 * @return @a choseong @a jungseong @a jongseong을 조합한 현대 한글 음절 코드,
 *         또는 0
 *
 * 이 함수는 @a choseong @a jungseong @a jongseong으로 주어진 코드 값을 각각
 * 초성, 중성, 종성으로 하는 현대 한글 음절 코드를 구한다.
 * @a choseong @a jungseong @a jongseong 이 조합 가능한 코드가 아니라면 
 * 0을 리턴한다. 종성이 없는 글자를 만들기 위해서는 jongseong에 0을 주면 된다.
 */
ucschar
hangul_jamo_to_syllable(ucschar choseong, ucschar jungseong, ucschar jongseong)
{
    ucschar c;

    /* we use 0x11a7 like a Jongseong filler */
    if (jongseong == 0)
	jongseong = 0x11a7;         /* Jongseong filler */

    if (!hangul_is_choseong_conjoinable(choseong))
	return 0;
    if (!hangul_is_jungseong_conjoinable(jungseong))
	return 0;
    if (!hangul_is_jongseong_conjoinable(jongseong))
	return 0;

    choseong  -= choseong_base;
    jungseong -= jungseong_base;
    jongseong -= jongseong_base;

    c = ((choseong * njungseong) + jungseong) * njongseong + jongseong
	+ syllable_base;
    return c;
}
```

쉽게 설명하자면 유니코드에서 한글 음절이 정의된 순서는 각 초성, 중성,
종성이 정의된 순서와 같기 때문에 일정한 오프셋을 가진다는 것이
핵심이다. 그래서 다음과 같은 공식으로 구현할 수 있는 것이다.

	음절 = ((초성인덱스 * 전체중성수) + 중성인덱스) * 전체종성수 + 종성인덱스

예를 들어보자.

`형` 이라는 음절을 찾는다고 하면 자모는 다음과 같이 구성되고,

| 자모 | 코드   | 비고 |
| ---- | ------ | ---- |
| `ᄒ` | 0x1112 | 초성 |
| `ᅧ` | 0x1167 | 중성 | 
| `ᆼ` | 0x11BC | 종성 |

위 계산식에 따르면, 

	초성인덱스(0x0012) = 초성코드(0x1112) - 초성코드베이스(0x1100)
	중성인덱스(0x0006) = 중성코드(0x1167) - 중성코드베이스(0x1161)
	종성인덱스(0x0015) = 종성코드(0x11BC) - 종성코드베이스(0x11a7)
	
	((0x12 * 21) + 0x6) * 28 + 0x15 + 음절코드베이스(0xac00)
	= ((18 * 21) + 6) * 28 + 21 + 0xac00 = 0x2a15 + 0xac00 = 0xd615

따라서 음절 `형`은 `0xd615`이 된다.

우리가 키보드로 각 자모를 입력하여 음절을 만들 때마다 이런 식의 계산이
일어난다.  그리고 두벌식 자판에서 초성인지 종성인지 확인하는 방법은
스테이트머신을 이용하여 초성,중성이 순서대로 입력되면 다음에 입력되는
자음은 종성이라는 식으로 확인할 수 있고 자음을 종성으로 변환하는
테이블을 정의해놓고 쓰면 될 듯 하다.


# 마지막으로

과거 한글입출력이 하드웨어 카드로 구현되던 시절에는 컴퓨터에 한글이
보여지고 입력도 가능하다는 것이 엄청난 기술처럼 보일 때도
있었다. 실제로 그때는 그게 엄청난 기술이었을 것이다. 문자당 할당된
데이터 크기가 7비트 혹은 한 바이트였기 때문에 완성형 한글 자체를 쓸 수
없었고, ascii 영역 이외에 특수문자 영역을 임의로 한글 코드로 정의하여
써야했기 때문에 호환성 문제도 발생했었다. 이런 저런 이유로 나에게는
한글 인코딩이 어렵고 힘든 영역이란 선입견이 있었다. 그게 조금은 깨진
것 같다.

끝.
