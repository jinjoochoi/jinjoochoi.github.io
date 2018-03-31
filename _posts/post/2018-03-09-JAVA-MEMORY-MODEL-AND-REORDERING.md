---
layout: post
title: "[번역] JAVA MEMORY MODEL AND REORDERING"
date:   2018-03-09
author : Jinjoo
post : true
---
<br/>

본 포스팅은 [JAVA MEMORY MODEL AND REORDERING]을 번역하여 작성했습니다.

<br/>

JAVA MEMORY MODEL AND REORDERING
================================

이 포스트는 double checked locking idiom을 해결하는 방법에 대한 또 다른 포스트가 아닙니다. 여기서 목표는 동기화 없이 무엇이 잘못될 수 있는지를 이해하는 것입니다.

Java Momory Model(JMM)의 [가장 중요한 약속들] 중 하나는 다음과 같습니다.
<br/>

> 프로그램이 올바르게 동기화된다면, 프로그램의 모든 실행이 순차적으로 일관되게 나타날 것입니다. 이것은 프로그래머에게 강력한 보증입니다. 프로그래머는 코드에 데이터 경합이 포함되어 있는지 판단하기 위해 순서를 변경하는 것에 대해 생각할 필요가 없습니다.
> 따라서 그들은 그들의 코드가 정확히 동기화되었는지를 판단할 때 순서를 변경하는 것에 대해 고려할 필요가 없습니다. 한번 이 코드가 올바르게 동기화된다는 결정을 내리면,
> 프로그래머는 순서 변경이 그녀 혹은 그의 코드에 영향을 미칠 것인지 걱정할 필요가 없습니다.

<br/>

Java Concurrency in Practice에서 가져온 다음 코드는 공유 변수 resource에 대한 접근이 완벽하게 동기화되지 않아 Thread safe 하지 않습니다.

{% highlight ruby %}
public class UnsafeLazyInitialization {
    private static Resource resource;

    public static Resource getInstance() {
        if (resource == null) //1
            resource = new Resource();  //2
        return resource; //3
    }
}
{% endhighlight %}

멀티 스레드 환경에서 innocent한 부분이 실행되었을 때 발생할 수 있는 문제들에 대해서 설명하겠습니다.

+ resource는 한 번 이상 초기화 될 수 있습니다.
+ getInstance는 일관되지 않은 상태로 객체를 리턴할 수 있습니다.
+ 더 흥미롭게도, getInstance는 null를 리턴할 수 있습니다.

<br/>

다중 인스턴스화
-----------
이것은 가장 분명한 이슈입니다 - Check-then-Act 패턴을 사용하는 코드는 두 스레드가 메서드에 동시에 도착하여 resource가 null임을 확인하고 변수를 초기화를 할 가능성이 있습니다. 어떻게 싱글톤이 두 개의 인스턴스를 가질 수 있을까요.

<br/>

부적절하게 생산된 객체
-------------------
이는 덜 분명한 사실입니다. (2)는 atomic으로 보이지만 JVM의 요구 사항과 다릅니다.
1. 메모리를 할당합니다.
2. 새로운 객체를 생성합니다.
3. 객체의 필드를 default값으로 설정하여 초기화를 합니다. (boolean은 false, 다른 primitive type은 0, 객체는 null)
4. 부모 생성자를 호출하는 것도 포함한 생산자를 run시킵니다.
5. resource에 새롭게 생성된 객체의 레퍼런스를 대입합니다.

동기화가 없기 때문에, JMM은 사실상 JVM이 이 과정을 아무 순서대로 수행하는 것을 허용합니다.
[double checked locking에 대한 유명한 토론 의 예제]를 참조하세요. 이는 JIT 컴파일러가 4단계 이전에 5단계를 실행하는 것을 보여줍니다. 그래서 getInstance는 null이 아니지만 일관되지 않은 객체의 레퍼런스도 리턴할 수 있습니다 (초기화되지 않은 필드와 함께).

<br/>

getInstance는 null을 리턴할 수 있다
-------------------------------

