## ORDER BY 처리
레코드를 정렬하기 위해 인덱스를 고를 수도 있지만 인덱스를 통해 모든 처리하는 것은 현실적으로 불가능하다. 그러한 이유를 정리해보면 아래와 같다.
- 정렬 기준이 너무 많아 요건 별로 모든 인덱스를 생성하는 것이 불가능한 경우
- GROUP BY의 결과 또는 DISTINCT 같은 처리의 결과를 정렬해야 하는 경우
- UNION의 결과와 같이 임시 테이블의 결과를 다시 정렬해야 하는 경우
- 랜덤하게 결과 레코드를 가져와야 하는 경우

MySQL에서는 정렬을 수행하기 위해 별도의 메모리 공간을 할당받아 사용하는데, 이 메모리 공간을 소트 버퍼라 한다.

소트 버퍼는 어느정도 일정 수준이상의 크기를 가지게 되면 성능이 향상된다. 그 이유를 이해하기 위해선 정렬해야 할 레코드의 건수가 소트 버퍼로
할당된 크기보다 큰 경우를 이해하면 된다.

MySQL은 정렬해야 할 레코드를 여러 조각으로 나눠서 처리하고 이 과정에서 임시 저장을 위해 디스크를 쓴다. 이후 다음 레코드를 가져와 다시 정렬해
반복적으로 디스크에 임시 저장한다. 이러한 작업을 멀티 머지라고 하는데 설정한 소트 버퍼의 크기만큼 정렬된 레코드를 병합 후 임시 저장을 반복하니
이 횟수를 줄일 수 있어서 소트 버퍼의 크기를 키우면 어느정도의 성능 향상을 기대할 수 있다. 

※ 참고로 이 멀티 머지 횟수는 sort_merge_passes라는 상태 변수에 누적돼 집계된다.

정렬 과정에서 데이터가 많아질 수록 파일소트 과정에서 임시 저장으로 디스크 접근 횟수가 줄어드니 소트 버퍼의 크기가 무조건 크면 좋을 것 같지만 
그렇지 않다. 그러한 이유는 두 가지로 설명을 할 수 있다.

1. 메모리가 지나치게 할당되면 리눅스 계열 운영체제가 프로세스를 강제종료할 수 있다. 소트 버퍼 설정은 여러 클라이언트가 공유해서 설정되는
   부분이다. 그 말은 소트 버퍼의 설정 크기가 바뀌면 MySQL 내의 커넥션 개수를 곱한 만큼 추가 소트 버퍼용 메모리를 할당한다는 것이다. 즉, 메모리 낭비가 심해진다.

2. 추가적으로 리눅스 계열 운영체제는 메모리 부족 현상을 겪으면 메모리 확보를 위해 메모리를 가장 많이 쓰는 프로세스를 OOM-Killer가 동작해 강제종료한다.
   당연히 메모리를 많이 쓰는 MySQL이 1순위이기 때문에 소트 버퍼의 크기를 무작정 키우는 건 좋지 않다.

그래서, Real MySQL에서는 대량 데이터 정렬이 필요한 경우 대량 데이터 정렬이 일어나는 세션만 소트 버퍼의 크기를
조정하여 처리하고 글로벌 설정은 크기를 작게 설정하는 것을 권장한다.

---

## 정렬 알고리즘

레코드를 정렬 할 때 레코드 전체를 소트 버퍼에 담을지 또는 정렬 기준 칼럼만 소트 버퍼에 담을지에 따라 "싱글 패스(single-pass)"와 "투 패스(Two-pass)"
2가지 정렬 모드로 나눌 수 있다. 

아래와 같이 설정을 해놓고 쿼리를 실행하면 결과에서 sort_algorithm이라는 부분에 sort_mode라는 이름으로 정렬 방식을 표기한 걸 볼 수 있다.

```MySQL
SET OPTIMIZER_TRACE="enable=on", END_MARKERS_IN_JSON=on;
SET OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;
```

MySQL 상에서는 3가지의 방식으로 동작을 한다. 설명 그대로 작성을 하지만 싱글 패스 방식과 투 패스 정렬 방식을 이해하면 쉽게 이해할 수 있다.

