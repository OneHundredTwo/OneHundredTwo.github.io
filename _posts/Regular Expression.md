# 정규표현식

## [A-Za-z]을 쓸 때 주의사항 

* https://stackoverflow.com/questions/29771901/why-is-this-regex-allowing-a-caret
* 오타시 발생할 수 있음
* `[A-z]` 로 표현하면 아스키코드 범위상 `[\]^_` ` 이 문자들이 포함된다. 

## Greedy and Lazy match

> <.+?> 을 쓰면 가장 짧게 일치하는 패턴을 리턴합니다.

* `.+`  : 바로 뒤에 특정 문자가 더이상 나타나지 않을 때까지 찾아서 리턴한다.
* `.+?` : 바로 뒤에 특정 문자가 나타나면 리턴한다.

수량표현식 (`* + {}`)은 욕심많은 연산자이기 때문에, 제공된 데이터에서 최대한 많이 매칭하려고 합니다.

예를 들어, `<.+>`는 `This is a **<div> simple div</div>** test` 문장에서 `<div>simple div</div>` 를 매칭시킵니다. 만약 `div` 태그만 매칭하려면 뒤에 `?`를 사용하여 게으르게 만들 수 있습니다.

```
<.+?>        < 와 > 사이에 하나 이상의 문자와 매칭하고, 가능한 짧게 매칭합니다.
             -> 실습해보세요!
```

더 나은 방법은, `.` 연산자를 사용하지 않고 좀더 엄격한 정규식을 사용하는 것입니다.

```
<[^<>]+>     < 와 > 사이에 <,>가 아닌 모든 문자가 하나 이상인 문자와 매칭합니다
             -> 실습해보세요!
```



## Multiline comment Regex

* https://www.oreilly.com/library/view/regular-expressions-cookbook/9781449327453/ch07s06.html
  * `/\*[\s\S]*?\*/` 
  * RegExp 테스트 사이트 등에선 되는데, VS Code에선 안됨.
* https://www.codegrepper.com/code-examples/whatever/Regex+to+match+a+multiline+comment
  * `((\/)([\*][\*]?)(.|\n)*?\3\2)|(\/\/.*)`  =>  `((\/)([\*][\*]?)(.|\n)*?\3\2)|(\/\/.*\/)`  
  * 한땀한땀 깎은건데, 정확도는 좀 떨어져도( `//";` 같은 패턴을 포함함 ) VS Code에서 multiline comment를 잡는다.
