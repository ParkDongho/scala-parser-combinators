## Getting Started

스칼라 구문 분석기 결합기는 일상적인 프로그램에서 사용할 수 있는 구문 분석기를 구축할 수 있는 강력한 방법입니다. 하지만 구성 요소와 시작 방법을 이해하기는 어렵습니다. 처음 몇 개의 샘플을 컴파일하고 작동시키면 배관 구조가 이해되기 시작합니다. 하지만 그 전까지는 어려울 수 있으며 표준 문서도 큰 도움이 되지 않습니다(일부 독자는 구문 분석기 결합기에 대한 원래의 "예제별 Scala" 챕터와 이후 개정판에서 이 챕터가 어떻게 사라졌는지 기억하실 것입니다). 그렇다면 구문 분석기의 구성 요소는 무엇일까요? 이러한 구성 요소는 어떻게 서로 맞을까요? 어떤 메서드를 호출할까요? 어떤 패턴을 일치시킬 수 있을까요? 이러한 부분을 이해하기 전까지는 문법 작업을 시작하거나 추상적인 구문 트리를 구축 및 처리할 수 없습니다. 따라서 복잡성을 최소화하기 위해 가능한 가장 간단한 언어인 소문자로 시작하고자 합니다. 이 언어에 대한 파서를 만들어 봅시다. 문법을 하나의 제작 규칙으로 설명할 수 있습니다:

```
word -> [a-z]+
```

구문 분석기의 모습은 다음과 같습니다: 


    import scala.util.parsing.combinator._
    class SimpleParser extends RegexParsers {
      def word: Parser[String]    = """[a-z]+""".r ^^ { _.toString }
    }


