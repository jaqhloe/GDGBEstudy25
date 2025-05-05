### DBMS(DataBase Management System)
    DB, database system, database와 사실상 동일한 개념

    구성요소(대체적으로)
        transport layer
        query processor
        execution engine
        storage engine

### Storage Engine(database engine)
    data를 store, retrieve, manage
    간단한 data의 manipulation API 제공
    schema, query language, indexing, transactions 등 기능은 이 layer 위 존재
    key, value는 정해진 형식이 없는 임의의 문자 나열 가능

    pluggable storage engine의 등장으로 이외의 subsystem에 집중 가능
        이외의 subsystem? 
          query parser, optimizer, transaction management, cache or buffer pool, index management
    또한 여러 engine 중에서도 선택 가능
        MYSQL(InnoDB, MyISAM, RocksDB)
        MongoDB(WiredTiger, MMAPv1)

    구성 요소는 이후 내용에 나옴

### Comparing Databases
    처음부터 내가 데이터베이스를 구현하려는 상황을 고려해 적합한 데이터베이스를 고른다
    장기적으로 사용하다가 데이터 크기가 커지면 옮기기 곤란해질 수 있다
    실제 상황을 고려하여 충분한 시간을 투자해 성능 확인

후순위 고려 사항

    구성 요소(storage engine, data 공유/복제/분배 방법 등)
    인지도
    구현 언어

실질 고려 사항

    동작 방식(black box로 대하지 않고 내부 작동 원리 이해)
    개발하려는 시스템의 workloads를 simulation 해서 비교
        capacity가 증가하거나 시간이 지나야 발견되는 문제들이 존재
        performance, debugging 뿐만 아니라 DB의 community 활성화도 확인

    DBMS 비교시 고려할 요소들
        Schema, Record 크기
        clients 수
        Query의 종류, Access patterns
        읽고 쓰는 query의 비율
        위 요소들에 대한 예상되는 변경사항

    위 요소들을 고려하여 점검할 사항
        DB가 주어지는 Query를 처리할 수 있는가
        저장할 것으로 예상되는 data의 규모를 handle할 수 있는가
        1개의 single node가 처리할 수 있는 read/write operation 개수
        예상되는 growth rate를 고려해서 cluster를 확장할 방법
        maintenance process가 어떻게 되는가

    DBMS가 제공하는 Stress Tool로 simulation
    혹은 performance benchmarking
        YCSB라고 Yahoo!에서 제공하는 framework와 workload 존재
        TPC-C Benchmark 

        Benchmark는 service-level agreement의 세부사항을 확인하기, system 요구사항, 용량 설정 등을 점검하기에도 유용

    이후에는 
        해당 DBMS를 구현한 코드(codebase)에 익숙해지기
            log record, configuration paramenters를 읽기 수월해진다
                configuratoin parameters?
                    동작 조정 설정값(변수)
                    성능, 보안, 메모리 사용, 캐시, 트랜잭션 동작 방식 등 포괄
            구현한 서비스나, database 그 자체에 존재하는 issue도 파악 가능

## Understanding Trade-Offs
얻는 것이 있으면 잃는 것도 있다
    optimized for low read or write latency
    maximize density
    operational simplicity

Storage engine 설계시 고려하는 것들
    physical data layout
    organizing pointers
    serialization format
    garbage collecting the data
    DB system에 storage engine 적용 방식
    *NEVER* lose any data

## Database Classification
분류는 다양할 수 있으나 아래의 3가지로도 분류할 수 있다
    1. Online transactoin processing(OLTP)
        user와 대면해 요청과 transaction 처리
        query가 predefined, short-lived

    2. Online analytical processing(OLAP)
        고수준 aggregation 처리
        analytics, data warehousing 등에도 활용
        query가 complex, long-running, ad-hoc

    3. Hybrid transactional and analytical processing(HTAP)
        1과 2의 혼합된 성질

    이외에도
        key-value stores
        relational databases
        document-oriented stores
        graph databases

## DBMS Architecture
구성 요소와 그 관계는 경우에 따라 다르게 분석되나 대부분 아래와 같다

Client/Server 모형
    DB의 instance(nodes) -> server
    application -> clients

Transport
    query 형태로 들어온 request를 수신, query processor에 전달
    다른 node나 database cluster와 통신
Query Processor
    query를 parse, interpret, validate
        parse -> optimizer 
            internal statistics(index cardinality, approximate intersection size)
            data placement
    access control check 수행(interpret 완료시)
Execution engine
    processor에서 만들어진 execution plan으로 수행

    remote execution
    local queries

Storage engine

    Transaction manager
        consistency 유지
    Lock manager
        database object 잠금
    Access methods(storage structures)
        heap file, B- tree 등으로 disk 관리
    Buffer manager
        memory에 data cache
    Recovery manager
        failure에 대비해 log와 restoring system 관리
    
## Memory-Based / Disk-Based
In-memory database management systems(main memory DBMS)
    RAM memory에 data 저장, disk로 recovery 및 logging
    disk에 비해 접근 비용이 낮고 접근 단위(정밀도)가 작으며 프로그래밍이 쉽다
    빠르지만 용량에 따른 비용 문제 발생
    RAM volatility 문제
        disk에 backup 생성
        write-ahead logs(batch로 취급, snapshot과 checkpointing)
        backup copy(sorted disk-based structure)
Disk-based DBMS
    disk에 대부분 data 저장, memory에 disk의 내용 cache
    설계적 제약으로 in-memory만큼의 성능이 안 나옴
NVM
