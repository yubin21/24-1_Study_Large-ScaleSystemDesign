# 9장 웹 크롤러 설계
## 크롤링 사용처
- 검색 엔진 인덱싱: 크롤러의 가장 보편적인 용례다. 크롤러는 웹 페이지를 모아 검색 엔진을 위한 로컬 인덱스를 만든다. 예를 들어 Googlebot은 구글 검색 엔진이 사용하는 웹 크롤러다.
- 웹 아카이빙: 나중에 사용할 목적으로 장기보관하기 위해 웹에서 정보를 모으는 절차를 말한다.
- 웹 마이닝: 웹 마이닝을 통해 인터넷에서 유용한 지식을 도출해 낼 수 있다.
- 웹 모니터링: 크롤러를 사용하면 인터넷에서 저작권이나 상표권이 침해되는 사례를 모니터링할 수 있다.

## 1단계 문제 이해 및 설계 범위 확정
### 크롤링 기본 알고리즘
- URL집합이 주어지면, 해당 URL이 가리키는 모든 웹페이지 다운로드
- 다운 받은 웹페이지 URL추출
- 추출된 URL들을 다운로드할 URL 목록에 추가하고 위의 과정을 반복
### 주의 사항
- 규모 확장성: 병행성을 활용하면 보다 효과적인 웹 크롤링 가능
- 안정성: 비 정상적인 입력이나 환경에 잘 대처해야 한다.
- 예절: 짧은 시간 동안 너무 많은 요청을 보내서는 안된다.
- 확장성: 새로운 형태의 콘텐츠를 지원하기가 쉬워야 한다.
### 개략적 규모 추정
- 매달 10억 개의 웹 페이지를 다운로드 한다.
- QPS = 10억 / 30일 / 24시간 / 3600초 = 대략 400 page / sec
- 최대 (Peak) QPS = 2 X QPS = 800 page/sec
- 웹 페이지의 평균 크기는 500k라고 가정
- 10억 페이지 x 500k = 500 TB /month.
- 1개월치 데이터를 보관하는 데는 500 TB, 5년간 보관한다고 가정하면 결국 500TB X 12개월 X 5년 = 30PB의 저장용량이 필요하다.

## 2단계 개략적 설계안 제시 및 동의 구하기
![](https://user-images.githubusercontent.com/42582516/186928449-612b2dfe-dc8b-4fbe-9dda-11d94d38509b.png)
- 시작 URL집합: 크롤링을 시작하는 출발점, 각각의 주제에 맞게 URL을 선택하는게 좋다
- 미수집 URL 저장소: 다운로드할 URL을 저장관리하는 컴포넌트, FIFO 큐라고 생각하면 된다.
- HTML 다운로더: 인터넷에서 웹페이지를 다운하는 컴포넌트
- 도메인 이름 변환기: URL에 대응되는 IP주소를 확인
- 콘텐츠 파서: 웹페이지를 다운하기 위해 파싱과 검증 절차를 거친다.
- 중복콘텐츠?: 웹페이지의 해시 값을 비교해서 중복 컨텐츠를 구분
- 콘텐츠 저장소: 대부분의 콘텐츠는 디스크에 저장, 인기 있는 콘텐츠는 메모리에 저장
- URL추출기: HTML 페이지를 파싱하여 링크들을 골라내는 역할
- URL 필터: 적절하지 않은 URL등을 크롤링 대상에서 배제하는 역할
- 이미 방문한 URL?: 블룸 필터나 해시 테이블을 사용해서 같은 URL을 여러번 처리하지 않게 한다.
- URL 저장소 : 이미 방문한 URL을 보관하는 저장소 
## 3단계 상세 설계
### DFS vs BFS
- BFS를 사용하는 것이 더 좋은 방법
- DFS는 병렬 처리로 인한 과부화 문제가 발생할 수 있다.
- 표준 BFS는 우선순위를 두지 않지만 웹 크롤링 시에는 더 좋은 데이터를 우선순위로 두는것이 좋다. 
### 미수집 URL 저장소
- 미수집 URL저장소를 활용하면 예의를 갖춘 크롤러, URL 사이의 우선순위와 신선도를 구별하는 크롤러 구현 가능
### 예의
- 수집 대상 서버로 짧은 시간 안에 너무 많은 요청을 보내지 말아야 한다.
- 동일 웹사이트에 대해 한번에 한 페이지만 요청해야한다. 
- 큐 라우터 : 같은 호스트에 속한 URL은 언제나 같은 큐로 가도록 보장하는 역할을 한다.
- 매핑 테이블 : 호스트 이름과 큐 사이의 관계를 보관하는 테이블
- FIFO 큐 : 같은 호스트에 속한 URl은 언제나 같은 큐에 보관된다.
큐 선택기 : 큐 선택기는 큐들을 순회하면서 큐에서 URl을 꺼내서 해당 큐에서 나온 URL을 다운로드하도록 지정된 작업 스레드에 전달하는 역할을 한다.
- 작업 스레드 : 작업 스레드는 전달된 URL을 다운로드하는 작업을 수행한다. 전달된 URL은 순차적으로 처리될 것이며, 작업들 사이에는 일정한 지연시간을 둘 수 있다.
### 신선도
- 데이터의 신선도를 유지하기 위해 주기적으로 페이지를 재수집 해야한다.
- 웹 페이지의 변경 이력 활용
- 우선순위를 활용해서 중요한 페이지를 더 자주 수집
### HTML 다운로더
- HTTP 프로토콜을 통해 웹페이지 다운받음
### Robots.txt
- 웹 사이트에 크롤링 해도 되는 페이지 목록이 들어가 있다.
- EX) Disallow: /creatorhub/*

### 성능 최적화
- 분산 크롤링: 크롤링 작업을 여러 서버에 분산하는 방법
- 도메인 이름 변환 결과 캐시: 도메인 이름과 IP주소 사이의 관계를 캐시해 보관해두고 크론 잡등을 돌려 주기적으로 갱신하면 성능을 효과적으로 높힐 수 있다.
- 지역성: 크롤링 서버가 크롤링 대상 서버와 지역적으로 가깝게 해두면 페이지 다운로드 시간을 줄일 수 있다.
- 짧은 타임아웃: 최대 대기 시간을 정해두고 응답하지 않을시 다음 페이지로 넘어간다.
### 안정성
- 안정 해시: 다운로드 서버 부하 분산할 때 적용 가능한 기술
- 크롤링 상태 및 수집 데이터 저장: 크롤링 상태와 수집된 데이터를 지속적 저장장치에 기록해두는게 바람직하다.
- 예외 처리: 예외가 발생해도 전체 시스템이 중단되지 않도록 한다.
- 데이터 검증: 시스템 오류를 방지하기 위해 중요 수단 가운데 하나
### 문제 있는 콘텐츠 감지 및 회피
- 중복 컨텐츠: 해시나 체크섬을 사용해 중복 콘텐츠를 보다 쉽게 탐지 가능
- 거미 덫: 크롤러를 무한 루프에 빠지게 설계한 웹 페이지, 자동으로 식별하기는 어렵기 때문에 수동으로 필터링 해야한다.
- 데이터 노이즈: 광고나 스팸 URL은 필요 없으므로 제외

## 4단계 마무리
### 추가로 논의해볼 점
- 서버 측 렌더링
- 원치 않으 페이지 필터링
- 데이터베이스 다중화 및 샤딩
- 수평적 규모 확장성
- 가용성, 일관성, 안전성
- 데이터 분석 솔루션