- <sort_key, rowid>: 정렬 키와 레코드의 로우 아이디(Row ID)만 가져와서 정렬하는 방식
- <sort_key, additional_fields>: 정렬 키와 레코드 전체를 가져와서 정렬하는 방식으로, 레코드의 칼럼들은 고정 사이즈로 메모리에 저장
- <sort_key, packed_additional_fields>: 정렬 키와 레코드 전체를 가져와서 정렬하는 방식으로, 레코드의 컬럼들은 가변 사이즈로 메모리에 저장

### 싱글 패스 정렬 방식
소트 버퍼에 정렬 기준 칼럼을 포함해 <u><strong>"SELECT 대상이 되는 칼럼 전부"</strong></u>를 담아 정렬을 수행하는 정렬 방식이다. 정렬에 필요하지 않은 칼럼까지
모두 읽어 소트 버퍼에 담아 정렬을 한다는 단점이 있어 한번에 많은 레코드를 처리하기는 힘들다는 단점이 있다.

하지만, 이 방식의 큰 장점은 해당 정렬이 끝나고 즉시 반환한 게 결과가 돼서 추가적인 데이터 접근이 필요가 없어 속도가 상대적으로 빠르다.
또한, SELECT 대상이 되는 칼럼 모두를 담는다는 특성이 정말 중요해서 강조 표시도 해놓았는데. SELECT * FROM TABLE 같은 방식으로 개발을 하면 소트 버퍼의 크기보다
커서 싱글 패스를 옵티마이저가 선택하기 어려워지면 결국 투 패스 방식으로 한 번 더 실제 데이터가 존재하는 공간으로 접근해 처리를 해야 한다.
따라서, 개발하는 과정에서 SELECT를 할 때에는 *로 모든 컬럼을 선택하는 것보다는 필요한 컬럼만으로 최소화해서 가져오는 습관을 가지는 것이 좋다.

### 투 패스 정렬 방식
투 패스 정렬 방식은 정렬 대상 칼럼과 프라이머리 키 값만 소트 버퍼에 담아 정렬을 수행하고 정렬된 순서대로 프라이머리 키를 통해 테이블에 읽어
SELECT할 칼럼을 가져오는 정렬 방식이다. 

프라이머리 키와 정렬이 필요한 칼럼만 가지고 정렬을 메모리에서 수행하기 때문에 한 번에 더 많은 양의 레코드를 정렬할 수 있어 디스크 접근을 최소화할 수 있다.
결국 다시 프라이머리 키로 데이터에 접근해야 한다는 단점이 있지만 정렬해야 하는 데이터가 매우 많은 경우에 유용하게 사용된다.

가급적이면 싱글 패스가 대부분의 상황에는 좋지만 현실적으로 선택하기가 어려운 경우엔 옵티마이저가 투 패스 정렬 방식을 선택하며 그러한 예시는 아래와 같다.

- 레코드의 크기가 max_length_for_sort_data 시스템 변수에 설정된 값보다 큰 경우
- BLOB이나 TEXT 타입의 칼럼이 SELECT 대상에 포함이 되는 경우

---

## 정렬 처리 방법

쿼리에 ORDER BY가 사용되면 3가지 방법 중 하나로 정렬이 처리된다. 아래의 표에서 아래로 갈 수록 처리 속도가 느려진다.

|          정렬 처리 방법          |             실행 계획의 Extra 칼럼 내용              |
|:--------------------------:|:-------------------------------------------:|
|        인덱스를 사용한 정렬         | 별도 표기 없음 |
|     조인에서 드라이빙 테이블만 정렬      | "Using filesort" 메시지가 표시됨 |
| 조인에서 조인 결과를 임시 테이블로 저장후 정렬 | "Using temporary; Using filesort" 메시지가 표시됨 |

인덱스를 사용할 수 있다면 별도의 "Filesort" 과정이 필요없으니 인덱스를 순서대로 읽어 결과를 반환할 수 있다. 하지만
인덱스를 쓸 수 없다면 WHERE 조건에 일치하는 레코드를 정렬 버퍼에 저장해 정 처리를 한다. 인덱스를 쓸 수 없을 때 방식은 두 가지로 분류된다.

