## MVCC(Multi-Version Concurrency Control, 다중 버전 동시성 제어)

### 1. MVCC
MVCC는 동시 접근을 허용하는 데이터베이스에서 동시성을 제어하기 위해 사용하는 방법 중 하나이다.
MVCC는 원본의 데이터와 변경중인 데이터를 동시에 유지하는 방식으로, 원본 데이터에 대한 Snapshot을 이용하여 데이터의 동시성을 제어한다.


#### 특징
- 일반적인 RDBMS보다 매우 빠르게 작동 (잠금을 필요로하지 않기 때문에)
- 사용하지 않는 데이터가 계속 쌓이게 되므로 데이터 정리 시스템 필요
- 데이터 버전이 충돌하면 애플리케이션 영역에서 이 문제를 해결해야 함


### MVCC에서의 스냅샷 읽기(Snapshot Read)
- 스냅샷 읽기는 MVCC(Multi-Version Concurrency Control)에서 중요한 개념이다.
- 트랜잭션이 시작될 때의 데이터베이스 상태를 스냅샷으로 저장하여 그 시점의 일관된 데이터를 읽을 수 있도록 하는 방식
- 이를 통해 여러 트랜잭션이 동시에 실행되더라도 데이터 일관성이 유지되며, 읽기 작업이 다른 트랜잭션의 쓰기 작업에 의해 방해받지 않는다.


### 1. 트랜잭션 격리 수준 관련
- Repeatable Read
  - 이 격리 수준에서는 트랜잭션이 시작된 시점의 스냅샷이 유지된다.
  - 동일한 트랜잭션 내에서 반복적으로 같은 데이터를 읽을 때, 트랜잭션이 시작된 시점의 데이터와 동일한 결과를 보장한다.
- Read Committed
  - 트랜잭션이 커밋될 때마다 새로운 스냅샷을 사용하여 데이터를 읽는다.
  - 따라서 트랜잭션 내에서 반복해서 읽기를 수행하면 최신의 커밋된 데이터를 볼 수 있다.


### 2. Undo 로그 관련
- 스냅샷 읽기는 Undo 로그를 활용하여 구현된다.
- 트랜잭션이 스냅샷을 생성할 때, 다른 트랜잭션이 데이터를 수정하고 커밋했더라도 Undo 로그를 통해 해당 데이터의 이전 버전을 읽어와 스냅샷 일관성을 유지한다.
- 이를 통해 데이터의 여러 버전이 동시에 존재할 수 있게 된다.


### 3. 가시성(Visibility) 판단
- InnoDB는 트랜잭션의 ID(Transaction ID)와 타임스탬프를 이용하여 어떤 데이터 버전이 특정 트랜잭션에 대해 가시성을 가지는지 판단한다.
- 예를 들어, 특정 트랜잭션이 커밋된 시점보다 이전에 생성된 데이터는 현재 트랜잭션에 가시적이게 된다.
- 가시성 판단을 통해 MVCC는 트랜잭션 간의 격리성을 유지하며 데이터의 일관된 읽기를 보장한다.



