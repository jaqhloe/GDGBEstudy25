Rebalancing, Load Balancing

  B-tree 구현 중 일부는 분할과 병합을 미루기 위해 같은 계층에서 원소를 옮기기도 한다. 
  
  insert 중 overflow 발생시: 일부 원소를 sibling node에게 이동
  
  delete 도중 Merge 필요시: 동일하게 원소를 형제 노드에게 이동
  
  이동 후 부모 노드에 있는 키를 업데이트해야 한다. 
  
  효과:
  
  - node occupancy 개선(절반 정도는 채워져 있게 된다)
  - 트리의 전체 높이가 과도하게 높아지는 것 방지해서 결과적으로 검색 성능 개선
  
  B*-tree는 모든 형제가 더 이상 입력할 수 없을 때 까지 형제 노드로 옮긴다.
  
  1개의 노드를 2개의 반쯤 차 있는 노드로 분리하는게 아니라 2개의 노드를 3개의 노드에 분배한다.
  
  SQLite는 이와 유사한 balance-siblings algorithm을 사용한다.


Right-Only Appends 

  대부분 데이터베이스 시스템들은 자동으로 단조 증가하는 값을 primary index key(기본키)로 사용한다.
  
  따라서 대부분 split은 가장 오른쪽 노드에서 일어난다.
  
  무작위로 나열된 키인 경우 보다 nonleaf page가 적게 fragmented 된다.
  
  PostgreSQL의 fastpath
  
    insertion이 일어난 key가 rightmost page의 첫번째 키보다 크고 공간이 충분하다면, 그 앞의 read path를 전부 생략할 수 있다.(righmost page가 cache 되어 있다.)
  
  SQLite의 quickbalance
  
    가장 오른쪽 노드가 꽉 차 있으면 rebalancing, splitting 대신 새로운 rightmost node를 작성해 부모 노드에 포인터를 저장한다.

Bulk Loading(대량 로딩)

1. 정렬된 대량의 데이터를 한꺼번에 삽입하는 경우 INSERT 등으로 하나씩 넣는 것 대신 bulk loading
2. defragmentaion 등으로 tree를 rebuild 해야할 때  

rightmost append로 leaf level에 page-wise로 작성

leaf level 작성이 끝나면 위의 계층을 작성하기 때문에 

1. split이나 merge가 disk에서 일어나지 않는다.
2. 트리에 필요한 최소한의 부분만 가지고 있어도 된다.

Immutable B-Tree도 같은 방식으로 트리를 구성하나 space overhead가 발생하지 않는다.

**불변자료구조?** 

  기존 내용을 복사해 새로운 버전을 생성하거나 변경이 없는 부분은 구조적으로 공유해서 메모리 사용 최적화하는 방법으로 mutable B-tree와 상반되는 개념
  
  버전을 관리하기 좋고 동시성 제어가 쉬워진다. 
  
  코드의 안정성이 높다. 
  
  함수형 프로그래밍(Haskell, Clojure, Scala 등)에 적합하다.(불변성, 순수 함수와 부합)
  
  블록체인, 버전 관리 시스템, 일부 데이터베이스 시스템(MVCC 등)에서도 사용된다.
  
  다만 공간 효율이 나빠지고 쓰기 성능 오버헤드가 일어난다. 

Compression 

  overhead 방지
  
  압축 비율이 높어질 수록 접근 1번으로 가져올 수 있는 데이터 양이 증가하나 압축 및 해제 과정에서 CPU, RAM cycle이 더 필요해진다. 
  
  Open source database들은 pluggable 압축 방법 라이브러리를 제공한다.
  
  Squash Compression Benchmark 등에서 성능 평가를 비교할 수 있다
  
  (memory overhead, compression performance, decompression performance, compression ratio 등)

압축은 여러 단위로 일어날 수 있다:

1. Entire file
    
    압축 비율은 높아지지만 파일 수정시마다 다시 통째로 압축해야한다. 데이터의 크기가 커질 수록 불리하다.
    
2. Entire index
    
    무작위 접근이 자주 일어나기에 더욱 더 불가하다. 압축된 데이터의 위치를 파악할 수 없어서 압축에 대한 metadata를 먼저 읽어야 한다. 
    
3. page-wise
    
    그 동안 책에서 페이지 단위로 다루는 알고리즘들은 많이 살펴보았기 때문에 가장 적합하다.
    
    다만, 일부는 디스크 블록에 걸쳐서 존재하므로 추가로 블록을 더 읽어야 할 수 있다. 
    
4. compress data only(row-wise, column wise)
    
    행이나 열로 독립적으로 압축할 수 있다. page managementdhk compression이 서로 독립적으로 작동하게 된다. 
    

Vacuum and Maintenance 

  background에서 실행해서 시간 및 자원 절약

  삭제 된 부분은 포인터만 없애서 직접 0으로 바꾸지 않기도 한다.

Garbage collection

  vacuum process로 더 이상 접근이 불가능할 때 cell 회수

  Transcation이 끝날 때까지 ghost record 상태

Fragmentation Caused by Updates and Deletes

  delete 발생 시 leaf 수준에서는 header에서 cell offset을 제거한다.

  page가 split되면 offset만 trim

Page Defragmentation(compaction, vacuum, maintenance)

  write 도중 공간이 없으면 동시에 logical 순서로 page rewrite 진행(compaction 제외)

  사용되지 않는 In-memory pages는 page cache로 반환, free page list에 추가된다
