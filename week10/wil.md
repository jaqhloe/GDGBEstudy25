**Concurrency Control(동시성 제어)**

여러 사용자가 동시에 데이터베이스에 접근해서 트랜잭션을 수행할 때 데이터의 일관성, 무결성 유지하는 기법

**Optimistic Concurrency Control**

충돌이 거의 일어나지 않을 것으로 생각해서 먼저 실행 후 나중에 검사(validate)

락을 사용하지 않고 작업 후 데이터베이스에 반영(Commit)하기 직전에 다른 트랜잭션에 의해 데이터가 변경되었는지 확인

충돌을 감지하면 수행했던 작업을 취소(rollback)

읽기가 흔히 일어나는 경우 락으로 인한 오버헤드가 적어지지만 쓰기 충돌이 많은 경우 롤백과 재시도로 인한 비용 오버헤드가 생길 수 있다

3가지 단계(phase)를 거쳐서 수행:

1. 읽기
    
    트랜잭션의 독립적인 공간에서 다른 트랜잭션이 보지 못하게 읽기 수행
    
    이후 read set이 알려지고 작성하려는 write set도 공개
    
2. 검증(validation)
    
    전 단계에서 결정된 read와 write set이 직렬성을 침해하지 않는지 충돌 검사
    
    (ACID의 성질이 유지되는지 확인)
    
    - out-of-date transaction을 읽는 경우
    - read phase 동안 다른 트랜잭션이 변경한 Read set
    - Backward-oriented: 기존 완료 트랜잭션과 충돌 확인
    - Forward-oriented: 현재 validation 중인 트랜잭션과 충돌 확인
3. 쓰기
    
    위의 단계들을 모두 문제 없이 수행했다면 private context를 데이터베이스의 상태에 반영할 수 있다
    

**Pessimistic Concurrency Control** 

충돌이 자주 일어난다고 생각해서 문제가 일어나는 것을 방지하는 것에 초점

데이터에 접근하기 전에 먼저 lock을 획득해서 다른 트랜잭션의 접근 차단

데이터의 일관성이 보장

충돌이 적을 경우에는 오버헤드 발생 가능성, 전체적인 성능(throughput)이 안 좋아진다

교착 상태(deadlock) 발생할 수 있다

- Timestamp ordering(lock-free)
    
    선행하는 타임스탬프가 커밋되었을 때만 트랜잭션의 연산을 수행
    
    읽을 때는 max_write timstamp 이후만 읽기 허용
    
    작성할 때는 max_read_timestamp 이전으로는 불가
    
    다만 max_write_timestamp 이전으로 작성은 가능하다(무시될 것이라서)
    

**Multiversion Concurrency Control(MVCC, 다중 버전 동시성 제어)**

→ Locking, scheduling, conflict resolution techniques, timestamp ordering

→ snapshot isolation 구현하는데 활용

- 데이터를 수정할 때 덮어쓰는 게 아니라 새로운 버전의 데이터를 생성
- transaction ID, timestamp 등으로 관리

가장 최신에 commit된 버전이 current으로 취급

트랜잭션 매니저는 매 순간 최대 1개만의 uncommitted value가 되도록 관리

시스템의 격리성 레벨에 따라서 읽기 단계에서 커밋되지 않은 값에 대한 접근 여부 결정

**Lock-Based Concurrency Control(잠금 기반 동시성 제어, Pessimistic)**

contention(경합), scalability issue 등이 발생할 수 있다

대표적으로 **two-phase locking(2PL)**이 존재

아래의 과정을 진행하는 중에서 data 읽거나 쓰는 것은 배제하지 않는다

2PL은 오직 잠금의 획득과 해제 시점에만 관심이 있다(보수적 2PL 제외)

- growing(expanding) phrase
    
    트랜잭션에 필요한 모든 락을 획득, 획득한 잠금은 해제할 수 없다
    
- shrinking phrase
    
    앞선 단계에서 획득한 락들을 해제하는 과정
    
    → 한번이라도 락을 해제했다면 새로운 잠금을 획득할 수 없다
    

**Deadlocks**

서로의 락을 기다리다가 영원히 실행할 수 없는 교착상태

- timeout으로 해소
- conservative 2PL은 연산 전에 모든 잠금을 획득하도록 설정
- 주기적 혹은 항상 Deadlock을 감지하는 시스템을 사용할 수 있다
- transaction timestamp로 우선순위 설정
- wait-die
    
    더 이후의 타임스탬프가 있는 트랜잭션이 lock 해제할 때까지 기다림
    
- wound-wait
    
    Lower timestamp인 트랜잭션이 막을 수 있다
    

**Locks, Latches**

- Logical data integrity → locks
    
    tree implementation보다 상위 개념인 database lock manager가 관리
    
- physical data integrity → latches
    
    트리가 수정되는 와중에 구조를 유지하기 위해 사용
    

**Readers-writer lock(RW lock)**

read끼리는 겹쳐도 상관 없으나 문제가 되는 읽기와 쓰기 혹은 쓰기와 쓰기가 겹치는 것 방지

**Latch crabbing(coupling)**

latch를 획득하기 위한 Optimistic 방법

- read path
    
    child node의 latch 위치를 알고 획득시 부모의 Latch도 획득 가능
    
- insert
    
    자식 노드가 꽉 차지 않는 경우 parent latch release
    
- delete
    
    자식 노드로 인한 Merge가 발생하지 않을 경우 부모 래치 접근 가능
    

**Blink-Trees**

B* tree 기반으로 high key, slbling link pointer를 사용하는 구조

Half-split이라는 특별한 상태 존재