이는 훨씬 덜 분명합니다. 이러한 간단한 코드에서 null을 리턴할 수 있는 실행 경로를 상상하는 것은 어렵습니다. 그러나 JMM은 이를 허용합니다. 왜 이것이 가능한지 이해하려면 우리는 읽기와 쓰기를 자세하게 분석하고 그들 사이에 전후 관계가 있는지 평가할 필요가 있습니다. 읽기와 쓰기를 명확하게 보여주기 위해 다음과 같이 다시 작성할 수 있습니다:


| **Thread 0** |
| <span style="color:#f60;">10</span> : resource = null; &nbsp; &nbsp; //default value | | //write |
| **Thread 1** | **Thread 2** |
| 11: a = resource; | <span style="color:#f60;">21</span>: x = resource; | //read |
| 12: if (a == null) | 22: if (x == null) |
| <span style="color:#f60;">13</span>:    resource = new Resource(); | 23:    resource = new Resource();| //write |
| 14: b = resource;       | <span style="color:#f60;">24</span>: y = resource; | //read |
| 15: return b; | 25: return y; |

<br/>


[JLS #17.4.5]는 읽기가 쓰기를 관찰할 수 있음을 허용하는 규칙을 제시했습니다:
> 변수 v 의 읽기 r 이 execution trace의 전후 관계의 부분 순서에 있는 경우, v 에 대한 쓰기 w 를 관찰할 수 있다고 말한다.
> + r 은 w 이전에 배치되지 않고 ( hb(r, x)의 경우가 아니다. ),
> + 쓰기 w' 는 v 에 개입하지 않습니다. ( hb(w, w')와 hb(w', r)같은 v 에 대한 쓰기 w' 는 없다. )

<br/>

그러므로 이 예제에서, <span style="color:#f60;">21</span>과 <span style="color:#f60;">24</span>는 <span style="color:#f60;">10</span> 또는 <span style="color:#f60;">13</span>을 관찰하는 것이 허용되며 프로그램의 올바른 실행은 다음과 같습니다.
(스레드 1이 resource가 null인 것을 확인하고 초기화할 것임을 가정합니다.)

 1. 21: x = not null (reads the write line 13)
 2. 22: false
 3. 24: y = null (reads the write line 10)
 4. 25: return null

<br/>

명령어 재배치
----------
예제에서, T2는 null이 아닌 값을 확인한 이후 null 값을 확인할 수 없지만, 컴파일러 또는 JVM 또는 JIT는 유사한 실행을 생성하는 방식으로 명령어들을 재배치할 수 있습니다. 예를 들어, 가능한 재배치(이론적인 실행과 같이)는 다음과 같습니다.

{% highlight ruby %}
public class UnsafeLazyInitialization {
    private static Resource resource;
    public static Resource getInstance() {
        Resource temp = resource; //null in T1 and T2
        if (resource == null) //null in T1 but not in T2 because it has been initialised by T1 in the meantime
            resource = temp = new Resource(); //only executed by T1
        return temp; //T1 returns the new value, T2 returns null
    }
}
{% endhighlight %}
이 재배치는, 비록 말이 안 되지만, 스레드 내부의 의미에 영향을 주지 않기 때문에 완벽하게 유효합니다. (싱글 스레드 환경에서 실행되었다면, 원래의 코드와 똑같은 결과를 보여줬을 겁니다.)

<br/>

결론
---

이 예제는 상당히 부자연스러운 예에서 부적절하게 동기화된 프로그램의 결과가 상당히 놀랄 수 있음을 보여줍니다. 실제로 어떤 컴파일러가 재배치를 수행할 것 같지는 않지만, 조금 더 복잡한 상황이 되면 분석이 불가능해질 수 있습니다.


[JAVA MEMORY MODEL AND REORDERING]: https://assylias.wordpress.com/2013/02/01/java-memory-model-and-reordering/

[가장 중요한 약속들]: https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5-410

[JLS #17.4.5]: https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5-500

[double checked locking에 대한 유명한 토론 의 예제]: http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html
