**Chapter 4. Implementing B-Trees** 

Implementing B-Trees 

앞으로 나올 단원의 내용 요약:

1. 키와 포인터의 관계 수립, 페이지 간 헤더와 링크 만들기
2. 트리의 루트에서 잎 노드로 가는 방법(breadcrumb의 관리, 노드의 분할과 합체 과정에서 부모 노드와 연결하는 법)
3. 최적화 기술(재분배, right-only appends, bulk loading), 유지 과정, 가비지 컬렉팅

Page Header: 페이지를 관리하는데 필요한 정보들을 가지고 있다.

플래그: 페이지 내용과 레이아웃, 페이지 내 셀의 개수, 빈 공간의 lower and upper offset 등 metadata

PostgreSQL: 페이지 크기, 레이아웃 버전

MySQL InnoDB: heap record 개수, level 등 implementation-specific 값

SQLite: cell 개수, rightmost pointer

Magic Numbers: 

page를 나타내는지 혹은 유형, 버전 등 정보

validation, sanity checks에도 활용

페이지의 유효성(loaded and aligned correctly)을 확인하기 위해 write 과정에서 header에 magic number 50 41 47 45를 넣고, read 과정에서 이 4개 byte가 예상과 같은지 확인

Sibling Links:

forward/backward link으로 양옆의 sibling page를 가르키면 부모 노드로 돌아갈 필요가 없다.

대신 update(split, merge) 발생시 주변 노드도 함께 반응해야 해서 additional locking 필요하다.

concurrent B-Tree 에서 유용하다.

Rightmost Pointers:

seperator key가 각 child pointer를 하나씩 가지고 마지막 pointer는 key와 연결되지 않게 따로 존재한다. (B-Tree의 노드는 자식 페이지 수+ 1개의 포인터를 가진다.)

제일 오른쪽에 있는 자식에서 split이 일어나서 부모와 새로운 노드를 rightmost pointer로 연결된다.

split된 노드는 key가 promote 되고 여기서 pointer를 가져다 쓴다.

Node High Keys: 

위와는 다르게 rightmost pointer를 노드의 high key(현재 node의 subtree 중 최대값)와 함께 저장하는 방식이다. 이 방식으로는 (키, 포인터) 처럼 pairwise로 저장해 비교적 유지가 단순하고 고려해야할 예외가 적다. (단편적으로, 이전까지의 방식으로는 범위의 최대값이 무한대까지가 된다.)

Overflow Pages:

노드의 크기와 tree의 fanout 값은 고정되어 있다. 따라서 남는 공간을 최소화하되 다양한 값을 저장하려면 필요에 따라 page 크기를 변화할 수 있는 방법이 필요하다. 

그래서 multiple linked page로 원래 노드(primary page)에서 overflow page로 연결한다. 

이 방식으로 항상 max_payload_size 이상의 byte를 확보할 수 있다. 

payload가 이보다 크면 node에 원래 연결된 overflow page가 존재하는지 확인 후 없으면 새로 할당한다.

첫번째 overflow page 생성시 page ID를 primary page의 header에 저장한다.

이후 붙는 page들은 이전 page의 header에 저장한다. 

key는 cardinality가 높기 때문에 primary page의 일부분으로 금방 찾을 수 있고 자주 접근하지 않는 data record의 경우 overflow된 부분까지 다시 가져와야한다. 

SQLite: 단일 파일 기반 경량 DB, 내부적으로 B-tree 사용해서 data 저장

큰 BLOB, text data를 메인 페이지(B-tree node)가 아니라 overflow page(또는 payload overflow)에 분리하여 저장

main page의 공간 효율성이 좋아지며 B-tree의 node split이 감소해 탐색 성능도 개선된다.

MySQL InnoDB: COMPACT, DYNAMIC 같은 ROW_FORMAT(행 형식)에 따라 긴 칼럼 데이터가 페이지에 저장되거나 overflow 되는 방식이 달라진다.

Binary Search 

정렬된 데이터에만 의미가 있다.

결과로 돌아온 수가 양수이면 해당 값이 input array에서 찾으려는 값의 위치이고, 음수이면 insertion point(해당 key보다 큰 최초의 값)를 돌려받은 것이다. 

Indirection Pointers(cell pointer에서 이분탐색이 일어나고 cell pointer가 실제 cell을 가리키는 방식)

Propagating Splits and Merges 

split이나 merge 발생시 root node까지 변화가 반영되어 올라가야 한다.

다만 하위 페이지가 상위 페이지에서 참조될 때 항상 메모리로 로드되기 때문에 부모 포인터의 정보를 디스크에 영구 저장할 필요는 없다. 부모 노드의 포인터는 부모 노트 변화시 업데이트한다. 

일부 구현(WiredTiger)에서는 형제 노드 포인터를 사용했을 때 발생하는 교착 상태를 피하기 위해 부모 포인터를 대신 사용한다. 

Breadcrumbs: 스택으로 역추적 경로 저장