#### MySQL에서의 MVCC : Undo Log 활용
- 과정 : [MVCC_supplement.md](https://github.com/Pearl-K/database_study/blob/main/chapter5/source/MVCC_supplement.md#%EC%96%B8%EB%91%90-%EB%A1%9C%EA%B7%B8-undo-log-%ED%99%9C%EC%9A%A9)
- Undo Log 영역의 데이터는 COMMIT or ROLLBACK 을 호출하여 InnoDB 버퍼풀도 이전의 데이터로 복구되고, 더 이상 Undo 영역을 필요로 하는 트랜잭션이 없을 때 비로소 삭제된다.


***

<p align="center"><img src="https://github.com/Pearl-K/database_study/blob/main/chapter5/source/image/undo%20log%20arch.png" width="650" height="350"/></p>

### 2. Undo 로그
- Undo 로그는 트랜잭션이 데이터베이스 내에서 데이터를 변경할 때, 변경 이전의 상태를 저장하는 로그이다.
- 트랜잭션이 취소되거나 오류가 발생할 경우, Undo 로그를 활용하여 데이터를 원래 상태로 되돌릴 수 있다.
- 격리 수준과의 관계:
  - Read Uncommitted: 다른 트랜잭션이 커밋되지 않은 데이터를 읽을 수 있다. (버퍼 풀에서 읽는다)
  - Read Committed: 커밋된 데이터만 읽을 수 있다. Undo 로그를 사용하여 트랜잭션이 롤백될 경우 변경 전의 데이터를 복구한다.
  - Repeatable Read: 이 격리 수준에서는 트랜잭션이 시작된 시점의 데이터를 일관되게 보여줘야 한다. Undo 로그를 통해 트랜잭션 중에 변경된 데이터가 아니라 트랜잭션이 시작될 당시의 데이터를 제공한다.
  - Serializable: 가장 엄격한 격리 수준으로, Undo 로그는 트랜잭션이 중단되었을 때 데이터 일관성을 유지하는 데 사용된다.


#### 1. Undo Log 레코드
   - Undo 로그에는 실제로 변경된 컬럼의 이전 값과 기본 키(PK) 값이 저장된다.
   - 클러스터형 인덱스 레코드를 사용하여 트랜잭션에 의한 최신 변경 사항을 취소하는 방법에 대한 정보를 포함한다.


#### 2. 버퍼 풀 데이터 변경
   - 데이터가 변경되면, 변경 사항이 Undo 로그 영역에 반영되기 시작한다. (데이터뿐만 아니라 인덱스의 변경도 포함)
   - 데이터와 인덱스 변경은 임시 메모리 공간인 인서트 버퍼(Insert Buffer)에 먼저 저장된다.
     + 인서트 버퍼(Insert Buffer) : 인덱스 페이지의 업데이트를 효율적으로 처리하기 위한 임시 메모리 공간


#### 3. 체크포인트와 Undo Log
   - 체크포인트 시점에 Undo 로그가 디스크 영역의 Undo 로그 파일에 저장된다.
   - 각 일반 행에는 실행 취소 로그에 있는 동일한 행의 최신 버전에 대한 포인터가 있다.
   - 실행 취소 로그의 각 행에는 이전 버전에 대한 포인터가 포함되어 있을 수 있다. (수정된 각 행에 기록 체인 형성)


***


<p align="center"><img src="https://github.com/Pearl-K/database_study/blob/main/chapter5/source/image/InnoDB_arch.png" width="600" height="450"/></p>


### 3. Redo 로그
- Redo 로그는 트랜잭션이 커밋된 후 데이터를 영구적으로 기록하는 로그이다.
- 시스템 장애 발생 시, Redo 로그를 사용하여 최근 커밋된 트랜잭션의 데이터를 복구할 수 있다.
- 모든 격리 수준에서 중요한 역할을 하며, InnoDB의 데이터 일관성과 내구성을 유지하는 역할이다.

#### InnoDB의 Redo 로그 공간과 그 구성
- Redo 로그의 크기: InnoDB 스토리지 엔진은 고정 크기의 Redo 로그 공간을 사용한다.
- 크기 조정: innodb_log_file_size와 innodb_log_files_in_group 설정에 의해 결정, 이 두 값을 곱하여 전체 Redo 로그 공간 설정 가능
- 장점: 쓰기 할 때 I/O 최적화
- 단점: Redo 로그 크기가 커질수록 시스템 충돌 발생 시 복구 시간이 길어질 수 있다.
 + Redo 로그 크기를 설정할 때 적절한 테스트를 통해 크기와 복구 시간 사이 균형을 맞춰 설정해야 함!


### Redo 로그 동작 순서
#### 1. 데이터 변경 및 더티 페이지 표시
- 버퍼 풀에서 데이터가 변경되면 해당 페이지를 수정한 후, 해당 페이지에 대해 더티(Dirty) 마크를 표시한다.


#### 2. Redo 로그 레코드 저장
- 변경된 데이터에 대한 Redo 로그 레코드는 메모리 내의 Double Write Buffer에 저장된다. (데이터 손실 방지를 위함,  디스크에 기록되기 전에 메모리에서 먼저 안전하게 처리)


#### 3. 로그 버퍼로 이동
- Redo 로그 레코드는 Log Buffer(Innodb_log_buffer)로 이동된다. 


#### 4. Redo 로그 파일에 flush
- Log Buffer에 저장된 Redo 로그 레코드를 모아서 Redo Log File에 flush 한다. (디스크에 로그 영구 기록)


#### 5. 체크포인트 수행
- 버퍼 풀 내의 변경된 Dirty 페이지에 대해 체크포인트를 수행하여 System Tablespace에 저장한다.


***


Ref.
- [Undo log](https://myinfrabox.tistory.com/262)
- [Redo log](https://myinfrabox.tistory.com/259)


