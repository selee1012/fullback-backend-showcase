# GlobalStep — 외국인 대상 다국어 산책로 가이드

서울시 산책로 공공데이터를 지도 위 경로로 바꾸고, 외국인이 회원가입이나 지명 입력 없이 영어·중국어로 산책로를 찾을 수 있게 만든 서비스입니다.
이 저장소는 그중 **백엔드 설계와 의사결정 기록**을 정리한 쇼케이스입니다. 원본 코드는 비공개 저장소에 있습니다.

<p>
  <img src="https://img.shields.io/badge/Java-25-007396?style=flat-square&logo=openjdk&logoColor=white" alt="Java 25">
  <img src="https://img.shields.io/badge/Spring%20Boot-3.5.5-6DB33F?style=flat-square&logo=springboot&logoColor=white" alt="Spring Boot 3.5.5">
  <img src="https://img.shields.io/badge/MySQL-Spatial-4479A1?style=flat-square&logo=mysql&logoColor=white" alt="MySQL Spatial">
  <img src="https://img.shields.io/badge/AWS-EC2%20%7C%20RDS%20%7C%20S3-FF9900?style=flat-square&logo=amazonwebservices&logoColor=white" alt="AWS">
  <img src="https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white" alt="GitHub Actions">
</p>

| 항목 | 내용 |
| --- | --- |
| 구분 | 명지대학교 융합소프트웨어학부 설계 프로젝트 |
| 기간 | 2026년 상반기, 개발 8주 (최종보고서 v1.0 · 2026-06-05) |
| 팀 | Full-Back 4인 — BE 1 / FE 1 / Infra·Test 1 / ETL 1 |
| 내 역할 | 백엔드 전반(Spring Boot API), LLM 비속어 필터링 |
| 서비스 | https://global-step-silk.vercel.app/ (현재 중지) |

## 왜 만들었나

재한 외국인의 43.4%가 언어 장벽 때문에 여가 생활에 불편을 겪지만, 서울시 산책로 공공데이터는 한국어 텍스트와 행정 좌표로만 제공됩니다.
그래서 좌표를 TM(EPSG:5181)에서 WGS84로 바꿔 지도 경로로 만들고, 가입 없이 태그만 골라 영어·중국어로 탐색하게 했습니다.

## 내가 맡은 일

- Course · Place · Note · Review · Guest · Map 6개 도메인의 REST API와 JPA 엔티티, 다국어 스키마 설계
- MySQL Spatial 공간 쿼리(반경 검색·거리 정렬), 공통 응답·예외 규격, Swagger 문서화
- Upstage Solar 기반 다국어 비속어 필터 — 호출 예산·동시성 가드와 규칙 기반 폴백의 2단 구조

부하 테스트(k6)는 팀원이 맡았고, 아래 결과 표에는 지표만 인용했습니다.

## 주요 설계 판단

**1. 가입 없는 좋아요가 인기 순위를 왜곡하는 문제**
회원가입을 없애자, 좋아요를 단순 카운트로 쌓으면 같은 방문자의 반복 클릭으로 순위가 흔들릴 수 있었습니다.
회원 인증 대신 Guest UUID(BINARY(16) unique)를 발급하고, 표시용 카운트와 방문자별 좋아요 이력을 분리해 이력 유무로 토글하게 했습니다.
다른 UUID 재발급과 동시 경합까지 막는 수단은 아니어서, 동시 요청의 UNIQUE 제약을 후속 보강 과제로 남겼습니다. → [05. 백엔드 설계](docs/05-backend-design.md)

**2. 조회마다 번역하면 생기는 오역·지연·비용**
외국인은 동·구 이름을 입력하기 어렵고, 관광 콘텐츠를 매번 자동 번역하면 고유명사 오역과 외부 API 지연·비용이 생깁니다.
대상 언어가 영어·중국어로 정해진 8주 프로젝트라, 런타임 번역 대신 영·중 컬럼과 사전 큐레이션을 택했습니다. 검색은 숲길·야경·랜드마크 같은 태그 선택으로 바꾸고, 필터는 영문 태그 enum 하나로 통일했습니다.
그 결과 언어를 바꿔도 필터 쿼리는 같고, 탐색 요청에서 번역 API를 호출하지 않습니다. 언어가 늘면 번역 테이블로 옮길지 다시 검토하도록 조건을 기록해 두었습니다. → [05. 백엔드 설계](docs/05-backend-design.md)

