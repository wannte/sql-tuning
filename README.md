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

#### 결과 (0.357s)

![Screenshot from 2021-10-11 15-31-45](https://user-images.githubusercontent.com/49307266/136743140-c9bd7df3-15ff-4836-9866-dd86815e15ce.png)

### 2. 인덱스 설정을 추가하여 50 ms 이하로 반환한다.

![before](https://user-images.githubusercontent.com/49307266/136744678-c637691c-2699-4d95-ae54-59f3afb08a3f.png)
분석을 해보면, 마지막 사원출입기록의 Table Full Scan에서 많은 비용이 발생하는 것을 확인할 수 있다. 사원번호에 index를 추가해주었을 때, 대폭 감소할 수 있었다.

![after](https://user-images.githubusercontent.com/49307266/136745406-5b0c3138-8b77-401a-8075-97fdb8c2b7fc.png)
![Screenshot from 2021-10-11 15-54-51](https://user-images.githubusercontent.com/49307266/136745556-fb375a4d-0665-4456-adbb-7877387238b1.png)

#### 결과 (0.0012s)
![Screenshot from 2021-10-11 15-52-10](https://user-images.githubusercontent.com/49307266/136745230-09025c0d-f756-405a-8ec6-9d2dfdc42f62.png)

<div style="line-height:1em"><br style="clear:both" ></div>
<div style="line-height:1em"><br style="clear:both" ></div>


## B. 인덱스 설계

### * 실습환경 세팅

```sh
$ docker run -d -p 13306:3306 brainbackdoor/data-subway:0.0.2
```
- [workbench](https://www.mysql.com/products/workbench/)를 설치한 후 localhost:13306 (ID : root, PW : masterpw) 로 접속합니다.

<div style="line-height:1em"><br style="clear:both" ></div>

### * 요구사항!

* 테이블 정보
[Screenshot from 2021-10-11 16-52-03](https://user-images.githubusercontent.com/49307266/136753849-a87b4542-d39b-4e1a-a5af-2b67112f4f8b.png)


> 주어진 데이터셋을 활용하여 아래 조회 결과를 100ms 이하로 반환

#### 1. [Coding as a  Hobby](https://insights.stackoverflow.com/survey/2018#developer-profile-_-coding-as-a-hobby) 와 같은 결과를 반환하세요.
![Screenshot from 2021-10-11 16-04-01](https://user-images.githubusercontent.com/49307266/136746732-2ce139b9-7f13-41a7-9c31-6fb8d63f50cf.png)
![Screenshot from 2021-10-11 16-04-49](https://user-images.githubusercontent.com/49307266/136746737-772cc711-04f2-4b32-8b6f-ae0fde635493.png)
index 사용(0.215s -> 0.067s)

#### 2. 각 프로그래머별로 해당하는 병원 이름을 반환하세요.  (covid.id, hospital.name)
```sql
SELECT covid.id, hospital.name 
FROM covid, hospital 
where covid.hospital_id = hospital.id;
```
    
1. index 없이
![Screenshot from 2021-10-11 16-54-41](https://user-images.githubusercontent.com/49307266/136753399-19dc28a2-0824-4930-a8b4-59a2893d872d.png)
2. index 도입 (covid_id: unique index, covid_hospital_id: index, hospital_id: unique_index)
covid 테이블의 Row가 많기에, covid.hospital_id의 Index와 hospital 전체를 순회하는 것을 확인할 수 있다.
![Screenshot from 2021-10-11 16-56-43](https://user-images.githubusercontent.com/49307266/136753643-56de5ca8-4633-47ba-aa7e-f29b998b7f38.png)
3. straight_join & index 도입 (covid_id: unique index, covid_hospital_id: index, hospital_id: unique_index)
![Screenshot from 2021-10-11 16-56-18](https://user-images.githubusercontent.com/49307266/136753722-25e52782-f60e-4146-9601-2276deb132ec.png)
straight_join을 사용해서 강제로 covid를 드라이빙 테이블로. hospial은 id로 인덱스 접근.
    
### 3. 프로그래밍이 취미인 학생 혹은 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요. (covid.id, hospital.name, user.Hobby, user.DevType, user.YearsCoding)

![Screenshot from 2021-10-11 17-17-04](https://user-images.githubusercontent.com/49307266/136756275-9284a57d-cf25-4986-b754-0fdbaea4f970.png)
index 설정을 해주었지만, 전체를 다 조회하는 것을 확인할 수 있다.
```sql
SELECT 
    COVID_NEWBIE.id,
    hospital.name,
    COVID_NEWBIE.hobby,
    COVID_NEWBIE.dev_type,
    COVID_NEWBIE.years_coding
FROM
    (SELECT 
        NEWBIE.id,
            covid.hospital_id,
            NEWBIE.hobby,
            NEWBIE.dev_type,
            NEWBIE.years_coding
    FROM
        (SELECT 
        id, hobby, dev_type, years_coding
    FROM
        programmer
    WHERE
        hobby = 'yes'
            OR years_coding = '0-2 years') AS NEWBIE, covid
    WHERE
        NEWBIE.id = covid.id) AS COVID_NEWBIE,
    hospital
WHERE
    COVID_NEWBIE.hospital_id = hospital.id
```

![Screenshot from 2021-10-11 17-48-37](https://user-images.githubusercontent.com/49307266/136761151-804bb82f-6dfe-4bc5-94d9-93c688b47b55.png)


    - [ ] 서울대병원에 다닌 20대 India 환자들을 병원에 머문 기간별로 집계하세요. (covid.Stay)

    - [ ] 서울대병원에 다닌 30대 환자들을 운동 횟수별로 집계하세요. (user.Exercise)

<div style="line-height:1em"><br style="clear:both" ></div>
<div style="line-height:1em"><br style="clear:both" ></div>

## C. 프로젝트 요구사항

### a. 페이징 쿼리를 적용 

### b. Replication 적용 
