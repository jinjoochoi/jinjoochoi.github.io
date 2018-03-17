---
layout: post
title: "[번역] Thanks for the memory, Linux (1. Introduction)"
date:   2018-03-16
author : Jinjoo
---
<br/>

본 포스팅은 [Thanks for the memory, Linux] 을 번역하여 작성했습니다.

<br/>
### Windows와 Linux에서 JVM이 어떻게 native memory를 사용하는 방법 이해


모든 Java 객체가 할당되는 Java 힙은 Java 애플리케이션을 작성할 때 가장 밀접하게 연결되는 메모리 영역입니다. JVM은 우리를 호스트 머신의 특성으로부터 보호하기 위해 설계되었기 때문에, 당신이 메모리를 생각할 때 heap을 떠올리는 것은 자연스러운 것 입니다. 당신은 객체 누수(object leak) 혹은 heap에 담기에는 너무 큰 데이터를 할당하려고 해서 생기는 Java heap OutOfMemoryError를 마주한 적이 있을겁니다. 그리고 아마 이런 시나리오에서 디버깅 하는 데 필요한 몇가지 트릭을 배웠을 겁니다. 그러나 Java 애플리케이션이 더 많은 데이터를 다루고 동시 로드(concurrent load)를 하면서, Java 힙이 가득 차 있지는 않지만 오류가 발생하는 시나리오에서는 당신의 트릭으로는 해결할 수 없는 OutOfMemoryErrors를 경험하게 될 수 있습니다. 이런 일이 발생할 때, 당신은 Java Runtime Environment (JRE)의 내부에서 일어나고 있는 것을 이해해야합니다.

Java 애플리케이션은 Java 런타임의 가상화된 환경(virtualized environment)에서 실행되지만, 런타임 자체는 native 메모리를 포함하는 native 자원을 사용하는 언어(C와 같은)로 작성된 native 프로그램입니다. native 메모리는 Java 애플리케이션이 사용하는 Java heap 메모리와 구분되는 런타임 프로세스에서 사용 가능한 메모리입니다. Java 힙 과 Java 스레드와 같은 모든 가상화 된 리소스는(virtualized resource) 가상 머신(virtual machine)이 실행될 때 사용되는 데이터와 함께 native 메모리에 저장되어야 합니다. 이는 호스트 머신의 하드웨어와 운영 체제 (OS)가 부과하는(impose) native 메모리 제한이 Java 애플리케이션이 수행되는 데에 영향을 미칠 수 있다는 것을 의미합니다.

이 기사는 다른 플랫폼에서 동일한 주제를 다루는 두 기사 중 하나입니다. 두 기사에서, 당신은 native 메모리가 무엇인지, Java 런타임이 어떻게 native 메모리를 사용하는지, 어떻게 실행되지 않는지, 어떻게 native OutOfMemoryError를 디버깅 하는지를 배우게 될 것입니다. 이 기사는 Windows 및 Linux를 다루며 특정 런타임 구현에 초점을 맞추지 않습니다. [companion 기사] 에서는 AIX에 대해 다루고 IBM Developer Kit for Java에 중점을 둡니다. (IBM 구현에 대한 기사의 정보는 AIX가 아닌 다른 플랫폼에서도 마찬가지입니다. 따라서 당신이 Linux 용 IBM Developer Kit 또는 Windows 용 IBM 32-bit Runtime Environment를 사용한다면, 기사가 유용 할 것입니다.)

[Thanks for the memory, Linux]: https://www.ibm.com/developerworks/library/j-nativememory-linux/
[companion 기사]: https://www.ibm.com/developerworks/java/library/j-nativememory-aix/