**3. 외부 API 장애가 서버 응답으로 번지는 문제**
서버가 지도 API를 대신 호출하면 외부 장애와 지연이 그대로 백엔드 응답에 실립니다.
경로 계산과 지도 렌더링은 프론트엔드로 옮기고, 서버는 좌표와 코스 데이터만 내려주게 했습니다. 좌표 순서(경도·위도 / 위도·경도) 변환은 한 서비스에 모았고, 거리는 MySQL `ST_Distance_Sphere`로 계산합니다.
LLM 비속어 필터도 같은 원칙으로, 외부 호출이 실패하면 규칙 기반 필터로 넘어가 커뮤니티 기능이 멈추지 않게 했습니다. → [03. 아키텍처](docs/03-architecture.md) · [06. LLM 비속어 필터링](docs/06-llm-moderation.md)

## 결과

| 지표 | 목표 | 결과 |
| --- | --- | --- |
| 다국어 정보 만족도 (UAT) | 5점 만점 4.0 이상 | 4.2점 |
| 부하 환경 가용성 (k6, 팀원 측정) | 99% 이상 | 2,166 요청 중 오류 0건 |
| 평균 응답 시간 | 1초 미만 | 16.9ms (피크에도 218ms 이내) |
| GIS 경로 정합성 | 위치 오차 10m 이내 | 전수 확인 |
| 데이터 적재 | 코스·POI 각 100건 이상 | 서울 전역 GIS 트랙 100+ / 정제 POI 100+ |

## 시스템 구조

<img src="assets/aws-architecture.png" alt="AWS 인프라 구성도" width="360">

```
[Next.js / Vercel]  ──HTTPS──▶  [Nginx (Public Subnet)]
                                      │
                                      ▼
                          [Spring Boot on EC2 (Private Subnet)]
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
        [MySQL Spatial / RDS]   [Amazon S3]     [Upstage Solar API]
         공간 인덱스·다국어      이미지 보관        비속어 필터
```

## 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| Language / Runtime | Java 25 |
| Framework | Spring Boot 3.5.5, Spring Data JPA, Spring Validation |
| Database | MySQL 8.x + Spatial, Hibernate Spatial 6.6, JTS 1.20 |
| API 문서 | springdoc-openapi 2.7 (Swagger UI) |
| 외부 연동 | Google Maps(Distance Matrix · Street View · 딥링크), Kakao Map 딥링크, Upstage Solar |
| Infra | AWS EC2 · RDS · S3, Nginx, Docker (linux/arm64) |
| CI/CD | GitHub Actions → Docker Hub → EC2 |
| Test | JUnit 5, Mockito, H2(통합 테스트), k6 |
| Frontend (팀원) | Next.js, React, Vercel |

## 문서

| 문서 | 내용 |
| --- | --- |
| [01. 프로젝트 개요](docs/01-overview.md) | 문제 정의, 솔루션과 차별점, 핵심 기능 |
| [02. 설계 최종보고서](docs/02-final-report.md) | 요약, 제한조건, 설계 목적, 서비스 흐름, 설계 평가 |
| [03. 아키텍처 & 인프라](docs/03-architecture.md) | 3-Tier 구성, ERD, AWS VPC, CI/CD |
| [04. API 명세](docs/04-api-spec.md) | 6개 도메인 REST API, 공통 응답 규격, 에러 코드 |
| [05. 백엔드 설계 상세](docs/05-backend-design.md) | 패키지 구조, 공간 쿼리, N+1 대응, 다국어 스키마, 트랜잭션 |
| [06. LLM 비속어 필터링](docs/06-llm-moderation.md) | Solar 모더레이션, 예산·동시성 가드, 규칙 기반 폴백 |
| [07. 성능 · 부하 테스트](docs/07-performance.md) | k6 시나리오, 측정 결과, 병목 분석 |
| [08. 의사결정 & 회고](docs/08-retrospective.md) | 기술 선택 근거, 개발 중 방향 전환, 한계와 향후 계획 |

## 서비스 화면

| 태그 기반 탐색 | 코스 상세 + 주변 정보 | 다국어 비속어 필터 |
| --- | --- | --- |
| <img src="assets/screen-tag-filter.png" alt="태그 필터"> | <img src="assets/screen-course-detail.png" alt="코스 상세"> | <img src="assets/screen-profanity-filter.png" alt="비속어 필터"> |

## 팀

| 이름 | 역할 |
| --- | --- |
| 이성은 | 백엔드 전반(Spring Boot API), LLM 비속어 필터링 |
| 신지훈 | 프론트엔드, UI/UX 설계 |
| 이성우 | 인프라, 성능 테스트 |
| 성예영 | 데이터 ETL |

지도교수: 명지대학교 융합소프트웨어학부 김상균 교수
