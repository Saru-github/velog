<h2 id="db-mysql--인덱싱">DB (MySQL) &amp; 인덱싱</h2>
<ul>
<li>이번주차에는 나의 콘서트 예약 앱에서 자주 조회하는 쿼리, 복잡한 쿼리를 파악 후에 동시 다발적인 요청이 들어 왔을 때, 어떻게 효과적으로 처리 할 수 있을지 고민 하는 시간을 가진다.</li>
<li>이때, 가장 효과적으로 수행시간을 줄일 수 있는 쿼리 Index 에 대해 알아보고, 직접 적용 후 실행시간을 비교 해 보기로 한다.</li>
</ul>
<h3 id="나의-db-환경">나의 DB 환경</h3>
<ul>
<li>Test 및 로컬 DB: H2</li>
<li>개발 및 운영 DB: MySQL</li>
<li>대기열 큐: Redis</li>
</ul>
<p>다음과 같이 운용할 계획이기 때문에 MySQL 을 기준으로 작성하도록 한다.</p>
<h3 id="1-mysql-주요기능---mvcc">1. MySQL 주요기능 - MVCC</h3>
<p>innoDB를 채택한 MySQL 버전의 경우 MVCC 기능을 이용 할 수 있다.
mvcc 개념은 6주차에 기록했던 내용을 첨부한다.</p>
<h4 id="mvcc-multi-version-concurrency-control">MVCC (Multi-Version Concurrency Control)</h4>
<blockquote>
<ul>
<li><code>개념</code>: MVCC는 다중 버전 동시성 제어로, 트랜잭션 간의 데이터 충돌 없이 데이터의 여러 버전을 동시에 유지하여 읽기 작업과 쓰기 작업의 충돌을 피하는 방법.</li>
<li><code>구현</code>: MVCC에서는 데이터를 변경할 때, 기존 데이터를 즉시 삭제하거나 덮어쓰지 않고 새로운 버전을 생성.
각 트랜잭션은 자신의 작업이 시작된 시점의 데이터 버전을 유지하면서 작업을 진행.
이를 통해 읽기 트랜잭션과 쓰기 트랜잭션이 동시에 수행될 수 있으며, 읽기 작업은 자신의 트랜잭션 시점의 데이터를 볼 수 있어 충돌이 발생하지 않는다.</li>
<li><code>특징</code>:<ul>
<li>Read Consistency: 트랜잭션이 시작된 시점의 데이터 버전을 사용해 일관된 결과를 제공한다.</li>
<li>성능 향상: 읽기와 쓰기를 구분해 충돌을 방지함으로써 트랜잭션의 동시성을 크게 개선할 수 있다.</li>
<li>Garbage Collection: 오래된 데이터 버전은 이후에 삭제되거나 정리.</li>
</ul>
</li>
<li><code>장점</code>: <ul>
<li>읽기 트랜잭션이 락 없이 작업할 수 있어 성능이 높고, 높은 동시성을 제공. </li>
<li>데이터 충돌이 적은 경우 성능 향상이 크며, 읽기 작업이 많은 시스템에서 매우 유용하다.</li>
</ul>
</li>
<li><code>단점</code>: 데이터 버전을 관리하는 오버헤드가 발생하며, 자주 변경되는 데이터에 대해 오래된 버전을 유지하면 스토리지 사용이 증가한다.</li>
</ul>
</blockquote>
<h3 id="2-pk-vs-index">2. PK vs Index</h3>
<p>이전가지 막연히 PK 지정시 Index 가 생성되며, PK 를 이용한 조회조건 설정 시 쿼리 성능이 향상 된다고만 알고 있었다. PK 지정을 통한 Index와  임의로 생성한 Index 와 어떤차이와 장단점이 있는지 확인 해 보았다.</p>
<blockquote>
<p>INDEX?</p>
</blockquote>
<ul>
<li>추가적인 쓰기 작업과 저장 공간을 활용하여 데이터베이스 테이블의 검색 속도를 향상시키기 위한 자료구조</li>
<li>데이터를 조회하는 SELECT 외에도 UPDATE나 DELETE의 성능이 함께 향상 됨<blockquote>
</blockquote>
</li>
<li>장점*<blockquote>
<ul>
<li>테이블을 조회하는 속도와 그에 따른 성능을 향상시킬 수 있다.</li>
</ul>
</blockquote>
</li>
<li>전반적인 시스템의 부하를 줄일 수 있다.<blockquote>
</blockquote>
</li>
<li>단점*</li>
<li>인덱스를 관리하기 위해 DB의 약 10%에 해당하는 저장공간이 필요하다.</li>
<li>인덱스를 관리하기 위해 추가 작업이 필요하다.</li>
<li>인덱스를 잘못 사용할 경우 오히려 성능이 저하되는 역효과가 발생할 수 있다.<blockquote>
</blockquote>
</li>
<li>Index를 활용하면 좋을 경우* </li>
<li>규모가 작지 않은 테이블</li>
<li>INSERT, UPDATE, DELETE가 자주 발생하지 않는 컬럼</li>
<li>JOIN이나 WHERE 또는 ORDER BY에 자주 사용되는 컬럼</li>
<li>데이터의 중복도가 낮은 컬럼</li>
</ul>
<blockquote>
<p>PK 의 특징</p>
</blockquote>
<ul>
<li>PK는 레코드의 저장 위치를 결정한다.</li>
<li>PK는 NOT NULL과 유니크 속성을 가진다.</li>
<li>MySQL에서는 PK 기준으로 유사한 값들이 묶여 저장되며, 이를 클러스터링이라 부른다.</li>
<li>일반적으로 PK는 클러스터 인덱스라고 불린다.</li>
<li>레코드를 저장하거나 PK를 변경할 때 처리 속도가 느려질 수 있다.</li>
<li>일반 인덱스는 논클러스터 인덱스로 불린다.</li>
<li>PK가 레코드의 물리적 저장 위치를 결정하기 때문에 인덱스는 PK에 의존한다.</li>
</ul>
<blockquote>
<p>클러스터링 Index(PK) 의 장단점</p>
<p><em>장점</em></p>
</blockquote>
<ul>
<li>PK로 검색할 때 처리가 매우 빠름</li>
<li>연속되는 PK로 조회할 경우 랜덤 I/O가 아닌 순차 I/O를 사용하여 처리 속도가 더욱 빠름</li>
<li>인덱스가 PK값을 가지므로 인덱스로 PK 값만 조회하는 경우 효율적으로 처리될 수 있음(=커버링 인덱스)<blockquote>
<p><em>단점</em></p>
</blockquote>
</li>
<li>모든 인덱스가 PK에 의존하므로 PK 값이 클 경우 전체적으로 인덱스의 크기가 커지고, 페이지 양이 많아짐</li>
<li>인덱스를 통해 검색할 때 PK로 다시 한번 검색해야 하므로 처리 성능이 느림</li>
<li>INSERT 시에 PK에 의해 레코드의 저장 위치가 결정되기 때문에 처리 성능이 느림</li>
<li>PK를 변경할 때 레코드를 DELETE 및 INSERT 해야 하므로 처리 성능이 느림</li>
</ul>
<hr />
<h3 id="3-추가학습">3. 추가학습</h3>
<ul>
<li>Index 종류와 동작방식
<a href="https://mangkyu.tistory.com/286">https://mangkyu.tistory.com/286</a></li>
<li>Index 기반으로 걸리는 Lock 동작
<a href="https://mangkyu.tistory.com/298">https://mangkyu.tistory.com/298</a></li>
</ul>
<hr />
<h3 id="4-그럼-나의-쿼리에서-index-를-적용할-곳을-찾아보자">4. 그럼 나의 쿼리에서 index 를 적용할 곳을 찾아보자.</h3>
<p>현재 내 프로젝트는 대기열구현은 redis queue 형식으로 구현 하며 쿼리를 덜어 냈기 때문에, 쿼리수가 크게 많지 않다.</p>
<p>크게보자면</p>
<p>1 유저(포인트)</p>
<ul>
<li>유저 포인트 조회</li>
<li>충전, 차감용 포인트 조회 (for update)</li>
</ul>
<p>2 콘서트</p>
<ul>
<li>콘서트 일자조회</li>
<li>콘서트 좌석조회</li>
<li>콘서트 예약건 조회</li>
<li>콘서트 예약건 만료 조회</li>
</ul>
<p>이정도의 쿼리 정도만 조회된다.</p>
<h3 id="1-유저포인트">1. 유저(포인트)</h3>
<ul>
<li>유저 포인트를 조회하는 select 쿼리와, update 시 락을 이용하는 쿼리 두가지로 분기 되어있다.</li>
<li>jpa 를 통한 쿼리 조회시 다음과 같다.<pre><code class="language-java">select
   u1_0.user_id,
   u1_0.balance,
   u1_0.created_at,
   u1_0.updated_at,
   u1_0.user_name,
   u1_0.version 