- 조인의 드라이빙 테이블만 정렬한 다음 조인을 수행
- 조인이 끝나고 일치하는 레코드를 모두 가져온 후 정렬 수행

### 인덱스를 이용한 정렬
인덱스를 정렬에 쓰기 위해선 ORDER BY에 명시된 칼럼이 먼저 읽는 테이블에 속하고, ORDER BY 순서대로 생성된 인덱스가 필요하다.
또한 WHERE 절에 첫 번째로 읽는 테이블의 칼럼에 대한 조건이 있다면 그 조건과 ORDER BY는 같은 인덱스를 사용할 수 있어야 한다.

인덱스를 사용하면 실제 인덱스의 값이 정렬이 돼 있어 인덱스의 순서대로 읽기만 하면 된다. 그래서, 별도의 추가 작업이 없다.

※ 인덱스와 ORDER BY 절에서의 주의사항

ORDER BY가 부가적인 작업이 일어날 수 있으니, 인덱스를 탈 거라 기대하고 넣지 않는 경우가 있다. 
이는 데이터 특성의 변화에 따라 실행 계획이 바뀌면 버그로 이어질 수 있으니 생략해서는 안된다. 인덱스는 어차피
부가적인 ORDER BY를 위한 정렬처리를 안하니 성능 문제가 발생하지 않기 때문이다.

### 조인의 드라이빙 테이블만 정렬
조인을 하면 결과 레코드가 급격하게 불어난다. 그래서 조인이 일어나기 전에 첫 번쨰 테이블의 레코드를 정렬한 뒤 레코드를 줄여서
처리하면 데이터를 최소화할 수 있다. 이를 통해 인덱스가 걸리지 않은 경우에도 적은 메모리로 정렬된 데이터를 얻을 수 있다.

대신 이 방법에서는 첫 번째로 읽히는 테이블의 컬럼만으로 ORDER BY 절을 작성해야 한다.

```SQL
SELECT * FROM employees e, salaries s 
         WHERE s.emp_no = e.emp_no
         AND e.emp_no BETWEEN 100002 AND 100010
         ORDER BY e.last_name;
```

위의 구문은 WHERE 절에 PK로 된 값인 emp_no로 선택할 레코드를 크게 줄일 수 있고, 두 테이블 사이를 emp_no라는 인덱스로
가지고 올 수 있기 때문에 employee 테이블을 드라이빙 테이블로 설정한다. 단, 주의사항이 검색으로 값을 좁히는 용으로는 인덱스를 쓸 수 있지만,
인덱스를 통해서 정렬을 하는 것은 불가능하다.

위의 상황은 지금 드라이빙 테이블로 선택된 employee 테이블 컬럼에 해당하는 필드인 last_name만을 통해 정렬하므로 인덱스 레인지 스캔으로 값을 가져오고
last_name 필드로 정렬을 메모리 내에서 수행(파일소트)하고 조인을 하는 방식으로 동작을 하게 된다. 
참고로 이 방식에서는 2개 이상의 테이블이 조인되면서 정렬이 실행되지만 임시 테이블까지는 사용하지는 않는다.

### 임시테이블을 이용한 정렬 방식
하나의 테이블을 조회해 정렬한다면 임시 테이블이 필요없지만 서브쿼리나 여러 테이블을 조인해서 결과를 정렬해야 하는 경우엔 임시 테이블이 필요하다.

이 방법은 정렬할 레코드가 가장 많기에 가장 느린 방법이다. 이번엔 앞의 예제와 다르게 ORDER BY를 드라이빙 테이블로 선택 가능한 employee가 아닌
드리븐 테이블에 존재하는 s.salary로 ORDER BY를 하는 경우를 살펴보자.

```SQL
SELECT * FROM employees e, salaries s 
         WHERE s.emp_no=e.emp_no
         AND e.emp_no BETWEEN 100002 AND 100010
         ORDER BY s.salary;
```

