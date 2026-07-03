### Day2 

#### M1. 크롤링 개념

- HTML 구조 탐색
- robots.txt와 요청 매너 이해

### P: 시나리오

```text
매주 월요일 임원 보고 자료에 ‘지난 주 보험 시장 핫이슈’ 가 들어간다. 
매번 검색 사이트에서 직접 ‘자동차보험’, ‘실손보험’ 을 검색해 5페이지씩 보는 게 일이다. 
내가 검색하지 않고도 매일 아침 ‘오늘의 보험 뉴스’ 가 CSV 로 도착하게 만들 수 있을까?
```

### P0 · 골격 만들기 (시작점)

```text
너는 Python 크롤링 튜터야. ‘보험 뉴스’를 수집하는 크롤러의 최소 골격을 작성해줘.
search_insurance_news(keyword="자동차보험", pages=1) → 네이버/다음 뉴스 검색결과에서
기사 제목·링크·언론사·날짜를 dict 리스트로 반환. 지금은 requests + BeautifulSoup으로
1페이지만, 구조를 단순·명확하게. 이후 단계에서 하나씩 개선할 거야.
```

### P1 · 체크리스트①: Network 탭 → API 우선

```text
이전 코드를 개선해줘. Selenium을 꺼내기 전에, 뉴스 검색에 숨은 API(XHR/JSON)가 있는지
먼저 확인하는 구조로 바꿔줘.
- try_news_api(keyword) 헬퍼: 검색 결과를 JSON으로 주는 엔드포인트를 requests로 호출해
  제목·링크·언론사·날짜를 파싱(있으면 반환)
- 없거나 실패하면 그때만 HTML 파싱/Selenium으로 폴백
‘F12 Network 탭에서 XHR을 먼저 보는 이유’를 주석으로 설명. 기존 반환 형식 유지.
```

### P2 · 체크리스트②③④⑤
 - time.sleep → 조건 대기
 - 기다릴 '기준 요소' 지정
 - 타임아웃·예외 설계
 - Headless로 가볍게 + 예의

```text
동적으로 로딩되는 뉴스 목록(스크롤·더보기)을 위해 Selenium 경로에서 time.sleep 대신
WebDriverWait + expected_conditions로 바꿔줘. 기사 목록이 실제로 나타날 때까지 ‘조건’으로
기다리게 하고, sleep보다 나은 이유를 주석으로. 기존 시그니처는 유지.

Selenium 수집 함수에 wait_selector 인자를 추가해줘. 보험 뉴스 목록의 로드 완료를 알리는
특정 요소(예: 기사 카드 컨테이너 CSS)를 기준으로 대기하게 하고, 이 요소가 왜 완료 신호로
적절한지 주석으로 설명. wait_selector가 None이면 기본 컨테이너를 쓰도록 처리.

뉴스 크롤러에 예외 설계를 추가해줘.
- TimeoutException: 목록이 제한시간 내 안 뜨면 로그 후 재시도(최대 3회·지수 백오프)
- StaleElementReferenceException: 목록이 갱신(무한스크롤)될 때 안전하게 재조회
- 한 페이지 실패해도 전체 수집은 계속, 최종 실패 시 해당 페이지는 건너뜀
- HTTP 429/5xx도 재시도. 각 예외를 왜 잡는지 주석으로.

Selenium에 headless 옵션(headless=True 기본)과 안정화 옵션(window-size, user-agent)을 추가해줘.
그리고 요청 예의를 위해 페이지 사이 0.8~1.5초 무작위 슬립을 넣어줘. 디버깅 시엔
headless=False로 화면을 보라는 팁을 주석으로. robots.txt·이용약관 확인 문구도 상단에.
```

### P6 · 체크리스트⑥: 리다이렉트/변화 감지 (더보기·페이지 이동)

```
‘더보기’ 클릭이나 페이지 이동으로 목록이 바뀌는 걸 감지하는 로직을 추가해줘.
next_page(또는 more 버튼) 후 (1) URL 변경 (2) 기존 첫 기사 요소의 소멸(staleness)
(3) 새 기사 등장 중 하나를 신호로 기다렸다가 다음 페이지를 수집하게 해줘(expected_conditions).
중복 기사(link 기준)는 제거. pages만큼 반복.
```

---

#### M3. 데이터 정제

##### P1 · 학습용 ‘지저분한’ 보험 뉴스 샘플 데이터 생성

