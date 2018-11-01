---
layout: post
title:  "[jQuery] IE9에서 Transform Animation 구현하기"
date:   2018-11-01 19:00:00
author: Jaden
categories: jQuery
cover:  "/assets/instacode.png"
tags: css3 transform transition jquery motion animation
share: true
---

## 서론

보통 `transform`을 통한 애니메이션을 구현할 때에는 `transition`을 활용한다.

| <center> </center> | <center>IE 8</center> |  <center>IE 9</center> |  <center>IE 10~</center>
| ------------ | ------------ | ------------- | -------------
| <center>transform</center> | <center>X</center> | <center>O<br>[-ms-]</center> | <center>O</center>
| <center>transition</center> | <center>X</center> | <center>X</center> | <center>O</center>

위 호환 표처럼 IE10부터 `transition`이 가능하기 때문에 IE9에서는 `transform`이 가능하더라도 CSS로 애니메이션을 사용할 수가 없다.

그래서 보통 IE9는 애니메이션을 자바스크립트로 구현을 해야 하는데, `setInterval`을 통해 구현하는 방법도 가능하지만 그보다 좀 더 편한 `jQuery.fn.animate` 함수를 통해 구현하는 것을 추천한다.

-----

## jQuery.fn.animate

```
  $(selector).animate({
    opacity:1
  }, {
    duration: 500
  });
```
위 코드는 `jQuery.fn.animate` 함수를 간단히 사용한 구문이다.

```
  $(selector).animate({
    opacity: 1,
    transform: 'translateY(100px)' // 작동하지 않음
  }, {
    duration: 500
  });
```
이를 `transform` 속성을 추가하고 끝! 하면 좋겠지만 추가된 문은 구현되지 않는다..

`jQuery.fn.animate`에서는 `top`, `left`, `margin`, `padding` 등 일반적으로 값이 **단일 숫자값인 경우에 작동**하기 때문에 `transform`은 작동이 되지 않는다. (표시 유형의 속성에서는 `show`, `hide`, `toggle`도 사용 가능)

그래서 이를 해결하기 위해 `step`이라는 메서드를 활용해야 한다.

### step

`jQuery.fn.animate`를 실행하면 애니메이션 단계마다 `step` 메서드가 실행되면서 인자에 해당 값을 전달하게 된다.

```
  // 기본이 opacity: 0 인 경우로 가정한다.
  $(selector).animate({
    opacity: 1
  }, {
    duration: 500,
    step: function (now){
      console.log(now);
    }
  });

  /*
  실행 시 아래와 같이 값이 출력된다.
  0
  0.0019331954284137476
  0.007177702425500976
  .
  .
  .
  0.990545258721667
  0.9968056552600042
  0.99984209464165
  1
  */
```

이 `now`라는 인자 값을 활용하여 `transform`에도 애니메이션을 줄 수가 있다.

```
  $(selector).animate({
    opacity: 1
  }, {
    duration: 500,
    step: function (now){
      var x = now * 100,
          y = now * -50;
      $(this).css({
        transform: 'translate3d(' + x + 'px,' + y + 'px,0)'
      });
    }
  });
```

이를 CSS로 표현하면

```css
selector {transform:translate3d(100px,-50px,0); transition:transform cubic-bezier(.02,.01,.47,1) 500ms;}
/* jQuery.fn.animate 의 ease 기본값은 'swing' 이며, transition-timing-function 으로는 cubic-bezier(.02,.01,.47,1) 와 동일 */
```

을 적용한 것과 같다.

위와 같이 `opacity` 값이 0 ~ 1로 변하는 경우를 통해 인자 값을 전달하는 경우는 적용이 수월하지만, 아래와 같이 복잡한 상황도 생각해봐야 한다.

```css
selector{position:absolute;top:100px;left:0;width:500px;}
```
```
  $(selector).animate({
    top: '+=20',
    width: '300px'
  }, {
    duration: 500,
    step: function (now, fx) {
      console.log(now);
    }
  });

  /*
  실행 시 아래와 같이 값이 출력된다.
  100 // fx.prop === 'top'
  500 // fx.prop === 'width'
  100.03866390856828
  499.61336091431724
  .
  .
  .
  119.996841892833
  300.03158107167
  120
  300
  */
```

