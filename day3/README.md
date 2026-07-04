### Day 3

# P1 수집(collect)

```text
너는 Python 데이터 수집 멘토야. 공공데이터포털의 교통사고 데이터를
수집하는 collect_accidents(region, year)를 작성해줘. 우선순위:
① 공공데이터포털 오픈API(endpoint=http://apis.data.go.kr/B552061/frequentzoneLg/getRestFrequentzoneLg)가 있으면 API를, 없으면 다운로드한 CSV를 읽는 경로도 함께 지원.
각 레코드 필드: sido, sigungu, spot_name, address, lat, lon, accident_count, death_count, 
              injury_count, accident_type, road_type, occurred_at.
제약: 표준 파라미터(serviceKey·page/pageNo·perPage/numOfRows)·페이지네이션, 인증키는 .env,
      요청 간 0.5~1초 슬립·timeout 10초, 한 페이지 실패해도 전체는 계속(try/except).
      좌표가 없는 소스는 address만 채워 반환(다음 단계에서 지오코딩).
```

# P2 저장(save)

```text
수집 결과(dict 리스트/DataFrame)를 저장하는 save_data(rows, path)를 작성해줘.
제약: output 폴더 없으면 생성, 인코딩 utf-8-sig(엑셀 한글·좌표 안 깨짐), CSV와 parquet 둘 다
지원(옵션), 저장 후 행 수 출력. 원본은 data/raw/, 정제본은 data/clean/으로 분리 저장.
```
```
코드를 실행하여 데이터를 수집해줘.
```

# P3 정제+지오코딩(clean)

```
수집 DataFrame을 정리하는 clean(df)를 작성해줘. 단계(각 단계 주석):
① 좌표(lat,lon) 기준 중복 제거, 좌표 없으면 (sido,spot_name) 기준
② 시도·시군구명 표준화(normalize_region, 매핑표 dict로 분리)
③ 좌표 결측 행은 geocode(address)로 보정(공개 지오코딩 API, 캐시로 중복 호출↓, 실패 시 None),
   그래도 없으면 제거
④ accident_count·death_count·injury_count 숫자형 변환(이상치 로그), 한국 좌표범위
   (위도 33~39, 경도 124~132) 밖은 플래그
⑤ 사고건수 내림차순 정렬. 처리 전후 shape·결측률을 요약 출력.
```

# P4 집계(지역·격자)

```
정제 데이터를 두 방식으로 집계하는 코드를 작성해줘.
① 지역 단위: 시군구별 사고건수·사망·부상·평균 심각도 합계
② 격자 단위: 위경도를 약 500m 격자로 반올림(그리드ID)해 격자별 사고 집계
각 집계 결과를 DataFrame으로 반환하고, 상위 20개를 출력해줘.
```

# P5 위험 스코어링 + 등급화 (핵심 분석)

```
지역/격자별 교통사고 위험 점수(0~100)를 만드는 calc_risk_score(df)를 작성해줘. 지표:
① 사고 밀도(면적 또는 도로연장 대비 사고건수) ② 치명도((사망+중상)/전체)
③ 사상자 강도(사고당 평균 사상자) ④ 취약군 비율(보행자·고령·어린이 관련 비율)
제약: 각 지표를 min-max로 0~1 정규화 → 가중치(상수 dict, 합=1)로 가중합 → 0~100 환산 →
5분위로 위험 등급(A안전~E위험) 부여. 각 지점에 '어떤 지표가 점수를 높였는지' 근거 컬럼 추가.
컬럼이 없으면 대체 지표를 제안해줘.
```

# P6 위험지도 시각화 (folium)

```
folium으로 교통사고 위험지도(HTML)를 만드는 build_risk_map(df)를 작성해줘. 레이어:
① HeatMap — 사고 지점 좌표·가중치(위험 점수)로 열지도
② 위험 상위 마커 — 클릭 시 지역명·점수·주요 지표 팝업
③ 시군구 단계구분도(choropleth) — 위험 등급별 색(녹→적)
제약: 중심좌표는 데이터 평균, 적정 zoom, LayerControl로 레이어 토글,
output/risk_map.html 저장. GeoJSON 경계가 필요하면 불러오는 방법도 주석으로 안내.
```

# P7 보조 차트 + 통합 파이프라인

```
(1) 상위 15개 위험지역 막대, (2) 사고밀도 vs 치명도 산점도(위험 상위 색 강조) 2종을
matplotlib으로 그려 output/charts에 저장(한글 폰트·axes.unicode_minus=False 포함).
그리고 collect→save→clean→집계→calc_risk_score→build_risk_map 전체를 run(region, year)
하나로 묶고, 각 단계 소요시간·행 수를 로그로 남겨줘. 예외 시 어느 단계 실패인지 표시.
```
