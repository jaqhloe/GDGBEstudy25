## Transaction Processing and Recovery

이전까지는 storage structure에 대해서 정리했고 이제는 그 상위 단계인 

buffer management, lock management, recovery에 대해 다룬다. 

transaction: 더 이상 작게 나눌 수 없는 reading과 writing의 논리적 최소 단위로 

아래의 4가지 성질을 만족해야한다.(ACID)

- Atomicity: transaction은 indivisible하다.
    
    아예 성공하거나 전혀 발생하지 않은 것으로 commit 또는 abort 된다.
    
    abort되면 다시 시도한다.
    
- Consistency: database의 모든 invariants를 유지하면서 valid state로만 database가 변경된다. user, application마다 정의가 달라진다.
- Isolation: 동시에 일어나는 transaction들은 서로를 간섭하지 않는다.
    - when the changes to the database state become visible
    - what changes become visible
- Durability: commit된 transaction은 disk에서 유지되어야한다.
    - power outages, system failures, crashs 등등 발생해도 안전해야한다.

위의 성질들을 만족하면서 transaction을 구현하기 위해 storage structure외의 components:

- 노드 자체의 transaction manager: coordinates, schedules, track transaction/steps
- lock manager: resources 접근에 대한 관리, data integrity 보장
    - lock requested: lock manager가 이미 다른 곳에 lock이 있는지 확인 후 lock
- page cache: 영구 저장 매체 disk와 다른 storage engine의 중간 매체
    - main memory의 상태변화 관리 및 disk 저장 전 cache 역할
- log manager: operation의 변화 과정을 저장하는 log entry
    - cache에 저장된, disk에는 저장안된 사항 기록
    - Undo changes시 활용

만약 distributed transaction이라면 coordination, remote execution이 추가로 필요하다.

Buffer Management 

disk 접근 횟수를 줄이기 위해 대부분 database는 page를 Main memory에 cache한다.

= virtual disk, buffer pool

- page in: cache된 게 없어서 disk에서 읽어옴
- dirty: cached page에 내용 변경 발생
- flush back on disk: 변경 사항이 disk에 저장
- eviction: cache 공간 확보를 위해 page disk로 돌려보냄

load된 페이지는 빈 공간에 disk와는 다르게 순서 없이 저장된다.

Caching Semantics

- one-way process: memory에서 disk로 저장이 일어난다.
- databse system 자체가 구현한다.
    - disk access 추상화
    - 논리적/실제 물리적 write operatIon을 구별한다.
- 과정
    - 이미 cache된 contents 있는지 확인
    - 없는 경우 physical address 찾은 후 memory에 내용 불러오기
    - cache 변경 시 disk의 내용도 변경하기
- referenced: cache에 올라갔다 내려온 page content
- pinning: eviction 방지

Cache Eviction

- 변경 사항이 없고 disk와 cache가 같은 내용이라면 바로 evict
- dirty page는 그전에 flushed back
    
    eviction 마다 flush back 되는 건 비효율적이므로 별개의 background process 존재
    
    durability 보장을 위해 flush에 대한 checkpoint 관리
    
    - write-ahead log(WAL)
    - page cache
- referenced page는 다른 thread가 사용 중이므로 유지

Locking Pages in Cache

read/write마다 disk 접근은 비효율적이므로 tree의 root에 가까운 page들은 변경이 적은 점을 이요해 cache에 저장한다.(Locking pages, pinning)

하위 노드에서 동시에 발생한 변경이 상위 노드로 갈수록 가까워지면 동시에 buffer 처리 가능

 

Page Replacement:

eviction policy이다.

단순히 cache 공간이 늘어나는 걸로는 효율이 좋아지지 않음

Latency, disk access횟수 등에 크게 영향을 미친다.

- FIFO: 먼저 들어온 페이지를 먼저 내쫓는다. 비효율적이다.
- LRU:
    - 기본: 가장 최근에 사용한 페이지를 계속 유지.
    - 2Q: 2개 queue로 최근 사용/ 자주 사용을 구분
    - LRU-K: 마지막 K번 접근을 관리
- CLOCK: 효율성을 예측 정도보다 중요시 여길 때
    - 참조되면 1
    - 참조되지 않으면 1→0
    - 0이면 eviction

최근 참조했을 때를 기록하는 대신에 횟수를 저장하는 게 나은 구현도 있다.

- LFU: Least frequently used
    
    아래 3가지 대기열을 frequency filter를 통해 관리한다 
    
    - admission: 새로 추가 된 내용
    - probation: 쫓겨날 내용
    - protected: queue에 더 유지되어야할 내용
