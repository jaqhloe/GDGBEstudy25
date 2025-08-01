**Chapter 6. B-Tree Variants** 

- B-Tree의 공통적 특징: 형태, 분할과 병합으로 재정렬, 탐색과 삭제 알고리즘
- 구현마다 차별화: 동시성 처리, 디스크 상 페이지 출력, 형제 노드 간 링크, 유지 과정

**B-Tree Variants** 

앞으로 등장할 5가지 B 트리의 방법론과 그 구현을 소개함

→ Copy-on-write, Lazy, FD, Bw, Cache-oblivious

1. **Copy-on-Write** 
    
    페이지에 수정이 발생하면 내용을 복사해서 복사한 페이지에 업데이트를 한다. 
    
    새로 만들어진 페이지는 기존 페이지와 수평으로 존재하고 업데이트 후 한꺼번에 포인터를 바꾸는 방식으로 수정한다.
    읽기와 쓰기는 메모리맵으로 직접 처리한다. 
    
    물론 저장 공간을 더 차지하고 프로세서 시간도 잡아먹지만 B 트리가 얕은 편이라서 장점이 훨씬 부각된다.
   기존 B 트리는 동시성 제어를 위해 모든 작업에 latching이 필요한 것과 다르게 변경 전 버전이 읽기 작업에 쓰이고 쓰기 작업과 동시에 존재할 수 있다.
   또한 불완전한 상태의 페이지가 보이는 일이 없고 모든 작업이 끝나야 topmost pointer가 한번에 수정되기 때문에 시스템이 중단되어도 일관된 데이터 상태를 유지할 수 있다. 
    
    **Implementing Copy-on-Write: LMDB**
    
    Lightning Memory-Mapped Database는 위의 기술을 사용한다.
    
    단일 레벨의 아키텍처(메모리맵으로 읽기 쓰기를 직접 수행해서 실체화가 필요없는 상황) 덕분에 페이지 캐시, WAL, 체크포인트, compaction 모두 불필요하다.
    
    root node의 버전이 lastest version과 새로 업데이트가 반영될 버전 2개로 존재한다.
    
    sibling pointer가 없다. 
    
    **Abstracting Node Updates** 
    
    수정이 발생하면 디스크보다 메모리에 있는 노드를 먼저 업데이트 하는데, 이때 표현 방버이 여러가지가 있다.
    
    첫번째로는 노드의 캐시된 버전에 직접 접근하는 것이고 두번째로는 wrapper object 사용, 마지막으로는 구현된 언어의 특징을 살려서 표현하는 것이다.
    
    1. RAW binary 직접 조작
        
        메모리 관리에 관심없는 언어들은 사용할 때는 노드를 구조체로 정의하고 포인터와 runtime cast의 raw binary를 직접 사용한다.
       메모리 오버헤드가 작아지고 속도는 빨라지는데 개발 난이도가 올라가고 잘못된 포인터 조작과 같은 실수가 발생할 수 있다.
        
    3. 언어의 기본 객체, 구조체 활용
        
        삽입, 업데이트, 삭제에 유용하며 저장할때 메모리 → 디스크 순으로 차례대로 업데이트 된다.
        
        1번과 반대 척도에 있는 장단점을 가지고 있다.
        
    4. wrapper 객체 활용
        
        메모리를 관리하는 언어들의 경우 노드를 wrapper obj를 통해 버퍼에 접근하게 한다.
        
    
3. **Lazy B-Trees** 
    
    버퍼를 사용해서 노드에 일어나는 업데이트를 미루는 유형
    
    **WiredTiger - 개별 노드에 적용**
    
    MongoDB의 기본 storage engine
    
    수정이 발생하면 각 노드에 있는 업데이트 버퍼에 먼저 저장한다. 
    
    업데이트 버퍼는 skiplist로 구현되어 있다.
    
    읽기 중에 업데이트 버퍼에 접근할 수 있고 이때 디스크의 정보와 일치하는 방향으로 읽는다.
    
    이 방식을 사용하면 백그라운드 스레드에서 분할과 병합이 일어나서 읽기 쓰기를 하는데 기다릴 필요가 없다.
    
    **Lazy-Adaptive Tree (LA tree) - 서브 트리로 그룹화해서 적용**
    
   위의 방식이 각 노드에 개별적으로 동작하는 방식이라면 이 방식은 부분트리의 꼭대기 노드부터 그 아래에 한꺼번에 적용하는 것이다.
   레코드를 삽입하면 꼭대기 노드에 들어가는데 이 노드가 꽉차면 아래의 노드로 옮겨간다.
   업데이트가 잎 노드에 도달하면 삽입, 업데이트, 삭제 연산이 한꺼번에 일어난다.
    
5. **FD-Trees(Flash Disk Trees)**
    
    변경 가능한 head tree와 그 아래에 여러 개의 immutable sorted runs로 이루어지 트리이다.
    
    head tree가 꽉 차야 그 아래 변경 불가능한 런으로 전파된다.
    
    append-only storage와 병합 과정을 여러 개 노드에 한꺼번에 적용해서 여러 개 노드를 한꺼번에 업데이트한다.
   쓰기를 위해 대상 노드를 정확하게 찾을 필요 없다.
    
    **Fractional Cascading**
    
    레벨 간 포인터를 관리해서 검색 성능을 높이기 위해 사용하는 기법이다.
    
    아래에 있는 원소 중 하나를 위로 가져와서 bridge(fence)로 설정해서 포인터를 연결하는데 하위 노드에 있는 원소를 찾을 때 가까운 곳부터 찾는데 도움을 줘 검색 성능이 좋아진다.  
    
    **Logarithmic Runs**
    
    Fractional cascading을 발전시켜서 맨 앞 원소를 올려보내는 방식을 사용하며, 추가로 각 런이 로그적 크기 관계를 가지며 증가하는 방식이다.
    하위 런은 상위 런에 더 이상 공간이 없을 때 생성된다. 이미 존재하면 병합한다.
    tombstone(filter entry)으로 삭제를 표시해놓고 가장 낮은 레벨로 전파되면 진짜 삭제한다.