위의 쿼리는 정렬 기준이 드라이빙 테이블이 아닌 드리븐 테이블에 있기 때문에 해당 쿼리는 조인된 데이터를 가지고서 정렬할 수 밖에 없다.
그래서 일단 조인된 결과를 임시테이블로 저장하고 임시 테이블로 정렬을 하는 방식으로 동작하며 실행 계획 확인 시
"Using temporary: Using filesort"라는 코멘트가 표시된다.

※ 이해 확인용 예제

```SQL
SELECT * FROM tb_test1 t1, tb_test2 t2
         WHERE t1.col1 = t2.col1
         ORDER BY t1.col2
         LIMIT 10;
```

지금 위의 쿼리는 드라이빙 테이블이 어떤 거로 선정되냐와 정렬 방법에 따라 읽는 건수, 조인 횟수, 정렬해야 할 대상 건수가 달라진다.
tb_test1 테이블의 레코드가 100건이고, tb_test2 테이블의 레코드가 1000건(tb_test1의 레코드 1건 당, tb_test2의 레코드가 10개씩)
있다고 생각하고 표 값을 예상해보자.

tb_test1이 드라이빙 되는 경우

|      정렬 방법       |             읽어야 할 건수              | 조인 횟수 | 정렬해야 할 대상 건수 |
|:----------------:|:---------------------------------:|:-----:|:------------:|
|      인덱스 사용      |   tb_test1: 1건<br>tb_test2: 10건   |  1번   |      0건      |
| 조인의 드라이빙 테이블만 정렬 |  tb_test1: 100건<br>tb_test2: 10건  |  1번   |     100건     |
|  임시 테이블 사용 후 정렬  | tb_test1: 100건<br>tb_test2: 1000건 | 100번  |    1000건     |


tb_test2이 드라이빙 되는 경우

|      정렬 방법       |             읽어야 할 건수              | 조인 횟수 | 정렬해야 할 대상 건수 |
|:----------------:|:---------------------------------:|:-----:|:------------:|
|      인덱스 사용      |  tb_test2: 10건<br>tb_test2: 10건   |  10번  |      0건      |
| 조인의 드라이빙 테이블만 정렬 | tb_test2: 1000건<br>tb_test1: 10건  |  10번  |    1000건     |
|  임시 테이블 사용 후 정렬  | tb_test2: 1000건<br>tb_test2: 100건 | 1000번 |    1000건     |

정렬에 있어서 인덱스 설정도 중요하지만 어떤 드라이빙 테이블을 선택하게 하냐도 중요함을 알 수 있다. 또한, Filesort 작업을 거치는
쿼리는 LIMIT 조건이 크게 도움이 안됨을 기억하자.

위의 개념에서 읽어야할 건수는 인덱스를 잘 활용하면 필요한 데이터만 읽는다는 점, 
드라이빙 테이블을 사용하면 드라이빙 테이블은 모두 읽지만 조인은 인덱스와 동일하게 필요한 값만 한다는 점,
그리고 임시 테이블은 그냥 모든 데이터를 조인해서 한 번에 메모리에 올려놓고 정렬한다는 것만 기억하면 크게 헷갈리지는 않을 것으로 생각합니다.

---

## 정렬 처리 방법의 성능 비교
웹 서비스에서는 보통 ORDER BY와 함께 LIMIT이 사용이 같이 된다. 하지만, ORDER BY나 GROUP BY 같은 작업은 주어진 조건을
만족하는 레코드를 모두 가져와 정렬을 수행하거나 GROUPING을 해야 LIMIT으로 건수를 제한할 수 있으므로 인덱스를 잘 걸더라도
ORDER BY나 GROUP BY에 의한 성능 저하가 일어날 수 있다.

### 스트리밍 방식
서버 쪽에서 처리할 데이터가 얼마인지 관계없이 조건에 만족하는 레코드가 검색될 떄마다 즉시 클라이언트에게 반환하는 방식이다.
이런 방식으로 동작을 하면 빠른 응답 시간을 보장해줄 수 있다.

### 버퍼링 방식
MySQL 서버에서 모든 레코드를 검색하고 정렬 작업을 하는 동안엔 클라이언트는 아무것도 하지 않고 기다려야 하기에 응답 속도가 느려진다.
결국 버퍼링 방식으로 처리되는 것은 결과를 모아 가공해야 응답이 가능하므로 LIMIT을 걸더라도 성능이 크게 바뀌지 않을 가능성이 높다.

참고로 JDBC는 응답 결과를 모두 받을 때까지 대기하며 MySQL에서 스트리밍 방식으로 보내는 데이터를 모아서 한번에 보내는 방식으로 동작한다.
이 방법은 불필요한 네트워크를 줄이고 서버 통신에서의 불필요한 자원 소모를 없앨 수 있다고 한다. 설정을 통해 MySQL과 JDBC 간 전송 방식을
스트리밍 방식으로 변경도 가능은 하다.

---

## GROUP BY 처리 

GROUP BY 작업은 인덱스를 사용하는 경우와 사용하지 않는 경우로 나눠 볼 수 있다. 인덱스를 사용할 때에는 인덱스를 차례대로 읽는
인덱스 스캔 방법과 인덱스를 건너뛰면서 읽는 루스 인덱스 스캔 방식으로 나눌 수 있다. 인덱스를 쓰지 못하는 쿼리에서의 GROUP BY 작업은
임시 테이블을 사용한다.

### 인덱스 스캔을 이용하는 GROUP BY(타이트 인덱스 스캔)

조인의 드라이빙 테이블에 속한 컬럼만 이용해 그루핑할 때 GROUP BY 칼럼에 이미 인덱스가 있다면 그 인덱스를 차례대로 읽으며 그루핑 작업을 수행한 뒤
결과로 조인을 처리한다. 하지만 그룹 함수의 그룹 값 처리를 위해 임시 테이블을 사용할 때가 있다.(SUM 같은 집계함수 사용부분으로 뒤에 나옴.)

이 때, 인덱스를 읽을 때 쿼리 실행 시점에 추가적인 정렬 작업이나 내부 임시 테이블은 필요하지 않지만 실행 계획에서 "Using index for group-by"나 
"Using temporary, Using filesort"가 표시되지는 않는다.

### 루스 인덱스 스캔을 사용하는 GROUP BY
루스 인덱스 스캔 방식은 인덱스의 레코드를 건너뛰면서 필요한 부분만 읽어 가져오는 것을 의미하는데. 
이 방식을 사용할 떈 실행 계획에 "Using index for group-by"라는 코멘트가 표시된다.

```MySQL
EXPLAIN
  SELECT emp_no
  FROM salaries
  WHERE from_date='1985-03-01'
  GROUP BY emp_no;

인덱스 상태: (emp_no, from_date)
```

위의 쿼리에서는 WHERE 조건만으로는 선두컬럼과 일치하지 않으므로 인덱스 레인지 스캔 방식이 불가능하지만 실제 쿼리 계획을 살펴보면
인덱스 레인지 스캔을 GROUP BY 처리를 위해 사용했음을 확인할 수 있다. 이 과정은 다음과 같이 분리해서 볼 수 있다.

1. (emp_no, from_date) 인덱스를 차례대로 스캔하며 emp_no의 첫 번쨰 유일한 값(그룹 키) "10001"을 찾는다.
2. (emp_no, from_date) 인덱스에서 emp_no가 '10001'에서 from_date 값이 '1985-03-01'인 레코드만 가져온다. 해당 과정에서
   where '10001' and from_dat='1985-03-01'처럼 동작하므로 인덱스 스캔이 가능하다.
3. (emp_no, from_date) 인덱스에서 emp_no의 그 다음 유니크한(그룹 키)를 가져온다.
4. 유니크한 emp_no가 없다면 이 루프를 종료하고 있다면 2번 과정부터를 반복한다.

이 과정을 보면 보통 인덱스를 걸었을 때 유니크한 값(카디널리티)가 많을수록 빠르지만 루스 인덱스 스캔을 쓰는 경우에는 반대로
유니크한 값의 수가 적을수록 성능이 증가하게 된다. 또한 이 과정에서는 인덱스를 사용하므로 별도의 임시 테이블을 생성하지는 않는다.

루스 인덱스 스캔 판별이 어렵고 자신이 없지만 아래의 예시를 보고 참고해서 감을 익히는 게 중요하다고 생각을 한다.

* 루스 인덱스로 동작하는 쿼리들
```MySQL
인덱스 컬럼 (col1, col2, col3)
SELECT col1, col2 FROM tb_test GROUP BY col1, col2;
SELECT DISTINCT col1, col2 FROM tb_test;
SELECT col1, MIN(col2) FROM tb_test GROUP BY col1;
SELECT col1, col2 FROM tb_test WHERE col1 < const GROUP BY col1, col2;
SELECT MAX(col3), MIN(col3), col1, col2 FROM tb_test WHERE col2 > const GROUP BY col1, col2;
SELECT col2 FROM tb_test WHERE col1 < const GROUP BY col1, col2;
SELECT col1, col2 FROM tb_test WHERE col3 = const GROUP BY col1, col2;
```

5번째 쿼리는 MAX(col3), MIN(col3) 같은 함정이 있지만 GROUP BY 절에서 일치하고 WHERE 절에서는 col2의 범위만 좁히는 역할을 하기 때문에 가능하다.
7번째 쿼리는 WHERE 절에 const가 있더라도 인덱스가 (col1,col2,col3)라 가능한 경우이다.

반대로 다음의 쿼리들은 루스 인덱스 스캔을 사용할 수 없는 쿼리 패턴이다. 인덱스는 위와 동일하여 생략한다.

```MySQL
-- // MIN()과 MAX() 이외의 집합 함수가 사용되었기 때문에 루스 인덱스 스캔은 사용이 불가능
SELECT col1, SUM(col2) FROM tb_test GROUP BY col1;

-- // GROUP BY에 사용된 컬럼이 인덱스 구성 컬럼의 선두 컬럼과 일치하지 않아 사용 불가
SELECT col1, col2 FROM tb_test GROUP BY col2, col3;

-- // SELECT 절의 칼럼이 GROUP BY와 일치하지 않기 때문에 사용 불가
SELECT col1, col3 FROM tb_test GROUP BY col1, col2;
```

### 임시 테이블을 사용하는 GROUP BY
GROUP BY를 사용할 때 인덱스를 사용하지 못할 땐 상황과 관계없이 임시 테이블을 사용한다.

```MySQL
인덱스(emp_no, from_date)
EXPLAIN
  SELECT e.last_name, AVG(s.salary)
  FROM employees e, salaries s
  WHERE s.emp_no=e.emp_no
  GROUP BY e.last_name;
```

위의 쿼리를 실행해보면 인덱스를 전혀 활용할 수 없기 때문에 임시 테이블을 사용해야 한다. 
이 때는 인덱스를 사용할 때 GROUP BY에서 아무런 추가적인 Extra 표기가 없던 것과 달리 "Using temporary"라는 메시지가 표시된다.

여기서 생각해볼 것은 MySQL 버전에 따른 차이이다. MySQL 8.0부터는 GROUP BY가 필요할 때 GROUP BY 절의 칼럼들로 구성된 유니크 인덱스
를 가진 임시 테이블을 통해 중복 제거와 집합 함수 연산을 수행한다. 
그렇게 조인의 결과를 한 건씩 가져와 임시 테이블에서 중복 체크를 하기에 별도의 정렬 작업이 필요가 없다.

하지만 반대로 그 이전 버전에서는 GROUP BY를 임시 테이블로 동작하는 경우 묵시적으로 ORDER BY 정렬까지 수행이 됐었다.
이를 방지하기 위해 ORDER BY NULL과 같은 방식으로 정렬을 회피하고는 했는데 더이상 그럴 필요가 없다.

대신 MySQL 8.0 버전에서 ORDER BY까지 정의하게 되면 명시적으로 정렬 작업을 적용해서 "Using temporary; Using filesort"
를 볼 수 있다.

### DISTINCT 처리
특정 칼럼의 유니크한 값만을 조회할 때는 SELECT 쿼리에 DISTINCT를 사용한다. 
이 때 인덱스를 통해 처리를 못하면 항상 임시 테이블이 필요한데, 실행 계획에서는 "Using temporary" 메시지가 출력되지 않는다.

### SELECT DISTINCT
단순히 유니크한 레코드를 가져오고자 할 때는 SELECT DISTINCT 형태의 쿼리 문장을 사용한다. 이 땐 GROUP BY와 동일하게 처리가 된다. 
GROUP BY와 동일하게 DISTINCT 또한 ORDER BY절이 없으면 정렬을 시행하지 않는다.

그럼 DISTINCT를 여러 컬럼에 대해서 조회하는 아래의 차이는 무엇일까?
```MySQL
SELECT DISTINCT first_name, last_name FROM employees;
SELECT DISTINCT(first_name), last_name FROM employees;
```

둘의 차이는 없다. MySQL에서는 저렇게 괄호를 치더라도 무시한다. 그리고, 위는 first_name에 대한 유니크한 값이 아닌
first_name과 last_name을 조합한 유니크한 값을 가져오는 조회임을 꼭 기억하자.

### 집합 함수와 함께 사용된 DISTINCT
COUNT(), MIN(), MAX() 같은 집합 함수 내에서 DISTINCT 키워드가 사용되는 경우 일반적인 SELECT DISTINCT와 다르게 해석된다.
앞에서 DISTINCT는 조회한 모든 칼럼의 조합이 유니크한 걸 가져오는 것을 보았을 것이다. 그래서 괄호를 쳐도 first_name만 유니크하게 가져오지 않고 생략하고 first_name과 last_name에 대해서 유니크하게 가져온다 하였다.

하지만, 집합 함수 내에서 사용된 DISTINCT는 그 집합 함수의 인자로 전달된 칼럼 값이 유니크한 걸 가져온다. 이 말이 헷갈리는데
예시를 보면 이해할 수 있다.

```MySQL
EXPLAIN
  SELECT COUNT(DISTINCT s.salary)
  FROM employees e, salaries s
  WHERE e.emp_no=s.emp_no
  AND e.emp_no BETWEEN 100001 AND 100100;
```
위의 쿼리는 "COUNT(DISTINCT s.salary)"를 처리하기 위해 임시 테이블을 쓴다.
하지만 임시 테이블 메시지는 표시되지 않는다. 
게다가 salary 칼럼의 값만을 저장하기 위한 임시 테이블을 만들고 유니크 인덱스가 생성되기에 레코드 건이 늘어나면 매우 느리다.

```MySQL
EXPLAIN
  SELECT COUNT(DISTINCT s.salary),
         COUNT(DISTINCT e.last_name)
  FROM employees e, salaries s
  WHERE e.emp_no=s.emp_no
  AND e.emp_no BETWEEN 100001 AND 100100;
```
위의 경우에는 이전에 DISTINCT를 여러 컬럼에 걸쳐 사용했을 때, 모든 컬럼에 대해서 묶어서 하나로 처리해 임시 테이블을 만들어 사용해
임시 테이블은 하나만 생성이 됐었다. 
하지만 위의 경우에는 각각의 salary 칼럼 값을 저장하는 임시 테이블과 last_name 칼럼의 값을 저장하는 다른 임시 테이블 이렇게 2개의 임시 테이블을 쓴다.

### 내부 임시 테이블 활용, 메모리 임시 테이블과 디스크 임시 테이블

MySQL 엔진에서 레코드를 정렬하거나 그루핑할 땐 임시 테이블을 사용한다.
MySQL 엔진이 사용하는 임시 테이블은 처음 메모리에 생성됐다가 테이블의 크기가 커지면 디스크로 옮겨진다.
특정 예외 케이스는 메모리를 거치지 않고 바로 디스크에 임시 테이블이 만들어지기도 한다.

임시 테이블은 다른 세션이나 다른 쿼리에서는 볼 수 없고 사용하는 것이 불가능하며, 내부적인 임시 테이블은 쿼리 처리가 완료되면 자동으로 삭제된다.

