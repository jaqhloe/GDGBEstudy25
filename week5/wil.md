이번 주차는 제 흥미 위주로 정리했습니다.

[54p] page 내부의 cell은 key만 가지거나 key와 value를 모두 가지는 형태/ 고정된 사이즈이거나 가변 가능한 사이즈 둘 중 하나의 uniform한 형태이다. 
균일하기 때문에 메타데이터를 페이지 레벨에 한 번만 저장하여 효율성을 높인다.

그렇다면 균일하지 않게 저장해 사용하는 경우가 있을까?

→ 존재하긴 한다. 완전한 비균일성은 존재하지 않는다. 비균일성을 인정하더라도 복잡성과 비효율성을 줄이기 위해 특정 패턴이 존재하거나 제약 사항을 정해둔다.

1. 컬럼 내 다양한 타입이 존재하는 레코드(가장 흔한 경우)
    
    관계형 데이터베이스의 행이 INT, VARCHAR, DATE, BOOLEAN 등 다양한 데이터 타입의 컬럼으로 구성되어 있는데 페이지 내부에서 하나의 ‘셀’ 또는 ‘레코드’로 저장되면 레코드 내 각 필드는 길이가 고정이거나 가변일 수 있다. 
    
    - 레코드 내부에 각 필드의 오프셋이나 길이가 저장된 헤더 등 별도의 메타데이터 저장한다.
    - TEXT, BLOB 등 크기가 큰 가변 길이 데이터는 해당 페이지가 아닌 오버플로우 페이지나 외부 저장소에 위의 정보를 저장하고 원래 레코드에는 포인터(주소)만 남긴다.
      
2. NoSQL 데이터베이스
    
    문서 기반 데이터베이스의 경우 스키마가 유연해서 같은 컬렉션 내의 문서라도 각각 다른 속성을 가지기도 한다. 각 문서를 저장될 때 페이지 내의 하나의 ‘셀’ 또는 ‘레코드’로 취급한다.
    
    - 각 셀(문서)를 저장할 때 그 자체로 self-describing 형태로 저장한다. 스키마 유연성이 증가하지만 메타데이터의 중복 저장, 공간 관리의 복잡성 등 문제가 존재한다.
      
3. LSM-Tree의 SSTable
    
    key-value pair로 정렬하여 저장하는데 value가 가질 수 있는 데이터 타입, 크기가 다양하다.
    
    - SSTable 자체는 key-value pair을 연속적으로 저장하지만, 각 key-value pair에는 자체적인 길이 정보나 오프셋 정보를 포함한다.
    - SSTable 내부에 index block을 두어 특정 키 범위를 빠르게 찾을 수 있도록 돕고, 블룸 필터(Bloom Filter) 등을 사용하여 특정 키의 존재 여부를 빠르게 판단할 수 있다.

      

[57p] SQLite는 unoccupied segments를 freeblock이라고 표현한다. 그리고 첫번째 freeblock을 표시하는 pointer와 전체 여유 byte를 page header에 저장한다. 

다른 데이터베이스 제품들은 같은 목표를 달성하기 위해 동일한 방법을 사용할까? 어떤 방법들이 있을까?

→  위의 방식도 훌륭하지만, 대규모 데이터베이스나 특정 workload에서는 다른 최적화를 사용하기도 한다.

1. Bitmap 
    
   page 내부 공간을 byte 등 더 작은 단이로 나누어 각 segment의 사용 여부를 bitmap의 0과 1로 표현한다.
   작은 공간도 정밀하게 관리할 수 있고 특정 크기의 빈 공간을 찾을 때 빠르다.
   하지만 bitmap 자체가 page 내의 공간을 추가로 차지할 수 있다.
   일부 시스템들은 page 내부의 작은 object(tuple header, item identifier 등)를 관리하기 위해 bitmap과 유사한 구조를 활용한다.
    
3. Free List, Free Space Map의 계층적 관리
    
   단일 page 내 빈 공간이 아니라 여러 page의 빈 공간을 관리하기 위해 빈 공간의 정보를 별도의 free space map에 저장하는 방법ㅇㅣ다.
   특정 크기의 빈 page를 찾거나 새 table, index를 할당할 때 전체 데이터 파일이 아닌 map만 참고해서 공간을 빠르게 찾을 수 있다.
   다만 map 자체를 관리하는 overhead가 발생하거나 map에 대해서도 동시성 제어가 필요하다.
   PostgreSQL이나 SQL Server 등은 tablespace나 filegroup 수준에서 빈 공간을 관리하는 별도의 방식을 가진다.
   PostgreSQL의 경우 각 page에 FSM(Free Space Map)을 통해 페이지 내의 여유 공간을 기록하고, 이를 통해 새로운 tuple을 삽입할 페이지를 찾는다.
    
5. Overflow Page 사용
    
   가변 길이 data가 너무 커서 Page에 들어갈 수 없는 경우 해당 data만 별도의 overflow page에 저장하고 원래 record에는 저장된 page에 대한 Pointer만 기록한다.
   main page에 저장된 data의 밀도가 높아지지만 overflow된 data를 읽을 때 disk 접근이 발생해서 성능이 나빠질 수 있다.
   MySQL의 InnoDB, PostgreSQL 등은 긴 TEXT나 BLOB data를 overflow page(TOAST 테이블)에 저장하는 기능이 있다.
