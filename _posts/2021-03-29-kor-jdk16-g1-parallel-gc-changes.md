---
layout: post
title:  "[번역] JDK 16 G1/Parallel GC 변경 사항들"
date:   2021-03-12 16:39:24 +0200
#categories: gc g1 parallel JDK-16 performance
tags: [GC, G1, Parallel, JDK 16, Performance]
---

이 글은 원저자이자 팀 동료인 Thomas Schatzl의 동의하에 한국어 번역을 것입니다.
원문은 [여기](https://tschatzl.github.io/2021/03/12/jdk16-g1-parallel-gc-changes.html) 입니다.

이 포스트는 JDK 16 Hotspot의 stop-the-world 가바지 컬렉터인 G1과 Parallel GC의 가장 중요한 변경 사항을 요약합니다.

첫째, 모든 GC 서브 컴포넌트의 간단한 개요: **ZGC**와 관련된 [JEP 376](https://openjdk.java.net/jeps/376)이 유일하다 할 수 있습니다. 변경 내용은 쓰레드-스택 처리를 stop-the-world pauses에서 concurrent하게 바꾸었다는 것인데, 이로 인해 pause가 1 밀리 세컨드 (1ms)이하로 됩니다. 그 외에 **G1**과 **Parallel GC**도 흥미로운 소소한 업데이트가 있습니다.

전체 Hotspot GC 서브 컴포넌트의 변경 사항 목록은 [여기](https://bugs.openjdk.java.net/issues/?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20resolution%20%3D%20Fixed%20AND%20fixVersion%20%3D%20%2216%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc%2C%20gc%2C%20gc%2C%20gc%2C%20gc)%20ORDER%20BY%20key%20ASC)에 있으며, 총 325 개입니다.

## Parallel GC

  * [JDK-8252221](https://bugs.openjdk.java.net/browse/JDK-8252221)의 변경 사항은 커밋된 **메모리의 pre-touch 병렬화**입니다. 이는 자바 힙을 많이 설정하고 `-XX:+AlwaysPreTouch` 옵션을 활성화 했을 경우 스타트업 시간을 단축 시킵니다. 또한 [JDK-8254699](https://bugs.openjdk.java.net/browse/JDK-8254699)과 후속 작업들 (G1도 해당 코드 공유)에서는 X64 및 ARM64 플랫폼에서의 작업 패킷 크기에 대한 개선도 있습니다.

  * [JDK-8247820](https://bugs.openjdk.java.net/browse/JDK-8247820)는 자바 힙에 대한 **내부 레퍼런스들** 처리시 **병렬화** 했다는 것인데, 경우에 따라 Parallel과 G1 컬렉터([JDK-8247819](https://bugs.openjdk.java.net/browse/JDK-8247819))에서 약간의 pause time을 줄여줍니다.

## G1 GC

  * 지금까지는 GC pause중에 자바 힙을 필요이상으로 소유하고 있다고 판단되고, 또한 사용자가 허용(힙의 최소 / 최대값이 다른 경우)한다면, 그 잉여분 메모리를 해당 pause에서 반환(uncommit)했습니다. 반환해야 할 메모리의 크기 (경우에 따라 GB 단위의 메모리도 흔함)에 따라 해당 pause는 꽤 길어졌습니다 (테스트에서는 50~100ms 이상). [JDK-8236926](https://bugs.openjdk.java.net/browse/JDK-8236926)는 이것을 바꾸었습니다. 해당 pause에서는 어느 메모리의 어느 만큼을 반환 할 지를 정하고, **실제 반환은 concurrent phase로 연기**해서 백그라운드로 진행합니다. 이 백그라운드 작업은 점진적(incrementally)으로 진행합니다. GC pause를 시작하기 위한 대기를 피하기 위해, 각각의 증가분은 수 밀리 초(ms)만 걸리도록 크기가 조절됩니다. 

    로그 채널중 `gc+heap`를 디버그 레벨로 설정하면 됩니다. 반환할 메모리의 크기, 각 단계에서의 소요 시간 등 G1이 수행한 작업 내용들을 볼 수 있습니다. 보다 자세한 내용은 [릴리즈 노트](https://jdk.java.net/16/release-notes#JDK-8236926)를 참고하시기 바랍니다.

  * [JDK-8240556](https://bugs.openjdk.java.net/browse/JDK-8240556)는 G1이 humongous 객체들을 eager reclamation (역자주: 기존 동작 방시과는 달리 좀 더 미리/열심히 humongous 객체들을 회수한다는 의미인데 원어 그대로 사용하겠습니다.)(JDK 9 에 추가된 기능 [JDK-8048179](https://bugs.openjdk.java.net/browse/JDK-8048179))할 수 있을때 **concurrent 마킹의 시작 시기를 계산할 때의 단점**(역자주: concurrent 마킹 - G1의 GC 단계중 하나, 자바 프로그램과 동시에 실행되며, 객체들을 확인-마킹하는 절차)을 해결 합니다. G1은 GC가 시작될 때, 현재 힙 점유율이 내부 임계값(역자주: InitiatingHeapOccupancyPercent)보다 큰지 아닌지 여부에 따라 concurrent 마킹을 준비해야 할 지 아닌지를 결정합니다. 만약 그럴 경우, G1은 GC중 몇 가지 추가 작업을 수행하여 그 concurrent 마킹을 준비합니다. 이전의 계산은 특정 GC중에 힙 점유율이 그 내부 임계값 이하로 다시 떨어질 수 있다는 것을 고려하지 않았습니다. 이럴 경우, **G1은 이제 해당 마킹을 취소**하고, 기존에 했던 작업들 일부를 취소하기 위한 준비를 합니다(물론 concurrent하게). 이것은 concurrent 마킹을 수행하고 필요 없어진 mixed collection(역자주: 영제네레이션과 올드제네레이션을 컬렉팅)을 수행하는 것보다 훨씬 빠르며 CPU도 덜 소비합니다.
  
  * 유사하게 JDK-8245511](https://bugs.openjdk.java.net/browse/JDK-8245511)는 **언제 마킹을 시작할 지에 대한 힙 점유율 임계값을 정하는 부분**에 대한 것을 개선했습니다. 이 힙 점유 임계값은 최근 올드 제네레이션에서의 메모리 할당율과 마킹을 시작한 후에 첫번째 올드 제네레이션을 컬렉팅 할 때까지의 시간을 사용해서 계산됩니다. 메모리 할당율이 높을 수록 및 / 또는 마킹 시작부터 올드 제네레이션의 첫번째 컬렉션까지의 시간이 길 수록 해당 임계점은 낮아 집니다. 즉, G1은 마킹을 일찍 시작합니다.
  
    Humongous 객체들은 항상 올드 제네레이션에 직접 할당되기에 문제이며 이 메모리 할당률에 크게 영향을 미칩니다. 그러나, JDK 9 이후로, 위에서 언급한 eager reclamation를 통해 humongous 객체를 매우 빠르게 회수 할 수 있으며, 올드 제네레이션에서의 전체 할당률은 실제로 매우 낮습니다. 이것은 불필요한 concurrent 마킹을 야기했습니다. 지금까지 언급한 JDK-8245511에 대한 변경 사항은 eager reclamation를 고려하기에 이 문제(불필요한 마킹)를 해결합니다. 변경 사항의 효과를 단적으로 보여주는 그래프가 [버그 레포트](https://bugs.openjdk.java.net/browse/JDK-8245511)에 있습니다.

## 기타 주목할 만한 변경 사항들

추가로, 중요하지만 사용자들에게 거의 혹은 전혀 보이지 않을 JDK 16 변경 사항들도 있습니다.

  * JDK 16 은 **객체 고정 지원** (참조 [JDK-8236594](https://bugs.openjdk.java.net/browse/JDK-8236594)) 을 위한 몇가지 사전 작업, 단일 쓰레드에서 객체가 이동하지 않아야 하는 경우(대부분 JNI를 통해 GC되지 않는 언어와의 상호작용), 메모리가 더 필요하기 때문에 GC를 시작하기를 바라는 자바 어플리케이션 쓰레드들을 막기보다는 G1은 고정된 객체를 제외하고 GC를 실행하게 됩니다. 
  
    A. Shipilev의 [글](https://shipilev.net/jvm/anatomy-quarks/9-jni-critical-gclocker/)에서 위에서 언급한 문제와 그것을 처리 할 수있는 방법들에 대해 설명합니다. 이 글에 따르면, 이 패치(JDK-8236594)는 G1 을 옵션 1 (GC를 완전히 비활성화)에서 옵션 3 (객체를 포함하는 부분 공간의 고정)(역자주: G1은 힙을 여러개의 힙 영역 나눠서 관리하는데, 특정 힙 영역 고정)으로 변경합니다.

    [JDK-8253600](https://bugs.openjdk.java.net/browse/JDK-8253600) 와 [JDK-8253081](https://bugs.openjdk.java.net/browse/JDK-8253081)는 그 준비 과정들의 예입니다. G1의 영역 기반 객체 고정 기능을 활성화하는 것은 **누락**되었습니다. 활성화 시키는 것은 [몇 줄의 코드](https://github.com/openjdk/jdk/compare/master...tschatzl:full-pin-support) 에 불과 하지만, 동작에 미치는 영향은 아직 완전히 조사되지 않았습니다. 일부 초기 테스트에서 영향 범위를 알 수 없는 몇 가지 잠재적인 문제가 나타났으며, [잠재적인 해결책](https://bugs.openjdk.java.net/issues/?jql=labels%20%3D%20gc-g1-pinned-regions)에 대해 브레인 스토밍했습니다. 이 문제에 대해 조사해보고 싶으시거나, 해결책을 시도해보고 싶으신 분은 언제든 저에게 연락 주세요. [contact me](https://tschatzl.github.io/about/)

## 향후 계획

우리는 Oracle LTS 릴리스가 될 JDK 17 에 집중하고 있습니다. 다음은 여러분이 관심을 가질 만한 간단한 목록입니다. 그리고 이것은 현재 계획이지 꼭 포함된다는 보장은 없습니다. 이러한 기능은 개발이 완료되어야 릴리스에 포함될 것입니다. ;-)

  * JDK 17 은 G1의 Java 힙이 아닌 메모리 사용, 특히 **remembered set 메모리 사용량**을 줄이기 위한 최적화에서 큰 개선을 할 수 있습니다. (역자주: remembered set은 힙 영역 내에 있는 객체가 다른 힙 영역에 있는 객체를 참조할 경우 표시 해놓는 내부 데이터) 이것에 대한 [포스트](https://tschatzl.github.io/2021/02/26/early-prune.html) 를 참조하세요. 변경 사항 ([JDK-8262185](https://bugs.openjdk.java.net/browse/JDK-8262185)) 은 변경에 따른 영향을 설명을 포함하고 있고 이미 메인라인에 푸쉬되었습니다. 또한, remembered set 메모리 사용량을 훨씬 더 줄일 수 있도록 오래 작업해온 변경 사항들이 있습니다. 계속 지켜봐 주세요!
  
  * Microsoft 는 G1 에서 **GC를 사전에 스케줄링**해서 evacuation 실패를 포함하는 **매우 긴 GC를 방지** 하는 매우 흥미로운 기능을 개발 중입니다. [JDK-8257774](https://bugs.openjdk.java.net/browse/JDK-8257774)

  * Hamlin Li 는 [JDK-8262068](https://bugs.openjdk.java.net/browse/JDK-8262068) 에서 간단한 heuristic을 통해 **G1 Full Collections을 향상** 시켰습니다: 만약 특정 힙 영역이 거의 꽉 찼을 경우, 컴팩트 하려 하지 않는 것입니다. (역자주: 거의 꽉 찼다는 것은 살아있는 객체가 많을 가능성이 높고, 이런 힙 영역을 컴팩트 한다 한들 효율은 낮을 것입니다. 그러므로 그 힙 영역은 그냥 건너뛰는게 나을 것이다라는 것이 요지입니다) 다른 가비지 컬렉터들은 이미  `-XX:MarkSweepDeadRatio` 옵션을 활용해 구현했습니다. G1에도 이런 기능을 넣는다면 좋겠지요. 현재 리뷰 중입니다.

## 감사합니다…

JDK 릴리스에 기여해주신 모든 분들께 감사드립니다. 다음 릴리스에서 만나요. :)