[scala.util.parsing.combinator](https://javadoc.io/static/org.scala-lang.modules/scala-parser-combinators_2.13/2.1.0/scala/util/parsing/combinator/index.html) 패키지에는 흥미로운 것들이 모두 들어 있습니다. 우리 구문 분석기는 어휘 분석을 수행하기 때문에 [RegexParsers](https://javadoc.io/static/org.scala-lang.modules/scala-parser-combinators_2.13/2.1.0/scala/util/parsing/combinator/RegexParsers.html)를 확장합니다. 
`""""[a-z]+"""".r`은 정규식입니다. `^^`는 "함수 적용을 위한 파서 결합기"로 [문서화](https://javadoc.io/static/org.scala-lang.modules/scala-parser-combinators_2.13/2.1.0/scala/util/parsing/combinator/Parsers$Parser.html#^^[U](f:T=>U):Parsers.this.Parser[U])되어 있습니다. 기본적으로 `^^` 왼쪽의 구문 분석이 성공하면 오른쪽의 함수가 실행됩니다. yacc 구문 분석을 수행했다면, `^^`의 왼쪽은 문법 규칙에 해당하고 오른쪽은 규칙에 의해 생성된 코드에 해당합니다. "word" 메서드는 문자열 타입의 파서를 반환하므로, `^^` 오른쪽의 함수는 문자열을 반환해야 합니다.

그렇다면 이 파서를 어떻게 사용할까요? 문자열에서 단어를 추출하고 싶으면


    SimpleParser.parse(SimpleParser.word(myString))

다음은 이를 위한 간단한 프로그램입니다.

    object TestSimpleParser extends SimpleParser {
      def main(args: Array[String]) = println(parse(word, "johnny come lately"))
    }


여기서 주목해야 할 두 가지가 있습니다:
 
* 객체가 SimpleParser를 확장합니다. 따라서 모든 앞에 "SimpleParser"를 붙여야 하는 번거로움을 피할 수 있습니다.
* 이 함수를 실행하면 구문 분석한 "단어"를 반환하는 것이 아니라 `ParserResult[String]`를 반환합니다. "단어"라는 이름의 메서드가 `Parser[String]` 유형의 결과를 반환하고 유형 매개 변수가 `ParseResult`로 전달되기 때문에 "String" 유형 매개 변수가 필요합니다.

프로그램을 실행하면 콘솔에 다음과 같은 내용이 표시됩니다:


    [1.7] parsed: johnny 


즉, 구문 분석기와 일치하는 입력의 첫 번째 문자가 위치 1에 있고 일치해야 할 나머지 첫 번째 문자가 위치 7에 있습니다. 이것은 좋은 시작이지만, 이 모든 것은 우리가 ParseResult를 가지고 있지만 원하는 것, 즉 단어가 없기 때문에 무언가를 놓치고 있음을 시사합니다. 우리는 `ParserResult`를 더 잘 처리해야 합니다. `ParseResult`에서 "get" 메서드를 호출할 수 있습니다. 그러면 결과를 얻을 수 있지만 모든 것이 작동하고 구문 분석이 성공적이라는 낙관적인 가정을 하는 것입니다. 우리는 입력이 유효한지 알 수 있을 정도로 입력을 제어할 수 없기 때문에 그렇게 계획할 수 없습니다. 입력은 우리에게 주어졌고 우리는 그것을 최대한 활용해야 합니다. 즉, 오류를 감지하고 처리해야 하는데, 이는 패턴 매칭을 위한 작업처럼 들리죠? 스칼라에서는 예외를 가두기 위해 패턴 매칭을 사용하고, 성공과 실패를 분기하기 위해 패턴 매칭(`옵션`)을 사용하므로 파싱을 처리할 때도 패턴 매칭을 사용할 것으로 예상할 수 있습니다. 그리고 실제로 다양한 종료 상태에 대해 `ParseResult`에서 패턴 매칭을 할 수 있습니다. 다음은 더 나은 작업을 수행하는 작은 프로그램을 다시 작성한 것입니다:


    object TestSimpleParser extends SimpleParser {
      def main(args: Array[String]) = {    
        parse(word, "johnny come lately") match {
          case Success(matched,_) => println(matched)
          case Failure(msg,_) => println("FAILURE: " + msg)
          case Error(msg,_) => println("ERROR: " + msg)
        }
      }
    }


`Option`이 기본적으로 두 가지 경우(Some과 None)를 가지는 것에 비해 `ParseResult`는 기본적으로 세 가지 경우를 가집니다: 1) `성공`, 2) `실패`, 3) `실패`. 각 케이스는 두 개의 항목으로 구성된 패턴으로 일치합니다. `성공`의 경우 첫 번째 항목은 구문 분석기가 생성한 객체("단어"는 `Parser[String]`를 반환하므로 문자열)이며, `실패` 및 `오류`의 경우 첫 번째 항목은 오류 메시지입니다. 모든 경우에서 일치 항목의 두 번째 항목은 일치하지 않는 나머지 입력이며, 여기서는 상관하지 않습니다. 하지만 복잡한 오류 처리나 후속 구문 분석을 수행한다면 세심한 주의를 기울일 것입니다. 실패`와 `오류`의 차이점은 `실패`의 경우 구문 분석이 계속될 때 구문 분석이 역추적되는 반면(이 규칙이 작동하지 않았지만 다른 문법 규칙이 있을 수 있음), `오류`의 경우 치명적이며 역추적이 없다는 것입니다(구문 오류가 있고 이 언어의 문법과 제공한 식을 일치시킬 방법이 없으므로 식을 편집하고 다시 시도하십시오).

이 작은 예제는 필요한 파서 결합기 배관을 많이 보여줍니다. 이제 나머지 배관 중 일부를 보여주기 위해 약간 더 복잡한(물론 인위적인) 예제를 살펴 보겠습니다. 우리가 실제로 찾고자 하는 것이 단어 뒤에 숫자가 오는 것이라고 가정해 봅시다. 이것이 긴 문서에서 단어의 빈도 수에 대한 데이터라고 가정해 보겠습니다. 물론 간단한 정규식 일치를 통해 이 작업을 수행하는 방법도 있지만, 좀 더 추상적인 접근 방식을 취해 결합기 배관을 좀 더 보여드리겠습니다. 단어 외에도 숫자를 일치시켜야 하고, 단어와 숫자를 함께 일치시켜야 할 것입니다. 먼저 단어와 개수를 모으는 새로운 유형을 추가해 보겠습니다. 다음은 이를 위한 간단한 케이스 클래스입니다:


    case class WordFreq(word: String, count: Int) {
      override def toString = "Word <" + word + "> " +
                              "occurs with frequency " + count
    }

이제 구문 분석기가 '문자열'의 인스턴스가 아닌 이 케이스 클래스의 인스턴스를 반환하기를 원합니다. 기존 구문 분석의 맥락에서 문자열과 숫자와 같은 원시 객체를 반환하는 프로덕션은 어휘 분석(일명 토큰화, 일반적으로 정규 표현식을 사용)을 수행하는 반면, 복합 객체를 반환하는 프로덕션은 추상 구문 트리(AST)를 생성하는 것에 해당합니다. 실제로 아래의 수정된 구문 분석기 클래스에서는 단어와 숫자를 정규식으로 인식하고 단어 빈도는 고차 패턴을 사용합니다. 따라서 문법 규칙 중 두 개는 토큰화를 위한 것이고 세 번째 규칙은 AST를 구축하는 것입니다:
 
    class SimpleParser extends RegexParsers {
      def word: Parser[String]   = """[a-z]+""".r       ^^ { _.toString }
      def number: Parser[Int]    = """(0|[1-9]\d*)""".r ^^ { _.toInt }
      def freq: Parser[WordFreq] = word ~ number        ^^ { case wd ~ fr => WordFreq(wd,fr) }
    }

그렇다면 이 새로운 프로그램에서 주목해야 할 점은 무엇일까요? '숫자'에 대한 구문 분석기는 '단어'에 대한 구문 분석기와 거의 비슷해 보이지만, '파서[문자열]`이 아닌 '파서[Int]`를 반환하고 변환 함수가 'toString'이 아닌 'toInt'를 호출한다는 점을 제외하면 다릅니다. 하지만 여기에는 세 번째 생산 규칙인 freq 규칙이 있습니다. 

