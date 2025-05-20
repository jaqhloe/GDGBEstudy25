### **Chapter 2. B-Tree Basics**

### Ubiquitous B-Trees

B-Tree 구조는 다양한 곳에서 발견된다

디스크 접근 횟수 최소화, 균형 잡힌 트리

- DB: 인덱스 구조 (MySQL InnoDB의 기본 인덱스는 B+ Tree)
- key-value 저장: RocksDB, LevelDB 등
- 파일 시스템: NTFS, HFS+ (macOS), Btrfs 등
- 디지털 포렌식: file의 metadata 구조 관리

### Counting Keys

key, child offset 세는 방법이 출처마다 다르다

- 적당한 page size를 나타내는 상수 k에 대해 k~2k개 key
- k+1~2k+1개의 child node pointer
- 1~2k개 key를 가지는 root page
- l+1개 key non-leaf

다른 출처

- N개의 separator key
- N+1개 pointer
- 나머지는 비슷하거나 동일

이 책에서는

- N개의 key(leaf node의 경우 key-value 쌍)

### B-Tree Node Splits(Overflow 등)

1. new node 할당
2. split 되는 노드의 절반을 new node에 복사
    - split point(midpoint)
3. new element를 배치
4. split된 노드의 parent node에 seperator key와 pointer 기록
    - promoted
5. 경우에 따라서 recursive하게 parent node도 split

- B-Tree에 값 insert
    
    target leaf 발견 후 insertion point 탐색
    
- B-Tree Update
- Overflow
    - leaf node: N개를 초과하는 key-value pairs
    - nonleaf node: N+1개를 초과하는 pointer

### B-Tree Node Merges(Underflow 등)

1. right node의 모든 elements를 left node로 복사
2. right node의 pointer를 parent에서 제거
3. right node 삭제

- Underflow
    - common parent
        - 1개의 node에 전부 저장 가능
        - 1개의 node에 전부 저장 불가능 → 재분배

### **Chapter 3. File Formats**

Chapter 2까지의 내용으로는 in-Memory B-Tree 구현 가능

Disk-based implementation을 위해서는 다른 방법 필요

### File Formats

- In-Memory access
    - transparent
    - virtual memory 덕분에 offset을 수동으로 관리할 필요가 없다
- Disk
    - system call로 접근
    - target file 내부의 offset을 명시해야 한다
    - main memory에 적합한 형태로 representation을 interpret
    

→ construct, modify, interpret 쉬운 형태의 file format 필요

- B-Tree implementation techniques
    
    split, merge 외에도 mechanic 구현
    
- on-disk structure의 pointer management
- on-disk의 B-tree는 page management mechanism으로 고려하기
    
    (page navigation algorithm)
    
- B-Tree의 complexity는 mutability에서 발생
    - page layouts
    - splitting
    - relocations
- LSM Trees의 complexity
    - sorting
    - maintenance

### Motivation

languages with an unmanaged memory model: garbage collector 없는 C, C++ 등(python, JAVA는 자동 관리)

file format을 만드는 과정도 이 언어들 내부의 data structure 만드는 과정과 유사(primitives, structures, pointers 등)

- 필요할 때 memory allocation 자유롭게 할 수 있다
    - contiguous memory segment available
    - fragmentation
    - free 이후에 일어나는 일
    - garbage collection

Data layout 중요성이 in-memory 보다 부각

(운영체제와 file system이 일부 보조하지만 여전히 직접 고려해야한다)

- 빠른 access 가능해야함
- persistent storage medium의 특성 고려
- binary data format
- serialize, deserialize 수단
- constraints
    - predefined size
        - variable-length fields
        - oversize data
    - explicit allocation, freement tracking

### Binary Encoding(efficient page layouts)

- serialize, deserialize하기 좋은 형태로 encode
- layout: malloc, free 대신에 read, write만 있다

→ binary format들이 모두 동일한 방식으로 작동

- create file
- serialization formats
- communication protocols

record를 page로 정리하기 전에 설명하는 개념들

- key와 data record를 binary form으로 표현하는 법
- 여러 개 value를 complex structure로 합치는 법
- variable size types, arrays의 implementation

### Primitive Types

type: interger, date, string 등 key와 value가 가지는 유형 분류

type은 binary forms로 serialized/deserialized 가능

대부분 numeric data type은 고정된 크기

encoding과 decoding시 같은 byte-order(endianness) 반드시 통일

- Big-endian: MSB부터 주소가 증가하는 방향으로 저장
- Little-endian: LSB부터 주소가 증가하는 방향으로 저장

RocksDB의 경우

platform specific definition으로 대상의 platform byte order 확인

EnodeFixed64WithEndian이 target과 value의 endian이 일치하지 않는 것 발견시 EndainTransform 사용

numbers, strings, booleans 혹은 그들의 조합으로 된 기록들은 network로 전송하거나 disk에 저장시 serialized 되어야 한다

- byte 8bits
- short 2 bytes
- int 4bytes
- long 8 bytes
- floating point numbers(float, double 등): approximation
    - sign 1bits
    - exponent 8bits
    - fraction 23bits

### Strings and Variable-Size Data

primitive numeric type은 고정된 크기

- combine primitive values into structures
- fixed sized arrays or pointers

string, variable-size datatypes 등 크기가 변하는 type

- number
- size bytes: actual data로 앞의 number가 크기를 표현

string의 경우

- USCD string
- Pascal string
    - constant time 내에 string의 길이 알 수 있음
    - language specific string 구성 가능(size byte를 잘라서 string constructor에 전달)
- null-terminated strings: byte-wise로 end-of-string symbol까지 읽음

### Bit-Packed Data: Booleans, Enums, and Flags

Bit-Packed Data: 공간을 절약하기 위해 bit 단위로 데이터를 압축해서 저장하는 방식

1. Booleans
    
    true/false의 정보를 담는 type
    
    1  bit만 차지하므로 8개씩 저장하는 등 방식
    
    - 1 set
    - 0 unset, empty
2. Enums(Enumerated types)
    - intergers
    - binary formats
    - communication protocols
    - low-cardinality values

B-Tree node type을 enum을 사용해서 구현

enum NodeType{ ROOT, INTERNAL, LEAF}; 3개 비트로 충분

1. Flags
    
    pack된 boolean과 enum의 조합
    
    bit mask로 2의 거듭제곱만 사용