`top`의 값은 기존값에서 20만큼 더, `width` 값은 500px 에서 300px 으로 값이 변해야 하며, `step`을 통해 값을 내려받아야 하는데 `top`과 `width`를 두 속성을 애니메이션하면서 `now` 인자 값도 2개씩 호출하는 상황이 되어버렸다.

**`step`은 각 단계의 애니메이션에 있는 속성의 수만큼 이벤트를 발생**하기 때문에 인자 값을 활용하기 위해서는 약간의 수정이 필요하다.

```
  $(selector).animate({
    top: '+=20',
    width: '300px'
  }, {
    duration: 500,
    step: function (now, fx) {
      var x, y;
      if (fx.prop === 'top') { // top의 값에 대한 인자 값을 기준으로 처리
        x = (now - 100) * 5 // 100 ~ 120 인 수를 0 ~ 100 으로 치환
        y = (now - 100) * -2.5 // 100 ~ 120 인 수를 0 ~ -50 으로 치환
      }
      $(this).css({
        transform: 'translate3d(' + x + 'px,' + y + 'px,0)'
      });
    }
  });
```

위와 같이 `step`의 두 번째 인자를 통해 해당 값의 속성을 찾아서 계산식을 통해 값을 추출해낼 수 있다.

*(jQuery 1.8 버전 이상에서부터는 `progress` 속성이 추가되었으며, 이는 속성 수와 상관없이 한 번만 실행해주고 있으니 이를 활용해도 된다.)*

하지만 속성값이 계산하기 더 복잡해지면 그때마다 계산하기도 어려워지고, 다른 속성은 건드리지 않고 `transform`만을 애니메이션 시켜야 하는 경우도 있다.

`text-indent`, `border-spacing` 처럼 값이 변해도 레이아웃에 영향이 없는 경우를 가진 속성을 추가시켜 활용하는 방법도 있지만, 필요 없는 CSS를 추가로 이벤트 시키는 방법보다 좋은 방법이 있다.

### jQuery(Object) 활용

```
  $({val:0}).animate({
    val:100 // 0 ~ 100
  }, {
    duration: 500,
    step: function (now) {
      var y = -(now / 2); // 0 ~ -50
      $(selector).css({
        transform: 'translate3d(' + now + 'px,' + y + 'px,0)'
      });
      ...
    }
  });
```

**단일 숫자값를 가지는 속성을 가진 객체를 jQuery로 래핑**하여 `jQuery(Object)` 형태를 구성하고, 이에 `animate` 함수를 실행한다.

그리고 `step` 내에서 인자 값을 활용해 직접 셀렉터에 변화된 값을 설정해준다.

이처럼 `jQuery(Object)`를 활용하여 `jQuery.fn.animate`를 사용할 경우에는 사용하기 편리하게 인자 값을 자유롭게 설정해줄 수 있는 것이 큰 장점이다.

**[\>\> 활용예제](https://codepen.io/anon/pen/XxveXz){: target="_blank"}**

-----

## 마무리

`jQuery.fn.animate` 를 활용한 애니메이션을 여러 방향으로 구현해보며 자바스크립트로 애니메이션을 주는 것이 익숙해진다면, [TweenMax](https://greensock.com/tweenmax){: target="_blank"}, [CreateJs](https://www.createjs.com/){: target="_blank"} 등 여러 애니메이션 플러그인들을 사용함에 있어서도 이해하는데 도움이 되지 않을까 싶다.

-----

## 참고 자료

[\>\> CanIUse : css-transitions](https://www.caniuse.com/#feat=css-transitions){: target="_blank"}

[\>\> jQuery(Object) API](http://api.jquery.com/jquery/#jQuery-object){: target="_blank"}

[\>\> jQuery.fn.animate API](http://api.jquery.com/animate/#animate-properties-options){: target="_blank"}