* 정규식이 아니므로 .r이 없습니다(결합기입니다).
* '파서[WordFreq]`의 인스턴스를 반환하므로 `^^` 연산자의 오른쪽에 있는 함수는 복합 유형 `WordFreq`의 인스턴스를 반환하는 것이 좋습니다.
* "단어" 규칙과 "숫자" 규칙을 결합합니다. 이 연산자는 `~`(물결표) 결합자를 사용하여 "먼저 단어를 일치시킨 다음 숫자를 일치시켜야 한다"고 말합니다. 물결표 결합기는 정규식이 포함되지 않은 규칙에 가장 일반적으로 사용되는 결합기입니다.
* 규칙의 오른쪽에 패턴 일치를 사용합니다. 때때로 이러한 일치 표현식은 복잡하지만 왼쪽에 있는 규칙의 메아리일 뿐인 경우가 많습니다. 이 경우 실제로는 규칙의 여러 요소에 이름을 지정하여(이 경우 "wd" 및 "fr") 해당 요소에 대해 작업할 수 있도록 하는 것뿐입니다. 이 경우 이름이 지정된 요소를 사용하여 관심 있는 객체를 구성합니다. 하지만 패턴 일치가 왼쪽의 메아리가 아닌 경우도 있습니다. 이러한 경우는 규칙의 일부가 선택 사항일 때 또는 일치해야 할 매우 구체적인 경우가 있을 때 발생할 수 있습니다. 예를 들어, fr이 정확히 0인 경우에 특별한 처리를 수행하고자 한다면 대소문자를 추가할 수 있습니다:
* 
```
case wd ~ 0
```

다음은 이 구문 분석기를 사용하기 위해 약간 수정한 프로그램입니다:
 
    object TestSimpleParser extends SimpleParser {
      def main(args: Array[String]) = {
        parse(freq, "johnny 121") match {
          case Success(matched,_) => println(matched)
          case Failure(msg,_) => println("FAILURE: " + msg)
          case Error(msg,_) => println("ERROR: " + msg)
        }
      }
    }

이 작은 프로그램과 이전 프로그램 사이에는 단 두 가지 차이점이 있습니다. 두 가지 차이점은 모두 세 번째 줄에 있습니다:

* "단어" 구문 분석기를 사용하는 대신 "빈도" 구문 분석기를 사용하는데, 이는 입력에서 얻으려는 객체의 종류이기 때문입니다.
* 입력 문자열을 새 언어와 일치하도록 변경했습니다.

이제 프로그램을 실행하면 다음과 같은 결과를 얻습니다:

     Word <johnny> occurs with frequency 121
 
이 시점에서 파서 결합기 배관을 충분히 보여드렸으니 이제 시작해서 유용한 작업을 해보시기 바랍니다. 이제 다른 모든 문서가 훨씬 더 이해가 되셨기를 바랍니다.
