## Column/Row - Oriented DBMS
  data record는 column과 row로 구성된 table
  field는 column과 row의 교점인 single value
  같은 column이면 data type 동일

  access pattern에 기반해 
  모든 열 고려해야하거나 point queries, range scan -> row
  버려지는 열 많거나 열의 일부를 aggregation -> column

  ### Column wise (MonetDB, C-store)
    같은 column에 있는 정보들을 순서대로 저장
    column을 기반으로 하는 query에 유리(one pass로 읽을 수 있다)
    aggregates(trend, average value 등) 처리 편리함

    column level에서 metadata 보존 방식
      1. 각 value가 key 소유
      2. implicit identifier(virtual ID)로 offset 
      3. 이외에도 다양한 colum-oriented file format 등장(Apache Parquet, ORC, RCFile 등)

  ### Row wise (MySQL, PostgreSQL 등 대부분)
    key로 유일하게 식별되는 경우
    열에 있는 항목들이 registration form 등으로 한번에 작성될 때 유리
    spatial locality(배열 순차 접근이나 루프를 통해 연속된 데이터 구조 처리시 발생)

    한 명의 user에 대한 모든 정보에 접근할 때 편리함
    한 가지 항목에 대해 모든 user 조회 시 불리함

## Distinctions and Optimizations
  row와 column 등 data layout 말고도 가능

  같은 column의 값 읽어오면 
    1. cache utilization 개선
    2. computational efficiency 증가
    3. compression ratio 개선

  ### Wide Column Stores(BigTable, HBase)
    column을 multidimensional map에 의해 grouping한 column families
    각 column 내 data는 row-wise
    key나 key의 연속으로 된 data 관리에 유리
    책에 webtable의 예시 등장

    schema가 유연하다(row가 동적으로 추가 가능)
    범위에 대한 query가 효율적
    data가 자동으로 partitioning 되어 여러 server에 분산 저장, 대규모 data 처리 적합
    다양한 application 지원(web indexting, 사용자 맞춤형 콘텐츠 제공 등)

## Data files, Index files
  implementation-specific format으로 data의 file 관리
    장점
    1. storage efficiency
      저장 시 발생하는 오버헤드를 줄일 수 있다
    2. access efficiency
      최소한의 단계로 record를 위치시킨다
    3. update efficiency
      disk 내에서 변화를 최소화할 수 있다

  각 table이 1개의 분리된 file
  또한 data file과 index file(metadata)도 대체로 분리하는 편
  table 내의 record는 search key로 탐색, index로 발견
  file은 page로 분리(sequenced records 또는 slotted pages)

  새 record는 key와 value의 쌍으로 표현
  삭제시 deletion marker를 사용해 shadow 후 garbage collection으로 제거

  ### Data files(Primary files)
    index-organized tables(IOT)
      index에 data 저장
      순차적 탐색으로 range scan 가능
      disk 탐색 횟수 1번 이상 감소
    heap-organized tables(heap files)
      작성 순서대로 저장
      추가적인 index 구조 필요
    hash-organized tables(hashed files)
      bucket에 record 저장, key의 hash로 위치 지정
      append order로 저장 가능
    
  ### Index files
    key를 data file의 위치와 연결
    
    primary index
    secondary index

    data record 접근시 직접(file offset 활용) 혹은 primary key index를 통해서 중 선택

## Buffering, Immutability, Ordering

  ### Buffering
    disk에 적기 전에 의도적으로 buffering 
    평균 I/O 비용 절감

  ### Immutability(Mutability)
    1. append-only
      맨 뒤에 추가
    2. copy-on-write
      복사 후 새로운 곳에 저장

  ### Ordering
    key의 순서로 disk의 page에 저장되어있는지 유무
