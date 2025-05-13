## B-Tree Basics

mutable: 데이터를 직접 수정 가능한 저장 구조

전통적인 B-tree, slottd page

in-place update(제자리 갱신): 대상 파일 내부의 기존 위치에서 직접 수정, 기존 데이터 덮어쓰기

immutable

LSM Tree, Append-only log, Git, Apache HBase/Cassandra, **Blockchain** 등

Out-of-place Update

- 데이터 무결성 보장: 기존 데이터 변경되지 않음
- 동시성 처리 용이: 여러 스레드, 트랜잭션의 읽기 작업을 충돌 없이
- 복구 유리: undo log 등 없이 롤백
- 버전 관리 가능: 상태 이력 저장 가능(Time travel, snapshot 등 구현)

**앞으로 책에서는 모든 key가 unique location의 1개의 data record와 연관되어 있다고 가정**

storage engine은 같은 data record를 여러 가지 버전이 저장되게 허용하기도 함

- multiversion concurrency control
- slotted page organization

## Binary Search Trees

- in-memory data structure
- root node에서 2개의 child node를 향한 pointer(left, right subtree)
- parent의 key보다 작은 값이 left subree
- leaf 도달 전 searched key 발견하면 탐색 종료

### Tree Balancing

unbalanced tree

- insert operation은 random하게 일어나므로 최악의 경우 pathological tree
- 성능 저하 발생 (logarithmic complexity가 아닌 linear tree)

balanced tree

- height of logN (최악은 여전히 O(N))
- leaf node의 최대 높이 차이가 1보다 작다
- rotation pivot을 기준으로 재정렬해서 만들 수 있다

### Trees for Disk-Based Storage

On-disk data structure로 Binary Search Tree를 사용하기 어려운 이유

- binary tree의 fanout == 2로는 유지 보수 비용이 비합리적
    - balancing
    - relocate nodes
    - update pointers
- locality가 확실치 않음
    
    새로 추가한 node가 parent 근처에 작성된다는 보장 없음
    
    child pointer가 여러 disk page를 넘어가는 경우 발생
    
- tree height이 과도하다
    
    O(logN)의 탐색, 즉 disk transfer
    

## Disk-Based Structures

space, complexity 요구사항을 고려한 이후에도 Database Structure는 저장 매체의 한계를 고려해야함

block drive abstraction: HDD와 SSD 모두 최소 단위가 byte가 아니다

single word 읽을 때에도 block 전체 읽음

### Hard Disk Drives(HDD)

spinning disk의 경우:

sector 단위로 operation 적용( 512btye~4KB)

- seek: disk rotation과 head movement로 비용 발생
    
    Sequential I/O의 장점 부각
    
- read/write: seek보다 상대적으로 저렴

### Solid State Drives(SSD)

Random이나 Sequential 접근 차이가 trivial

prefetching, reading contiguous pages, internall parallelism은 순차 접근이 빠르다

garbage collection이 random write 성능에 영향 줌

SSD 하위 물리적 구성:

- NAND Flash
    - Cell type(SLC, MLC, TLC, QLC 등)
    

논리적 구성:

data of 1~N bits → memory cell 

→ strings(32~64 cells)

→ arrays

→ pages (2~16KB)

read/write 가능한 최소단위

→ blocks (64~512KB)

erase 가능한 최소 단위(비어있는 memory cell에만 수정 가능)

→ planes

→ die

Flash Translation Layer(FTL)

- page ID를 물리적 위치에 매핑, page tracking(empty, written, discarded)
- garbage collection

### On-Disk Structures

data의 양이 커서 일부만 in memory에 cache, 전체 dataset은 접근이 용이한 형태로 disk 저장

pointer 제어 최소 단위가 block

pointer를 다룰 때 발생하는 경우 2가지

pointer dependency가 증가하면 code가 복잡해지므로 개당 수명과 개수 최소화

1. pointer가 가르키는 대상보다 먼저 disk에 적히는 경우
2. pointer가 기록되기 전 memory에 cache된 경우

pointer 최적화

1. improving locality
2. 구조의 internal representation 최적화
3. out-of-page pointer의 개수 최소화

**Paged Binary Trees**

locality 증가: 이미 불러 온 page에 다음 node 존재

page 사이의 node와 pointer로 인한 overhead 발생 가능

## Ubiquitous B-Trees

B-Tree의 node는 직사각형으로 표현

Pointer block에서 화살표로 child node로 연결

- 정렬된 tree
- logarithmic complexity
- level jump시 disk seek 1번
- point query(=)와 range query(≤,>,≥,<) 표현 가능

## B-Tree Hierarchy

- page organization 기술이다 보니 node == page 같은 의미
- B+-Tree: fanout이 높은 multiway tree, B-Tree와 혼용

각 node가 N개 key와 N+1개의  child node pointer 가진다

split, merge 등으로 balance 유지

node 분류: Root node, Leaf node, Internal node

occupancy: number of keys / node capacity

## Separator Keys

Keys == index entries == separator keys == divider cells

subtrees == branches == subranges

key range로 전체 트리를 subtree로 분할

보통 범위의 왼쪽이 포함

경우에 따라 sibling node pointer 가짐(양방향도 가능)

bottom to top 방식으로 작성

storage utilization이 최저 50%, 하지만 대부분 그 이상

## B-Tree Lookup Complexity

고려사항:

1. block transfer 횟수 == tree의 height
    
    M개의 node가 있는 tree에서
    
    각 level 별로 K배로 node 증가시 K를 밑으로 하는 log M == height
    
2. comparison 횟수
    
    각 node 내부에서 key 탐색 시 이진탐색, logN
    

## B-Tree Lookup Algorithm

찾는 value보다 큰 첫번째 seperator key 탐색으로 subtree 특정

1. point query, update, deletion: exact match
    
    exact value 없으면 못 찾은 것
    
2. range scan, insert: finding predecessor
    
    가장 비슷한 key-value pair 선택 후 sibling pointer 따라서 이동하거나 범위 조건을 더 이상 만족하지 않을 때까지 탐색
    
    → B+Tree가 범위 탐색에 유리한 이유
