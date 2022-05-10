# Design Pattern



## Decorator

> 객체에 행위나 역할을 부여하는 방식으로 기능을 확장하는 패턴

* 기존의 기능확장개념 : 상속 

  * 자바스크립트에서는 `Class.call(this)`

* 데코레이터 패턴의 기능확장 

  * 어노테이션을 제공하는 경우(컴파일언어) : 클래스에 어노테이션을 명시하여 역할/행위를 부여한다.

    * 컴파일타임에 그 어노테이션이 정의하는 역할에 대한 기능 및 멤버가 추가된다.(리플렉션)

  * 스크립트언어의 경우(자바스크립트) : 객체를 함수의 인수로 넘기고, 함수 내부에서 역할/행위를 부여하는 멤버를 객체에 추가한다.

    * ex)

    * ```
      function Espresso(){
      	this.cost = 1500;
      }
      function Water( espresso ){
      	espresso.cost = espresso.cost + 500;
      	espresso.water = 250;
      	return espresso; //for chanining
      }
      var americano = Water(new Espresso());
      ```

    * 아메리카노는 Espresso에 water속성을 250ml 추가(Water가 역할을 부여)한 결과이다.

* 상속을 통한 확장은 엄격하게 클래스로 구분하여 각 클래스로 인해 생성하는 객체가 갖는 역할 및 행위가 달라지는 반면, 데코레이터패턴을 이용한 기능확장은 한 클래스에 역할을 유연하게 부여하므로써 유연하게 기능을 확장할 수 있다.

