---
layout: post
title: "[번역] Fastest Thread-safe Singleton in Java"
date:   2018-03-12
author : Jinjoo
---
<br/>

본 포스팅은 [Fastest Thread-safe Singleton in Java]을 번역하여 작성했습니다.

<br/>


최근에, 우리는 Java에서 thread-safe "초기화 지연(lazily initializing)" 싱글톤을 구현하는 여러가지 방법을 살펴 보았습니다. 가장 간단한 방법은 'synchroinzed'입니다. 그러나 몇은 옳고 몇은 옳지 않은 (double checked locking) 몇 가지 다른 제안이 있었습니다.

그러나 한가지 접근법은 25배 빠릅니다.

<br/>

# 지연 싱글톤

우리의 기본적인 요구 사항은 어플리케이션 내의 싱글 톤 (아마도 일부 서비스)을 지연 초기화(lazily initialized)하는 것입니다. 일반적으로 이것은 설정 비용이 많이 들고 가끔 필요한 서비스에 유용 할 수 있습니다.

우리는 여기서 설정 비용에 관심을 두지 않습니다. 이것은 서비스에만 해당됩니다. 우리가 관심을 갖고있는 것은 일단 그것이 설정되면 싱글 톤에 접근하는 (즉, thread-safe방식으로 얻는) 퍼포먼스 비용입니다.

<br/>

# 다양한 접근법

이 작업은 근본적으로 단순하지만 여러 가지 방법으로 다양합니다.
나의 첫 번째 접근 방식은 단순함이 좋다는 것입니다 - 언어가 제공하는 기능을 사용하고 메소드를 동기화합니다. 다른 사람들은 다양한 접근법을 제공했지만.. 모두가 반드시 이상적이거나 올바른 것은 아닙니다!

+ 'synchronized' method
+ AtomicReference fast-path before a ‘synchronized’ section
+ AtomicReference with a spinlock
+ double-checked locking  (Java에서는 reliable 하지 않습니다.)
+ 'volatile' field를 이용한 double-checked locking


'synchroinzed' 메소드를 사용하는 것은 확실히 간단하고 성능이 좋습니다.

AtomicReference는 'synchroinzed' 보다 기술적으로 더 간단하고 빨라서 잠재적으로 성능상의 이점을 제공 할 수 있습니다. 그러나 스핀 락(Spinlocks)은 피해야합니다.

대중화된 패턴이였던 Double-checked locking은 자바에서 레퍼런스가 저장될 때 생성자 코드가 잠재적으로 실행되지 않을 수 있기 때문에 안전하지 않습니다. JVM 최적화 및 코드 재배치는 "synchroinzed" 경계를 고려하도록 지정되었지만, double-checked locking은 이를 스킵합니다. 추천하지 않습니다. 그러나 1.5 이후에는 이 문제를 해결하기 위해서 'volatile' 필드를 사용할 수 있습니다.

<br/>

# '내부 클래스(Inner Class) 접근법

Java에서는 클래스 초기화가 '필요에 따른(on-demand)'으로, 클래스가 처음 사용될 때 수행됩니다. 보통, 이런 기본적인 행동은 별로 관심이 없습니다.. 하지만 우리가 그것을 사용할 수 있을까요?

여기서는 'holder'를 내부 클래스로 생성하여 singleton을 정적으로 초기화하는 방법을 사용합니다.
이 패턴은 "필요에 따른 초기화(initialization-on-demand holder)" 라고 알려져 있습니다.

{% highlight ruby %}
public class Example {
    private static class StaticHolder {
        static final MySingleton INSTANCE = new MySingleton();
    }

    public static MySingleton getSingleton() {
        return StaticHolder.INSTANCE;
{% endhighlight %}

getSingleton()을 호출하면 inner class를 참조해서, JVM이 로드 및 초기화를 합니다. Classloading에는 잠금(lock)이 사용되기 때문에 thread-safe합니다.

이후에 호출을 하면 JVM은 이미 로드된 내부 클래스를 resolve하고 기존의 싱글톤을 반환합니다. - 캐시.  JVM 최적화의 마법 덕분에 매우 효율적입니다.

<br/>

# 성능

Java에서의 성능 벤치마킹은 까다로운 분야로 - 웜업(warmup)과 안정적인 조건을 필요로 하며, JIT가 벤치 마크 전체를 최적화하지 않도록 주의해야 합니다.
이 벤치 마크의 경우 2만개의 웜업(wramup)루프를 사용해서 1,000만개의 루프를 측정했습니다. 테스트 코드가 최적화되는 것을 막기 위해, 반환된 싱글톤을 사용했습니다(그들의 hash-code를 합하여). 또한 루프 및 해시 코드 오버 헤드를 보여 주고 Java가 이 형식을 최적화하는 데 얼마나 효율적인지 정확하게 보여 주는 작업당 비용을 포함하여 업데이트했습니다.

수치는 다음과 같습니다.

<br/>

| **Technical Approach**   | &nbsp;&nbsp;**Total Time** | &nbsp;&nbsp;**Minus Overhead** | &nbsp;&nbsp;**Per Operation** |
| ‘synchronized’ method  | &nbsp;&nbsp;&nbsp;858 ms | &nbsp;&nbsp;834 ms | &nbsp;&nbsp;83.4 ns |
| double-checked locking, ‘volatile’ field  | &nbsp;&nbsp;&nbsp;39.27 ms | &nbsp;&nbsp;15.79 ms | &nbsp;&nbsp;1.58 ns |
| <span style="color:#f60;">**inner-class static init**</span>  | <span style="color:#f60;">&nbsp;&nbsp;&nbsp;**33.4 ms**</span> | <span style="color:#f60;">&nbsp;&nbsp;**9.92 ms**</span> | <span style="color:#f60;">&nbsp;&nbsp;**0.99 ns**</span> |
| loop & hashcode overhead  | | &nbsp;&nbsp;23.48 ms | &nbsp;&nbsp;2.35 ns |

<br/>

<br/>

25배나 더 빠릅니다!

JVM 덕분에 내부 클래스 참조, 클래스 로딩 및 스레드 안전성(thread-safety)이 모두 JIT로 갈 것입니다. CPU가 실행될 남아있는 것은 기본적으로 정적 필드에서 읽은 오는 메모리 뿐입니다.


2.4GHz CPU 테스트에서, 'synchronized'메서드는 - 스레드 경쟁(thread contention) 없이 - 206 사이클이 필요했습니다. 비교하자면, 루프 반복(loop iteration) 및 '내부 클래스(inner class)' 싱글톤은 단 8 CPU 사이클에 접근할 수 있었습니다.

이 패턴은 singleton 특정(singleton-specific)이고, map기반 캐시에는 도움이 되지 않습니다. 하지만 싱글톤 서비스의 경우에는 - 빠름일까요, 아니면 뭘까요! JVM개발자들과 이 기술을 내놓은 이들에게 칭찬을 보냅니다.

<br/>

이 접근법에 대해 어떻게 생각하세요? 지금 코멘트 달아주세요.

<br/>

<br/>

참고자료:
- [Java Concurrency blog:  Double-checked locking]
- [Wikipedia: Initialization-on-demand holder idiom]


[Fastest Thread-safe Singleton in Java]: http://literatejava.com/jvm/fastest-threadsafe-singleton-jvm/

[Java Concurrency blog:  Double-checked locking]: http://jeremymanson.blogspot.kr/2008/05/double-checked-locking.html

[Wikipedia: Initialization-on-demand holder idiom]: https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom
