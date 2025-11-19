## 좌표값 필터링

### 1. 시행착오

어떤 대상이 움직일 때마다 1초 단위로 EPSG:4326 좌표계의 좌표(위경도) 를 받아 기록한 정보를 지도에 경로로 표현해줘야 할 일이 생겼다.<br>
처음에는 단순하게 경도, 위도를 point geometry로 변환하고, EPSG:3857로 좌표계도 변환한 뒤에 postgis의 st_makeline 함수를 이용해서 linestring을 만들었다.

```sql
-- 단순 문자열 x, y 좌표를 point 로 변환해 1차적으로 point 테이블에 저장
-- ※ 바로 linestring으로 변환하지 않는 이유는 추후 point 정보도 WFS를 통해 함께 활용하기 위함
insert into point_table
(
    ...
    geom
    ...
)
select
    ...
    ST_TRANSFORM(ST_SetSRID(ST_MakePoint(xcrd::double precision, ycrd::double precision), 4326), 3857),
    ...
from coord_table
where
...
```

```sql
-- point로 변환한 좌표들을 grouping하여 linestring으로 변환하여 저장
insert into line_table
(
    ...
    geom
    ...
)
select
    ...
    ST_MakeLine(geom order by reg_ymd, reg_tm),
    ...
from point_table
where
    ...
group by
    reg_ymd
```

간단하게 마무리 했다고 생각했으나, 만들어진 linestring을 확인해 본 결과 정상적인 경로 위에 불특정 주기마다 특정 위치로 좌표값이 튀는 듯한 모습이 관찰되었다. 때문에 원본 좌표데이터에 오류 데이터가 포함되어있다는 것을 알게되었다.

## 2. 원인분석

원본 평문 좌표데이터를 확인해보니 좌표값을 기록하는 기기의 오류로 인해 전원이 공급되지 않을 때 일시적으로 0,0이나 불특정 좌표값이 기록되는 양상을 확인했다.

## 3. 개선방향 탐색

postgis에서 관련하여 편차가 큰 좌표값을 필터링해주는 함수가 있는지 확인해 본 결과, 현재 같은 상황을 해결할 수 있는 함수는 없었다. 비슷한 역할을 하는 솔루션을 추렸으나, 현재 나에게 맞는 용도를 가지는 함수는 없었다.

1. PL/SQL을 이용해 칼만 필터 함수 구현 후 적용 -> 필터링 하는 것이 아닌 평균경로로 바꿔줌. 부적합
2. ST_SimplifyVW -> 형태 단순화 알고리즘이라 속도,시간,연속성을 보지 않음. 부적합
3. ST_SnapToGrid -> 단일 점 이상치는 잘 잡히지만 이동경로의 방향성 고려 못함
4. ST_FilterByM -> M값을 기준으로 필터링 할 수 있으나, 필터링의 기준인 M값을 point에 미리 만들어놔야함. 필터링의 개념으로 활용하기보다는 sequence값을 순차적으로 넣어놓고 linestring 등을 만들 때 범위지정 할 때 유용해보임.

## 4. 개선방향 결정

마땅한 솔루션이 발견되지 않아 그냥 윈도우 함수를 이용해서 직전 좌표와 현재 좌표의 거리, 시간차를 계산하고 속도와 거리를 기준으로 필터링하기로 했다. 성능이 걱정되었다.

## 5. 구현

다음 조건을 만족하며 필터링을 수행해야 했다.

```text
1. 오류 좌표의 갯수는 여러개이며, 특정 지점에 모이는 경향이 있다.
2. 좌표 간의 거리와 시간차를 이용해 현실적인 좌표인지 검증 후, 아니라면 필터링한다.
3. 원본 데이터에 좌표기록기기에 전원이 공급되지 않은 시간 동안에는 좌표데이터 기록이 없을 수 있다. (기록시간이 점프할 수 있다.)
```

조건을 만족하는 쿼리를 작성했다. 쿼리가 지저분하지만 일단 원하는대로 잘 동작하고, 약 33000개의 좌표 데이터 기준, 0.2초의 수행속도를 확인했다. 성능 걱정이 줄었다.

