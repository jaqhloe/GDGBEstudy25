**Recovery** 

DB는 여러 개의 하드웨어와 소프트웨어를 기반으로 작동하기 때문에 안정성과 신뢰성을 보장하는 방법이 필요하다. 

**WAL(write-ahead log, commit log)**: 추가만 가능한(append only) 보조적인 디스크 기반 구조로 페이지 캐시가 디스크로 내려갈 때까지 변경 내용을 기록해둔다. 

주요 기능: 

- 페이지 캐시가 디스크에 저장된 페이지에 반영될 때까지 버퍼링 가능
- 변경 내용에 대한 기록이 실제 반영되기 전에 디스크에 저장
- 메모리에서 사라진 내용들을 기록을 바탕으로 재구성하는데 도움을 준다

**Log Semantics**

WAL은 append-only 구조이므로 변경할 수 없으며 순차적이다. 따라서 복구를 위해 활용할 때에는 어느 시점을 기준으로 그 이후를 참고하면 되는 편리함이 있다.

내부의 각 레코드는 단조 증가하는 숫자인 LSN(log sequence number)를 가진다. 

로그 기록들은 디스크 블록 전체를 필요로 하지 않기 때문에 내용이 로그 버퍼에 캐시되고 force operation으로 LSN 순서 그대로 디스크에 기록된다. 
이 동작은 로그의 버퍼가 가득찼을 때 발생하며, transaction manager나 page cache의 요청으로 발생한다. 이때 LSN의 커밋 기록 없이는 커밋되지 않은 것으로 간주된다.

일부 시스템은 CLR(compensation log records)를 Undo 과정에서 사용해 복구를 돕는다.

checkpoint가 존재해 더 이상 필요하지 않은 log를 trimming 한다.

또한 한번에 전부 다 디스크에 기록하는 건 비효율적이므로 fuzzy checkpoint라는 개념으로 
마지막 저장된 체크포인트는 last_checkpoint, 시작과 끝 부분은 begin_checkpoint, end_checkpoint 등 포인터로 관리한다.

**Operation Versus Data Log**

shadow paging: 

새로 기록된 정보는 unpublished shadow page, 이후 Pointer flip으로 가시화해서 업데이트 하는 방식

before-image와 after-image로 Undo/Redo를 편리하게 구현

- Physical logging: 변경되어야할 전체 페이지 상태나 바이트 단위 변경 사항 → redo
    
    before/after image를 저장하고 있다.
    
- Logical logging: 적용되어야할 operation → undo

**Steal and Force Policies**

page cache에 대해 적용하는 경우가 많으나 recovery의 맥락에서 주목할만하다.

- steal/no-steal: 트랜잭션에 의해 변경된 페이지를 트랜잭션 커밋 이전에 디스크에 기록하는 것을 허용/비허용
    - steal dirty page: 메모리의 내용을 다른 내용을 디스크에서 로딩하는 동시에 기록하는 것
- force/no-force: 트랜잭션이 커밋되기 전에 모든 변경된 페이지들을 디스크에 기록할 것을 강제/강제안함
    - force a dirty page: 커밋 전에 디스크에 기록

트랜잭션 복구의 관점에서 보면

- Undo: rolls back updates to forced pages for committed transactions
- Redo: applies changes perforemd by committed transactions on disk
- no-steal policy: redo만 사용해서 복구
- No-force: 메모리에 캐시해두고 디스크를 복구
- force: 이미 디스크에 기록되어 있으므로 추가로 작업 필요가 없지만 읽고 쓰는 비용이 더 많다.

**ARIES(Algorithm for Recovery and Isolation Exploiting Semantics)**

steal/no-force, physical redo, logical undo, WAL record, undo 중 compensatoin log record 작성

LSN으로 레코드 구분, fuzzy check-pointing 활용

3단계의 Recovery process:

1. analysis phase:
    
    dirty page와 문제 발생 시점에 진행 중이던 트랜잭션을 찾아낸다.
    
    dirty page의 정보로 redo할 지점을 설정하고 트랜잭션의 정보는 undo 시점에 활용한다.
    
2. Redo phase:
    
    충돌 시점 까지의 활동을 반복하고 이전의 상태로 복구한다. 
    
    imcomplete transaction 뿐만 아니라 디스크에 기록되지 않은 commit만 된 것도 대상에 해당한다.
    
3. undo phase:
    
    가장 최근의 consistent state로 복구하기 위해 imcomplete transaction을 역순으로 Roll back 한다.
    

**Concurrency Control**

트랜잭션 매니저와 락 매니저의 작동

분류:

- Optimistic concurrency control(OCC)
    - transactoin이 서로를 간섭하지 않는 경우(serializable)
    - 각자의 히스토리가 관리가 됨
    - 커밋 전에 히스토리끼리의 충돌을 확인함
    - 트랜잭션 간 모순 발생시 둘 중 하나 폐기
- Multiversion concurrency control(MVCC)
    - 여러개의 타임스탬프 버전 관리
    - 검증 기술: Lockless(timestamp 등)/lock-based(two-phase locking 등) 활용
- Pessimistic/Conservative concurrency control(PCC)
    
    shared resources를 관리하고 접근을 부여하는 방식에 따라 2가지로 분류
    
    - lock-based: 데이타베이스의 기록을 편집과 접근하는데 락을 사용하여 한개씩만 접근
    - nonlocking: 스케쥴을 설정해서 읽기와 쓰기를 관리하고 실행을 제한한다.
    

**Serializability**

schedule: database의 상태에 대해서 read, write, commit, abort하는 경우 등의 연산 목록

- A schedule is complete: 내부의 모든 Operation을 실행함
- correct schedule: 원래 연산과 동일한 구성이나 병렬 처리, 또는 효율을 위한 재배열 가능
- said to be serial: 모든 트랜잭션이 독립적으로 일어나서 순서를 자유롭게 바뀔 수 있는 경우
- serializable schedule: 순서가 자유로운 schedule

**Transaction Isolation** 

언제, 어떻게 Transaction의 결과가 visible 하게 될 지의 정도

추가 발생 비용:

- additional coordination
- synchronization

**Read and Write Anomalies(이상현상)**

Read Anomalies

- dirty read: 다른 트랜잭션의 커밋되지 않은 변경 사항을 읽을 수 있는 경우
- nonrepeatable(fuzzy) read: 같은 부분을 다시 읽었을 때 그 사이에 변경이 발생해서 다른 값을 반환
- phantom read: range 쿼리에 대한 nonrepeatable read 발생

Write Anomalies

- lost update: 서로 다른 트랜잭션이 같은 값 수정
- dirty write: 아직 커밋되지 않은 값을 바탕으로 사용하거나 수정해서 저장
- write skew: 각 트랜잭션은 요구되는 불변 사항을 만족하지만 전체 데이터베이스의 상태에서는 만족하지 않는 경우

**Isolation Levels**

이 각자 허용하는 범위는 아래로 갈수록 작아진다. 

- Read Uncommitted: Dirty, Non-Repeatable, Phantom
- Read Committed: Non-Repeatable, Phantom
- Repeatable REad: Phantom
- Serializable: none

snapshot isolation: 모든 트랜잭션은 상태 변화를 확인할 수 있고 작동 중 값이 변하지 않았을 때만 커밋할 수 있다. 
두 개의 트랜잭션이 동시에 업데이트를 희망할 경우 lost update abnormaly를 방지하기 위해 하나만 허용한다. 
write skew abnormaly의 발생 가능성은 존재한다.
