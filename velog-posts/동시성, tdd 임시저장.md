<h2 id="추가-과제---step2-동시성-제어-방식에-대한-분석-및-보고서-작성">추가 과제 - STEP2. 동시성 제어 방식에 대한 분석 및 보고서 작성</h2>
<h3 id="동시성-제어-방식-분석-보고서">동시성 제어 방식 분석 보고서</h3>
<h4 id="1-개요">1. 개요</h4>
<p>Java의 멀티스레딩 환경에서 동시성 문제는 흔히 발생하는 과제입니다.<br />여러 스레드가 동시에 동일한 자원에 접근할 경우, 자원의 일관성을 유지하기 위해 적절한 동시성 제어가 필요합니다.
Java의 전통적인 synchronized 키워드와 ReentrantLock을 사용한 동시성 제어 방식을 비교하고, 각 방식이 멀티스레드 환경에서 어떻게 동작하는지
CompletableFuture를 사용한 비동기 요청으로 테스트 하였습니다.</p>
<h4 id="2-synchronized를-이용한-동시성-제어">2. synchronized를 이용한 동시성 제어</h4>
<h5 id="21-synchronized-메서드">2.1 synchronized 메서드</h5>
<pre><code class="language-java">public synchronized UserPoint charge(long id, long amount);
public synchronized UserPoint use(long id, long amount);</code></pre>
<p>synchronized 키워드는 Java에서 기본적으로 동시성을 제어하기 위해 제공되는 방법입니다.
해당 키워드를 사용하면 동일한 자원에 대해 동시에 여러 스레드가 접근하지 못하도록 막을 수 있습니다.
아래는 CompletableFuture를 사용해 비동기적으로 charge와 use 메서드를 호출한 예입니다.</p>
<pre><code class="language-java">// given
CompletableFuture.allOf(
        CompletableFuture.supplyAsync(() -&gt; pointService.charge(1L, 1000L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.charge(1L, 1000L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.use(1L, 500L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.use(1L, 500L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.charge(1L, 1000L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.charge(1L, 1000L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.use(1L, 500L)).thenAccept(System.out::println),
        CompletableFuture.supplyAsync(() -&gt; pointService.use(1L, 500L)).thenAccept(System.out::println)
).join();</code></pre>
<p><img alt="" src="https://velog.velcdn.com/images/saruru/post/7107e537-56ca-4e1d-b325-f1f32b012000/image.png" /></p>
<p>테스트 결과 순차적으로 요청 되었으나, 테스트의 마지막 CHARGE 인걸로 봐서, 순차적으로 처리 되지 않은 것으로 판단됩니다. </p>
<p>이 방식에서는 동일한 자원에 대한 동시 접근을 방지하지만, 
실행 순서가 보장되지 않습니다. 즉, 어떤 스레드가 먼저 자원을 획득할지 예측할 수 없으므로, 순차적인 결과를 기대하기 어렵습니다.</p>
<p>또한 두명이상의 유저가 접근 하였을 때도 문제가 발생합니다.
다음은 유저2, 유저3 이 각각 교차로 충전과 사용을 비동기적으로 요청한 테스트 코드입니다.</p>
<pre><code class="language-java">CompletableFuture.allOf(
                CompletableFuture.supplyAsync(() -&gt; pointService.charge(2L, 1000L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.charge(3L, 1000L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.use(2L, 500L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.use(3L, 500L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.charge(2L, 1000L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.charge(3L, 1000L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.use(2L, 500L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result)),
                CompletableFuture.supplyAsync(() -&gt; pointService.use(3L, 500L)).thenAccept(result -&gt; System.out.println(&quot;Result: &quot; + result))
        ).join();</code></pre>
<p><img alt="" src="https://velog.velcdn.com/images/saruru/post/4b5d9b15-7652-4a4e-81fd-958444a5dd61/image.png" /></p>
<p>2명이상의 유저가 접근 될 경우, 제대로 무결성을 보장하지 못하는 것을 알 수 있습니다.</p>
<h5 id="22-문제점">2.2 문제점</h5>
<ul>
<li>순서 보장 불가: 여러 스레드가 경쟁적으로 자원을 요청할 때, 자원 획득 순서가 보장되지 않으며, 이는 예상하지 못한 결과를 초래할 수 있습니다.</li>
<li>공정성 부족: synchronized는 스레드가 자원을 획득하는 순서를 제어하지 않으므로, 기아 상태(starvation)가 발생할 수 있습니다. 즉, 특정 스레드가 자원을 오랜 시간 동안 획득하지 못하는 상황이 발생할 수 있습니다.</li>
<li>범위 설정: lock 방식의 비해 범위를 설정하는 선택의 수가 적습니다. (메서드 단위, 변수 단위 등)</li>
</ul>
<h4 id="3-reentrantlock--concurrenthashmap을-이용한-동시성-제어">3. ReentrantLock + ConcurrentHashMap을 이용한 동시성 제어</h4>
<h5 id="31-reentrantlock과-concurrenthashmap">3.1 ReentrantLock과 ConcurrentHashMap</h5>
<p>ReentrantLock은 synchronized보다 더 유연한 동시성 제어 메커니즘을 제공합니다. 
특히, 공정성(fairness) 옵션을 통해 더 오래 대기한 스레드가 먼저 자원을 획득할 수 있도록 할 수 있습니다. 
또한, ConcurrentHashMap과 결합하여 각 사용자별로 고유한 락을 부여함으로써 사용자별로 동시성을 제어할 수 있습니다.</p>
<pre><code class="language-java">    // 각 회원 id 에 따른 동시성 보장
    ReentrantLock lock = userMap.computeIfAbsent(id, userId -&gt; new ReentrantLock());
    lock.lock();

    try {
        // amount 검증
        long validChargeAmount = pointValidator.verifyChargeAmount(amount);
        // 검증된 amount 를 통한 충전 이후 포인트 총합 계산
        long calculatedChargePoint =  defaultUserPointRepository.selectById(id).calculateChargePoint(validChargeAmount);
        // totalPoint 검증
        long validTotalPoint = pointValidator.verifyChargePoint(calculatedChargePoint);

        // 최종 검증된 포인트로 업데이트
        UserPoint returnUserPoint = defaultUserPointRepository.insertOrUpdate(id, validTotalPoint);

        // 포인트 히스토리 기록
        defaultPointHistoryRepository.insert(id, amount, TransactionType.CHARGE, System.currentTimeMillis());

        return returnUserPoint;

    } finally {
        lock.unlock();
    }</code></pre>
<p>이 방식은 각 사용자 ID 마다 락을 설정하여 동일한 사용자의 포인트 작업에 대해서만 동시성을 제어하고, 다른 사용자 작업은 병렬로 처리할 수 있도록 합니다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/saruru/post/8a8bfff1-604c-41c3-b467-7cf759405fbd/image.png" /></p>
<p>유저 1에 대한 충전, 사용이 순차적으로 이루어 지고 있음을 알 수 있습니다.</p>
<p><img alt="업로드중.." src="blob:https://velog.io/9650b3fd-ed28-4f84-b377-5c8cdc58f2ae" /></p>
<p>유저 2,3 의 요청의 완료 시간은 비동기 적이나, 히스토리를 열람했을때 순차적으로 잘 들어 간것이 확인됩니다.</p>
<h5 id="32-장점">3.2 장점</h5>
<p>공정성 보장: ReentrantLock은 오래 기다린 스레드가 먼저 자원을 획득할 수 있도록 공정성을 보장할 수 있습니다.
명시적인 동시성 제어: lock()과 unlock()을 사용하여 더 유연하게 동시성 제어가 가능합니다.
순서 보장 가능: 사용자별로 락을 관리함으로써 순차적 실행을 강제할 수 있습니다. 이는 synchronized보다 더욱 세밀한 동시성 제어가 가능합니다.</p>
<h5 id="4-정리">4 정리</h5>
<p>자원의 순차적인 사용, 공정성, 설정 범위를 고려 하였을 때,
지금 작성중인 코드에는 ReentrantLock + ConcurrentHashMap을 사용한 동시성 제어 방식이 적합하다고 느껴 채택 하게 되었습니다. 보통 Transaction 같은 DB 기반의 동시성 제어를 많이 접했던 것 같은데, JAVA 코드레벨 적으로 동시성을 제어 하여 멀티 쓰레드 환경에도 순차적으로 동시성을 보장할 수 있는 방법을 터득한 계기가 되었던 것 같습니다.</p>
<hr />
<h4 id="커밋태그">커밋태그</h4>
<table>
<thead>
<tr>
<th align="left">태그</th>
<th align="left">설명</th>
</tr>
</thead>
<tbody><tr>
<td align="left">feat</td>
<td align="left">새로운 기능 추가</td>
</tr>
<tr>
<td align="left">fix</td>
<td align="left">버그 수정</td>
</tr>
<tr>
<td align="left">refactor</td>
<td align="left">코드 리팩토링</td>
</tr>
<tr>
<td align="left">comment</td>
<td align="left">주석 추가(코드 변경 X) 혹은 오타 수정</td>
</tr>
<tr>
<td align="left">docs</td>
<td align="left">README와 같은 문서 수정</td>
</tr>
<tr>
<td align="left">merge</td>
<td align="left">merge</td>
</tr>
<tr>
<td align="left">rename</td>
<td align="left">파일, 폴더 수정, 삭제 혹은 이동</td>
</tr>
<tr>
<td align="left">report</td>
<td align="left">해당 주차별 과제 등록 및 풀이</td>
</tr>
<tr>
<td align="left">chore</td>
<td align="left">설정 변경</td>
</tr>
</tbody></table>