# 🚀 조회 성능 개선하기

## A. 쿼리 연습

### * 실습환경 세팅

```sh
$ docker run -d -p 23306:3306 brainbackdoor/data-tuning:0.0.1
```
- [workbench](https://www.mysql.com/products/workbench/)를 설치한 후 localhost:23306 (ID : user, PW : password) 로 접속합니다.

<div style="line-height:1em"><br style="clear:both" ></div>


> 활동중인(Active) 부서의 현재 부서관리자 중 연봉 상위 5위안에 드는 사람들이 최근에 각 지역별로 언제 퇴실했는지 조회해보세요.
(사원번호, 이름, 연봉, 직급명, 지역, 입출입구분, 입출입시간)


### 1. 쿼리 작성만으로 1s 이하로 반환한다.

#### sql문
```sql
select  TOP5MANAGER_NAME.사원번호, 이름, 연봉, 직급명, 입출입시간, 지역, 입출입구분 from
  (select TOP5MANAGER.사원번호, 이름, 연봉, 직급명 from
    (select TOP5.사원번호, 연봉, 직급명 from 
      (SELECT 활동부서관리자.사원번호, 연봉 from 
        (SELECT 사원번호 
          from (SELECT * FROM 부서 where 비고="ACTIVE") as 활동부서,
            (select * from 부서관리자 where 종료일자 > "2021-10-11") as 활동관리자 
            where 활동부서.부서번호=활동관리자.부서번호) as 활동부서관리자, 
          급여 where 활동부서관리자.사원번호=급여.사원번호 and 급여.종료일자 > "2021-10-11" 
        order by 급여.연봉 desc limit 5) as TOP5,
      직급 where TOP5.사원번호=직급.사원번호 and 직급.종료일자 > "2021-10-11") as TOP5MANAGER,
    사원 where TOP5MANAGER.사원번호=사원.사원번호) as TOP5MANAGER_NAME,
  사원출입기록 where TOP5MANAGER_NAME.사원번호 = 사원출입기록.사원번호 and 사원출입기록.입출입구분 ="O" order by TOP5MANAGER_NAME.연봉 desc, 사원출입기록.지역
```

#### 결과

![Screenshot from 2021-10-11 15-31-45](https://user-images.githubusercontent.com/49307266/136743140-c9bd7df3-15ff-4836-9866-dd86815e15ce.png)


<div style="line-height:1em"><br style="clear:both" ></div>
<div style="line-height:1em"><br style="clear:both" ></div>


## B. 인덱스 설계

### * 실습환경 세팅

```sh
$ docker run -d -p 13306:3306 brainbackdoor/data-subway:0.0.2
```
- [workbench](https://www.mysql.com/products/workbench/)를 설치한 후 localhost:13306 (ID : root, PW : masterpw) 로 접속합니다.

<div style="line-height:1em"><br style="clear:both" ></div>

### * 요구사항

- [ ] 주어진 데이터셋을 활용하여 아래 조회 결과를 100ms 이하로 반환

    - [ ] [Coding as a  Hobby](https://insights.stackoverflow.com/survey/2018#developer-profile-_-coding-as-a-hobby) 와 같은 결과를 반환하세요.

    - [ ] 각 프로그래머별로 해당하는 병원 이름을 반환하세요.  (covid.id, hospital.name)

    - [ ] 프로그래밍이 취미인 학생 혹은 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요. (covid.id, hospital.name, user.Hobby, user.DevType, user.YearsCoding)

    - [ ] 서울대병원에 다닌 20대 India 환자들을 병원에 머문 기간별로 집계하세요. (covid.Stay)

    - [ ] 서울대병원에 다닌 30대 환자들을 운동 횟수별로 집계하세요. (user.Exercise)

<div style="line-height:1em"><br style="clear:both" ></div>
<div style="line-height:1em"><br style="clear:both" ></div>

## C. 프로젝트 요구사항

### a. 페이징 쿼리를 적용 

### b. Replication 적용 