MySQL 8.0 버전부터는 메모리는 TempTable이라는 스토리지 엔진을 사용하고 디스크에 저장되는 임시 테이블은 InnoDB 스토리지 엔진을 사용하도록 개선됐다.
기존 MEMORY 스토리지 대비 가변 길이도 지원하기 때문에 메모리 낭비를 줄여준다.

TempTable을 사용하는 경우엔 temptable_max_ram 시스템 변수로 TempTable이 최대한 사용 가능한 메모리 공간의 크기를 설정할 수 있다.
기본 값은 1GB로 MySQL 서버에서는 임시 테이블의 크기가 1GB보다 커지는 경우 메모리의 임시 테이블을 디스크로 기록한다.
이때 MySQL 서버는 MMAP 파일로 디스크에 기록 혹은 InnoDB 테이블로 기록 중 한 방식을 선택한다.

MySQL 서버에선 MMAP 파일로 기록을 temptable_use_mmap 시스템 변수로 설정할 수 있는데, 기본 값은 ON이다.
메모리의 TempTable 크기가 1GB를 넘으면 MySQL 서버는 TempTable을 MMAP 파일로 전환하는데 이는 InnoDB 테이블로 전환하는 것보다 오버헤드가 적다.
이 때 임시 테이블은 tmpdir 시스템 변수에 정의된 디렉터리에 저장된다. 하지만, 이는 MYSQL 서버 종료 혹은 쿼리 종료 시 임시 테이블의 삭제를 보장하기 때문에 볼 수 없다.

### 임시 테이블이 필요한 쿼리
아래의 쿼리는 MySQL 엔진 내에서 별도의 데이터 가공 작업이 필요해 내부 임시 테이블을 생성한다. 혹은 인덱스를 사용하지 못할 때에도 내부 임시 테이블을 생성한다.

- ORDER BY와 GROUP BY에 명시된 칼럼이 다른 쿼리
- ORDER BY나 GROUP BY에 명시된 칼럼이 조인의 순서상 첫 번째 테이블이 아닌 쿼리
- DISTINCT와 ORDER BY가 동시에 쿼리에 존재하는 경우 DISTINCT가 인덱스로 처리되지 못하는 쿼리
- UNION이나 UNION DISTINCT가 사용된 쿼리
- 쿼리 실행 계획에서 select_type이 DERIVED인 쿼리

위의 3가지는 실행 계획을 살펴볼 때 Extra 칼럼에 "Using temporary"라는 메시지가 표시된다.
하지만, 아래의 3개는 임시 테이블을 쓰지만 "Using temporary"가 표시되지 않는다.

위의 첫 번째부터 네 번째까지의 쿼리 패턴은 유니크 인덱스를 가지는 내부 임시 테이블을 만들고 
마지막 쿼리는 유니크 인덱스가 없는 임시 테이블을 생성한다. 이 때 유니크 인덱스가 있는 내부 임시 테이블은 그렇지 않는 쿼리보다 성능이 느리다.

### 임시 테이블이 디스크에 생성되는 경우
아래의 경우엔 디스크 기반의 임시 테이블을 사용한다.

- UNION이나 UNION ALL에서 SELECT되는 칼럼 중에서 길이가 512바이트 이상인 크기의 칼럼이 있는 경우
- GROUP BY나 DISTINCT 칼럼에서 512바이트 이상인 크기의 칼럼이 있는 경우
- 메모리 임시 테이블 크기가 temptable_max_ram 시스템 변수 값보다 큰 경우

참고로 MySQL 8.0.13 버전에서 BLOB이나 TEXT 칼럼을 가진 임시 테이블에 대해서도 메모리에 임시 테이블 생성이 가능하다.

### 임시 테이블 관련 상태 변수
실행 계획 상에서 "Using temporary"가 표시되지 않아도 임시 테이블이 생성되는 경우가 있다.
그리고 임시 테이블을 하나만 사용하지 않고 여러 개 사용하는 경우, 디스크에 생성한 건지 메모리에 생성한 건지 알고 싶다면 아래의 쿼리를 통해 확인 가능하다.

```MySQL
SHOW SESSION STATUS LIKE 'Created_tmp%';
```