from
   `user` u1_0 
where
   u1_0.user_id=?

</code></pre>
</li>
</ul>
<p>select
      u1_0.user_id,
      u1_0.balance,
      u1_0.created_at,
      u1_0.updated_at,
      u1_0.user_name,
      u1_0.version 
  from
      <code>user</code> u1_0 
  where
      u1_0.user_id=? for update        </p>
<pre><code>해당 user 테이블의 pk 가 이미 userId 이기 때문에,
이 userId 값으로 조회하게 되었을 때는, 이미 유니크한 값을 가지게 된다. 또한 충전, 사용, 조회는 한 사용자가 자신의 포인트를 조회 하는 것으로 아주 빈번하게 까지는 사용하지 않을 것이라 판단하였다. 따라서 따로 index 추가를 안하여도 PK index만으로 빠른 조회가 가능 할 것이다.

### 2. 콘서트 
#### 1.콘서트 스케쥴 조회
```java
select
        cs1_0.concert_schedule_id,
        cs1_0.concert_date,
        cs1_0.concert_id,
        cs1_0.created_at,
        cs1_0.updated_at 
    from
        concert_schedule cs1_0 
    where
        cs1_0.concert_id=?</code></pre><p>JPA 코드상 다음과 같은 쿼리가 발생하게 된다.
만약 운영 상황이라고 생각한다면, 얼마나 많은 콘서트를 관장하게 될 지는 모르겠으나, 날짜의 한계가 있기 때문에 한콘서트당 많은 row가 발생하지 않을 것이다.</p>
<p>다음은 30개의 콘서트가 365일간 개최된다고 가정했을 때의 위의 쿼리의 실행 계획 및 실행속도이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/saruru/post/e223d4ad-0f1d-47b1-ab98-a7180c71d6c5/image.png" /></p>
<p><img alt="" src="https://velog.velcdn.com/images/saruru/post/8cb0b276-87b2-47ca-9c18-6fcad9a9b78b/image.png" /></p>
<p>물론 지난 데이터도 있을 것이지만, 콘서트가 365일 내내 개최된다는 것도 상상하기 어렵기 때문에, 큰 부하가 있을 것이라고 생각되지 않는다. 따라서 별도의 index를 걸어 줄 필요는 없을 것이라고 판단하였다.</p>
<h4 id="2콘서트-예약가능-좌석-조회">2.콘서트 예약가능 좌석 조회</h4>
<pre><code class="language-java">select
        cs1_0.concert_seat_id,
        cs1_0.concert_schedule_id,
        cs1_0.created_at,
        cs1_0.price,
        cs1_0.seat_number,
        cs1_0.status,
        cs1_0.updated_at 
    from
        concert_seat cs1_0 
    where
        cs1_0.concert_schedule_id=?
    and      
        cs1_0.status=?
</code></pre>
<p>먼저 운영시 생성 될만한 데이터를 생각해보자.
한 콘서트당 정말 월드스타라면.. 최대 좌석의 수는 약 10만개 라고 생각했다. 하지만 테스트의 편의를 위해 15만으로 설정 했고, 거기에 개최되는 콘서트의 수는 10개라고 가정하고(실제로 훨씬 많은 날짜가 생성될 것이다.) , 여기서 좌석의 상태는 현재는 총 3가지 밖에 존재 하지 않는다. 일단 row 수도 크게 늘어 날 수 있고, 예약가능 좌석은 경쟁적으로 계속 조회 되기 때문에 index 설정이 필수라고 느껴졌다. </p>
<p>10개의 콘서트 날짜, 3개의 좌석상태 상태당 5만 row 삽입</p>
<p>풀스캔
<img alt="" src="https://velog.velcdn.com/images/saruru/post/19b7c660-199a-4a19-b5d2-7a2c9f3cb288/image.png" />
(무슨일인지 150만개가 아닌, 175만개가 들어가고 말았다)</p>
<p>concert_schedule_id 단일 index
<img alt="" src="https://velog.velcdn.com/images/saruru/post/4ae42a51-5253-4d04-9726-af5afe4ea783/image.png" /></p>
<p>status 단일 index
<img alt="" src="https://velog.velcdn.com/images/saruru/post/f9d78258-24ee-49cc-a1f6-d8d59db5662f/image.png" /></p>
<p>concert_schedule_id + status 복합 index
<img alt="" src="https://velog.velcdn.com/images/saruru/post/60908241-1eee-400f-b3dd-f9b6ccb2fc33/image.png" /></p>
<p>다만 처음에는 복합index 로 concert_schedule_id + stauts 로 걸어야 한다고 생각하였으나, 결과적으로는 단일 index 로 concert_schedule_id 만 index 설정을 하였다. 상태값은 3개 뿐이라, 카디널리티가 낮을 것으로 판단되며, 이것을 모든 로우에서 복합 index 로 조회할 경우, 단일 index로 스케쥴만 조회 했을 때 보다 불리함이 있을 수 있을 것이라 판단했다. 특히 한 콘서트당 좌석의수는 10만개 보다 적을 것이며, 이때 콘서트의 스케쥴 갯수만 늘어 날 것이기 때문에, concert_schedule_id를 단일키로 잡는편이 효율면에서 합리적인 선택이라고 생각하였다.</p>
<h3 id="3-콘서트-예약">3. 콘서트 예약</h3>
<pre><code class="language-java">select
        cs1_0.concert_seat_id,
        cs1_0.concert_schedule_id,
        cs1_0.created_at,
        cs1_0.price,
        cs1_0.seat_number,
        cs1_0.status,
        cs1_0.updated_at 
    from
        concert_seat cs1_0 
    where
        cs1_0.concert_seat_id=? for update</code></pre>
<p>예약시 좌석을 가져오는 쿼리또한 PK를 이용한 쿼리기 때문에, 따로 Index 설정을 해주지 않아도 될 것이다.</p>
<h3 id="4-콘서트-예약건-만료-조회">4. 콘서트 예약건 만료 조회</h3>
<pre><code class="language-java">Hibernate: 
    select
        cb1_0.concert_booking_id,
        cb1_0.concert_seat_id,
        cb1_0.created_at,
        cb1_0.expired_at,
        cb1_0.status,
        cb1_0.updated_at,
        cb1_0.user_id,
        cb1_0.version 
    from
        concert_booking cb1_0 
    where
        cb1_0.status=? 
        and cb1_0.expired_at&lt;?</code></pre>
<p>상태가 예약되었지만, 만료시간이 초과한 예약건에 대하여 만료하는 스케쥴 로직이다.
<img alt="" src="https://velog.velcdn.com/images/saruru/post/55876652-9fa8-46e3-8629-d9462a493b8c/image.png" />
한가지 신기한점은, status 로 걸었을 때 index 가 동작하였으나, 부등호로 구했을 때는 index가 작동하지 않았다. 인덱스 설정이 부등호나 날짜에 상당히 민감한걸로 아는데, 일단 status 로 단일 인덱스를 구현하고, 나머지 에 대해서는 추후 공부 하도록 하겠다. 일단 status로 index 를 설정해도 성능상 큰 무리가 없어보이고, 스케쥴러 이기 때문에 다음 스케쥴러에 방해만 안될 정도면 된다고 생각한다.실제 해당 로직은 몇초 안되는 시간에 돌 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/saruru/post/eeacbcb4-9c1c-4eca-a26b-ee73e416e932/image.png" />
<img alt="" src="https://velog.velcdn.com/images/saruru/post/aade527b-c0a9-4aa0-ad69-7ae1a876934a/image.png" />
<img alt="" src="https://velog.velcdn.com/images/saruru/post/2cdb926c-0631-4df7-bcd5-ddab4ca7f473/image.png" /></p>
<h3 id="3줄요약">3줄요약</h3>
<ul>
<li>대부분 pk index를 사용하여 조회하도록 구현되어 있기때문에 크게 바꾸지 않아도 됐다.</li>
<li>최대 서버 부하를 생각했을때, 콘서트 좌석, 삭제 스케쥴러 조회에 index를 부여</li>
<li>날짜 부등호에 대한 추가 학습 및 index 적용 방법 고려가 필요 할 것같다.</li>
</ul>
<p>감사합니다. (급마무리)</p>