```
너는 Python 데이터 분석 튜터야. Pandas 정제를 연습할 수 있도록, 일부러 문제를 섞은
‘보험 뉴스’ 샘플 CSV를 만드는 make_sample_news(path="output/news.csv", n=200)를 작성해줘.
컬럼: title, url, press, date, keyword, views.
일부러 넣을 문제(각 몇 %씩):
  · 결측: press·date 일부 비움, keyword 일부 비움
  · 중복: 같은 url이 2~3번 등장(완전중복) + url 같고 views만 다른 '일부만 같은 행'
  · 이상치: views에 음수·비정상적 초대형 값 몇 개
  · 지저분한 표기: 날짜가 '2시간 전','3일 전','2026.05.30.','2026-05-30' 섞임,
    제목 앞뒤 공백·중복 공백, press에 '땡땡일보 '처럼 공백
보험사명(삼성화재·DB손해보험·현대해상·KB손해보험 등)이 제목에 자연스럽게 들어가게 해줘.
seed 고정(재현 가능), utf-8-sig 저장. 저장 후 shape와 결측 개수를 출력.
```

##### P2 · Pandas 기본기 7가지

```
방금 만든 output/news.csv로 Pandas 핵심 7가지를 실습하는 코드를 작성하고, 각 줄이 무엇을
하는지 주석으로 설명해줘.
① read_csv로 읽기 ② head() 상위 5행 ③ shape·columns·dtypes 메타정보
④ describe(include="all") 기술통계 ⑤ df["keyword"].value_counts() 빈도 집계
⑥ drop_duplicates(subset=["url"]) 중복 제거 ⑦ dropna(subset=["title"]) 결측 행 제거
마지막에 to_csv("output/news_clean.csv", index=False)로 저장.
각 단계 실행 결과(예: value_counts 상위 5개)도 보여줘.
```

##### P3 · 결측 (Missing) — 제거 vs 대치

```text
news.csv의 결측을 다루는 코드를 작성해줘.
· isna()/notna()로 컬럼별 결측 비율(%) 출력
· 규칙: press 결측 → '미상'으로 fillna, date 결측 → 제거, views 결측 → 중앙값 fillna
· 결측 비율이 30% 이상인 컬럼은 '제거 고려' 대상으로 경고 출력
각 선택(제거 vs 대치)의 이유를 한 줄 주석으로 달아줘.
```

##### P4 · 중복 (Duplicate) — 완전 vs 일부

```
news.csv의 중복을 다루는 코드를 작성해줘.
· 완전 같은 행: df.duplicated()로 개수 확인 후 제거
· 뉴스 dedup: url 기준 중복 제거(같은 기사)
· '일부만 같은 행'(url 같고 views 다름): date 최신 1건만 남기기(sort 후 drop_duplicates)
제거 전후 행 수를 출력하고, '완전 같은 행 vs 일부만 같은 행'의 차이를 주석으로 설명.
```

##### P5 · 이상치 (Outlier) — IQR·Z-score·도메인 룰

```
news.csv의 views 이상치를 점검하는 코드를 작성해줘.
· describe()와 boxplot(matplotlib, 한글 폰트·axes.unicode_minus=False)로 시각 점검
· IQR 룰(Q1-1.5·IQR ~ Q3+1.5·IQR) 밖과 Z-score 3 이상을 각각 이상치로 표시
· 도메인 룰: views 음수는 오류로 간주해 제거, 초대형 값은 상한(capping)으로 보정
이상치를 무조건 삭제하지 말고 우선 is_outlier 플래그로 표시하는 이유도 주석으로.
```

##### P6 · 한국어 날짜 표준화 + 통합 파이프라인

```
지저분한 date를 datetime으로 통일하는 parse_korean_date(s)를 만들고('2시간 전','3일 전',
'2026.05.30.','2026-05-30' 등 여러 패턴 순서대로 시도, 실패 시 NaT), 제목 공백 정리까지 포함해
clean(df) 하나로 묶어줘. 단계: 공백정리 → 날짜표준화 → 결측처리 → 중복제거 → 이상치 플래그.
처리 전후 shape·결측률을 비교 출력하고 output/news_clean.csv로 저장.
```

##### P7 

```
방금 코드에 대해 정상·경계·예외 케이스 테스트 3개를 만들어 통과까지 보여줘.
(예: 결측만 있는 열, 완전중복+부분중복 혼재, views 음수·초대형) 실패하면 원인 설명 후 고쳐줘.
```