```sql

  select
      tb2.car_no,
      tb2.reg_ymd,
      ST_MakeLine(tb2.geom order by tb2.reg_ymd, tb2.reg_tm)
  from
  (
      select
          tb1.car_no,
          tb1.reg_ymd,
          tb1.reg_tm,
          tb1.geom,
          ((ST_Distance(tb1.geom, (LAG(tb1.geom) OVER (ORDER BY tb1.car_no, tb1.reg_ymd, tb1.reg_tm))))) as distance_beforepoint_meter,
          case
              when tb1.car_no = lag(tb1.car_no) OVER (ORDER BY tb1.car_no,  tb1.reg_ymd, tb1.reg_tm) and tb1.reg_ymd = lag(tb1.reg_ymd) OVER (ORDER BY tb1.car_no,  tb1.reg_ymd, tb1.reg_tm)
              then ((ST_Distance(tb1.geom, (LAG(tb1.geom) OVER (ORDER BY tb1.car_no  tb1.reg_ymd, tb1.reg_tm))))*60) / ((EXTRACT(EPOCH FROM (to_timestamp(tb1.reg_ymd||tb1.reg_tm, 'YYYYMMDDHH24MISS') - LAG(to_timestamp(tb1.reg_ymd||tb1.reg_tm, 'YYYYMMDDHH24MISS')) OVER (ORDER BY tb1.car_no,  tb1.reg_ymd, tb1.reg_tm)))/30)*1000)
              else 0
          end as real_spee
      from (
          select
              car_no,
              reg_ymd,
              reg_tm,
              geom,
              case
                  when car_no = lag(car_no) OVER (ORDER BY car_no,  reg_ymd, reg_tm) and reg_ymd = lag(reg_ymd) OVER (ORDER BY car_no,  reg_ymd, reg_tm)
                  then ((ST_Distance(geom, (LAG(geom) OVER (ORDER BY car_no, reg_ymd, reg_tm))))*60) / ((EXTRACT(EPOCH FROM (to_timestamp(reg_ymd||reg_tm, 'YYYYMMDDHH24MISS') - LAG(to_timestamp(reg_ymd||reg_tm, 'YYYYMMDDHH24MISS')) OVER (ORDER BY car_no, reg_ymd, reg_tm)))/30)*1000)
              else 0
          end as real_spee
          FROM point_table
          ) tb1
      where 1=1
      and 4 <= tb1.real_spee -- 실제 평균속도가 5보다 크거나 같은 녀석들만. (오류좌표 제자리 기록된거 필터링)
  ) tb2
  where 1=1
  and tb2.real_spee < 300
  and tb2.distance_beforepoint_meter < 2500 -- 일반적인 차량이 30초간격에 갈 수 있는 최대거리 2500m(시속300km)
  group by
      tb2.car_no,
      tb2.reg_ymd;
```

## 6. 특이사항

위 쿼리를 group by 없이 point 데이터로 Select 할 때는 약 10초의 시간이 소요되었다.<br>배치작업을 통해 데이터를 테이블에 저장하고, 사용자는 만들어진 linestring을 WMS를 통해서 조회하는 구조였기에 point 데이터의 select 성능은 중요하지 않았으나, group by로 linestring을 만들 때와 그냥 point만 조회할 때의 성능 차이가 50배 가량 난다는 것이 의아했다.<br>"group by를 하면 추가 연산이 필요해서 성능이 더 안좋아져야 하는 것 아닌가?" 하는 생각이 들었다.
<br>
<br>
그래서 조사해 본 결과 상황에 따라 group by를 했을 때 성능이 향상될 수 있다고 한다.<br>
"group by를 사용했을 때 성능이 무조건 줄어든다"는 사람들의 편견이었다..<br>
데이터베이스가 group by 때문에 일을 덜 하게 되는 구조가 발생하기 때문이라고 한다.<br>

```text
※ group by 를 이용하게 되면, 출력해야 할 row가 적어져 옵티마이저가 사용할 수 있는 실행 전략의 폭이 넓어진다.

1. Postgresql에서 group by를 사용하면 옵티마이저가 내부적으로 HashAggregate를 사용해서 메모리 사용량을 늘리고, 속도는 상당히 빨라진다.
2. 일반 시퀀스 스캔 방식에서 parallel 시퀀스 스캔 방식으로 변경된다. 즉, CPU 1코어에서 순차 읽기를 하던 방식에서 다중 코어를 사용하는 방식으로 변경하여 테이블 접근을 가속화한다.
3. Gather Merge 방식의 정렬을 사용할 수 있다. parallel 시퀀스 스캔과 마찬가지로 정렬 작업을 멀티 코어 기반 병렬로 처리한다.

```
