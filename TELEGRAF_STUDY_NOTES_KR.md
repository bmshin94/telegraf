<!-- markdownlint-disable MD013 -->
# Telegraf 분석 및 활용 정리 (한국어)

> 이 문서는 Telegraf 저장소를 처음 접한 개발자를 위해, 저장소 내용을 직접
> 확인하며 분석한 내용을 정리한 학습/기획 노트입니다.

## 📎 관련 GitHub 주소

| 구분 | 주소 |
| --- | --- |
| 원본 저장소 (InfluxData 공식) | <https://github.com/influxdata/telegraf> |
| 이 저장소 (포크) | <https://github.com/bmshin94/telegraf> |
| 릴리스 | <https://github.com/influxdata/telegraf/releases> |
| 기여 가이드 | <https://github.com/influxdata/telegraf/blob/master/CONTRIBUTING.md> |
| 외부 플러그인 목록 | <https://github.com/influxdata/telegraf/blob/master/EXTERNAL_PLUGINS.md> |
| Input 플러그인 | <https://github.com/influxdata/telegraf/tree/master/plugins/inputs> |
| Output 플러그인 | <https://github.com/influxdata/telegraf/tree/master/plugins/outputs> |
| 공식 문서 | <https://docs.influxdata.com/telegraf/> |
| 커뮤니티 Slack | <https://influxdata.com/slack> |

---

## 1. Telegraf란 무엇인가

**Telegraf는 InfluxData가 만든 오픈소스 메트릭 수집 에이전트(데몬)입니다.**

한 줄 요약: **서버 · 장비 · 애플리케이션에서 데이터를 수집 → 가공 → 원하는
저장소로 전송해 주는 자동화된 수집기**.

- 언어: **Go 1.27** (`go.mod` 기준)
- 버전: **1.41.0** (`build_version.txt` 기준)
- 라이선스: **MIT** (상업적 이용/수정/재배포 자유)
- 특징: **의존성 없는 단일 바이너리**로 컴파일됨
- 설정: **TOML** 파일 한 개
- 커뮤니티: 컨트리뷰터 1,200명 이상

### 핵심 파이프라인

```text
[INPUT] → [PROCESSOR] → [AGGREGATOR] → [OUTPUT]
  수집        가공          집계          전송
```

- `agent.interval`(기본 10초)마다 수집
- `flush_interval`(기본 10초)마다 전송
- 위 과정을 무한 반복하는 백그라운드 데몬

### 저장소 구조

```text
telegraf/
├── cmd/telegraf/      # 실행 파일 진입점 (main.go)
├── agent/             # 수집→처리→전송 루프 엔진
├── plugins/           # 핵심. 300개 이상의 플러그인
│   ├── inputs/        (248) 어디서 수집할지
│   ├── outputs/       (71)  어디로 보낼지
│   ├── processors/    (39)  중간 가공
│   ├── aggregators/   (12)  집계 (평균/최소/최대/히스토그램)
│   ├── parsers/       (26)  입력 데이터 해석 (JSON/CSV/XML/Prometheus)
│   ├── serializers/   (18)  출력 포맷
│   └── secretstores/  (11)  비밀정보 보관 (Vault/OS 키체인 등)
├── docs/              # 30개 이상의 공식 문서
├── migrations/        # 구버전 설정 자동 마이그레이션
└── Makefile           # 빌드/테스트/패키징
```

### 최소 설정 예시

```toml
[[inputs.cpu]]      # CPU 수집
[[inputs.mem]]      # 메모리 수집
[[outputs.file]]    # 표준출력/파일로 전송
```

코드를 작성하지 않고 **설정 파일만으로** 동작하는 것이 Telegraf의 핵심 장점입니다.

### 주요 활용 사례

| 분야 | 활용 |
| --- | --- |
| 서버 모니터링 | CPU/메모리/디스크/네트워크 실시간 감시 |
| 컨테이너 운영 | Docker, Kubernetes 리소스 추적 |
| DB 성능 관리 | MySQL, PostgreSQL, Redis, MongoDB |
| 웹서비스 감시 | Nginx 응답시간, 헬스체크, SSL 인증서 만료(`x509_cert`) |
| 산업용 IoT | Modbus, OPC UA, S7comm, BACnet (스마트팩토리) |
| 스마트홈 | MQTT, HomeKit, 온습도 센서 |
| 클라우드 | AWS CloudWatch, Azure Monitor, GCP |

### 개발자에게 주는 이점

1. **무료로 모니터링 구축** — TIG 스택(Telegraf + InfluxDB + Grafana)
2. **설치가 단순** — 단일 바이너리 복사로 끝
3. **내 서비스와 연동 용이** — `inputs.exec` / `inputs.http` / `inputs.execd`
4. **아키텍처 학습 교재** — 깔끔한 플러그인 인터페이스 설계
5. **커리어 가치** — DevOps/SRE 채용 공고에 자주 등장하는 스택

---

## 2. 쉬운 비유로 이해하기

### 택배 기사 비유

- **INPUT(수거)**: "CPU님 지금 몇 %인가요?" 하고 데이터를 받아옴
- **PROCESSOR(분류)**: 불필요한 항목 제거, 태그 추가
- **AGGREGATOR(합치기)**: 1분치 10개를 평균 1개로 요약
- **OUTPUT(배달)**: InfluxDB 등 목적지로 전달

### 멀티탭 비유

- 왼쪽 구멍 248개(inputs) · 오른쪽 구멍 71개(outputs)
- 사용자는 **꽂기만 하면 됨** → 설정 두 줄로 수많은 조합을 구성

### 전체 그림 (TIG 스택)

```text
  내 서버        앱         공장 센서
      │           │            │
      └───────────┼────────────┘
                  ▼
           [ Telegraf ]   수집 · 정리
                  ▼
           [ InfluxDB ]   시계열 저장
                  ▼
           [ Grafana  ]   시각화
                  ▼
                사용자
```

### 자주 하는 오해

- **Telegraf는 데이터를 저장하지 않습니다.** 저장은 InfluxDB/Prometheus 담당
- **그래프를 그리지 않습니다.** 시각화는 Grafana 담당
- Telegraf는 그 사이를 잇는 **파이프 역할**

---

## 3. 자주 묻는 질문 (Q&A)

### Q1. 설치 및 사용법

**설치 방법**

```bash
# Docker (가장 빠름)
docker pull telegraf
docker run --rm -v $PWD/config.toml:/etc/telegraf/telegraf.conf telegraf

# 패키지 매니저
brew install telegraf          # macOS
sudo apt-get install telegraf  # Ubuntu/Debian
sudo yum install telegraf      # CentOS/RHEL
choco install telegraf         # Windows

# 소스 빌드 (이 저장소에서, Go 1.27 필요)
make            # 빌드
make test       # 테스트
make check      # fmt + vet 검사
make docker-image
```

**사용 3단계**

```bash
# 1) 설정 생성 (필요한 플러그인만 골라서)
telegraf config --input-filter cpu:mem:disk --output-filter influxdb_v2 > telegraf.conf

# 2) 테스트 (실제 전송 없이 화면 출력)
telegraf --config telegraf.conf --test

# 3) 실행
telegraf --config telegraf.conf
sudo systemctl start telegraf      # 서비스로 실행 (Linux)
telegraf --service install         # Windows 서비스 등록
```

**유용한 옵션**

```bash
telegraf --version
telegraf --config-directory /etc/telegraf/telegraf.d/
telegraf --debug
telegraf --once
telegraf plugins inputs
```

**실전 설정 예시**

```toml
[agent]
  interval = "10s"
  flush_interval = "10s"
  hostname = "my-server"

[[inputs.cpu]]
  percpu = true
  totalcpu = true

[[inputs.mem]]

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs"]

[[processors.rename]]
  [[processors.rename.replace]]
    tag = "host"
    dest = "server_name"

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "${INFLUX_TOKEN}"
  organization = "my-org"
  bucket = "metrics"
```

### Q2. 플러그인인가, 스킬인가, MCP인가?

**셋 다 아닙니다. 독립 실행형 에이전트(데몬) 소프트웨어입니다.**

| 구분 | Telegraf 해당 여부 | 설명 |
| --- | --- | --- |
| 플러그인 | ❌ | Telegraf가 본체이고, 내부에 플러그인 300개를 품은 구조 |
| Claude Skill | ❌ | AI 관련 개념과 무관 |
| MCP | ❌ | Telegraf는 2015년 등장, MCP는 2024년 등장한 별개 계층 |

유사 제품군: Prometheus Node Exporter, Fluentd, Logstash, Vector, Datadog Agent

### Q3. API 토큰이 필요한가?

**용도에 따라 다릅니다.**

- **불필요**: 로컬 시스템 수집(`cpu`, `mem`, `disk`, 로컬 `docker`), 파일 출력
- **필요**: InfluxDB v2/v3(API Token), Datadog/New Relic(API Key),
  AWS/GCP/Azure(자격증명), DB 계정, GitHub input 등

**안전하게 다루는 방법**

1. 환경변수 참조

   ```toml
   token = "${INFLUX_TOKEN}"
   ```

2. Secret Store 사용 (`plugins/secretstores/`: os, jose, vault, docker,
   systemd, http, oauth2, googlecloud 등 11종)

   ```bash
   telegraf secrets set mystore influx_token
   ```

   ```toml
   [[secretstores.os]]
     id = "mystore"

   [[outputs.influxdb_v2]]
     token = "@{mystore:influx_token}"
   ```

3. 파일 권한 제한: `chmod 600 telegraf.conf`

> ⚠️ 토큰을 설정 파일에 평문으로 적고 저장소에 커밋하지 마세요.
> `.gitignore`에 반드시 추가합니다.

### Q4. 왜 GitHub에서 유명한가?

1. **설정 파일만으로 동작** — 진입 장벽이 매우 낮음
2. **플러그인 300개 이상** — 지원하지 않는 대상이 거의 없음
3. **단일 바이너리** — 런타임/의존성 설치 불필요, 어디서나 실행
4. **기여 장벽이 낮은 설계** — 함수 2개만 구현하면 플러그인 완성

   ```go
   func (s *MyPlugin) SampleConfig() string { ... }
   func (s *MyPlugin) Gather(acc telegraf.Accumulator) error { ... }
   ```

5. **기업(InfluxData)이 유지보수** — 지속성 보장, 정기 릴리스
6. **MIT 라이선스** — 상업적 활용 자유 → 기업 채택률 상승
7. **TIG 스택의 표준 구성원** — Grafana 생태계와 함께 성장
8. **산업용 IoT 사실상 표준** — Modbus/OPC UA 지원 경량 에이전트가 희소

### Q5. 로컬 에이전트 구축에 도움이 되는가?

**A. 로컬 모니터링 에이전트 구축 → 매우 적합.** 그 자체가 본래 용도입니다.

**B. 로컬 AI 에이전트(LLM Agent) 구축 → 간접적으로 유용.**

- AI 에이전트의 **감각기관** 역할: Telegraf 수집 → 시계열 DB → AI가 조회/판단
- **아키텍처 학습**: `SampleConfig()`는 MCP의 tool schema,
  `Gather()`는 tool execute와 개념적으로 동일 → MCP 서버 설계 참고 자료
- **에이전트 자체 모니터링**: 토큰 사용량, 응답시간, 에러율 수집
- **`execd`로 확장**: Python 등 외부 스크립트를 플러그인처럼 연결

  ```toml
  [[inputs.execd]]
    command = ["python3", "my_ai_collector.py"]
    signal = "none"
  ```

**불가능한 것**: LLM 호출/추론, 복잡한 조건 분기 워크플로, 자연어 처리

### Q6. 수익화 아이디어

상세 내용은 아래 4장 참고. 요약하면 다음 5가지가 대표적입니다.

1. Telegraf 설정 GUI 빌더 (SaaS)
2. 스마트팩토리 모니터링 SI/구축
3. 업종별 Grafana 대시보드 템플릿 판매
4. 한국 특화 커스텀 플러그인
5. 한국어 강의/eBook 콘텐츠

### Q7. React나 PHP로 만들 수 있는가?

**Telegraf 본체를 포팅하는 것은 비현실적입니다.**

| 이유 | 설명 |
| --- | --- |
| 성능 | 수천 개 메트릭의 동시 수집에는 Go의 고루틴 병렬성이 필요 |
| 배포 | 단일 바이너리는 Go의 강점, PHP/Node는 런타임·의존성 필요 |
| 시스템 접근 | `/proc`, SNMP, Modbus, WMI 등 저수준 접근에 Go가 유리 |
| 물량 | 플러그인 300개 재구현은 현실적으로 불가능 |

**대신 주변 생태계는 React/PHP로 충분히 만들 수 있고, 사업성도 더 높습니다.**

React로 만들 수 있는 것:

- 설정 파일 GUI 빌더 (드래그앤드롭 → TOML 생성)
- 커스텀 대시보드 (InfluxDB API + Recharts/D3)
- 멀티 에이전트 관리 콘솔
- 알림 임계치 관리 UI

PHP로 만들 수 있는 것:

- 관리 백엔드 (Laravel): 설정 저장, 에이전트 배포, 사용자 관리
- 메트릭 수신 엔드포인트 (`outputs.http` 수신부)
- Telegraf가 수집해 갈 지표 API 제공

```php
<?php
// Telegraf inputs.http 가 주기적으로 이 엔드포인트를 수집
header('Content-Type: application/json');
echo json_encode([
    'active_users'  => User::whereActive()->count(),
    'orders_today'  => Order::today()->count(),
    'revenue_today' => Order::today()->sum('amount'),
]);
```

```toml
[[inputs.http]]
  urls = ["https://example.com/metrics.php"]
  data_format = "json"
  interval = "10s"
```

`execd`를 쓰면 PHP로 작성한 스크립트를 플러그인처럼 연결할 수도 있습니다.

**권장 아키텍처**

```text
[ Telegraf (Go, 수정 없이 사용) ]   ← 수집
             ▼
[ InfluxDB / TimescaleDB ]          ← 저장
             ▼
[ PHP(Laravel) / Node 백엔드 ]      ← 직접 구현
             ▼
[ React 프론트엔드 ]                ← 직접 구현
```

핵심: **수집은 Telegraf에 맡기고, 사용자 경험(UX) 영역에 집중**하는 것이
가장 효율적입니다.

---

## 4. 수익화 아이디어 상세

> 라이선스 유의사항: Telegraf는 MIT 라이선스로 상업적 이용이 자유롭습니다.
> 다만 "Telegraf" 상표를 제품명으로 쓰는 것은 피하고
> ("Telegraf 기반", "Powered by Telegraf" 정도로 표기),
> 바이너리 재배포 시 **MIT 라이선스 고지문을 포함**해야 합니다.

### TIER 1 — 진입 장벽이 낮고 즉시 시작 가능

**1) 한국어 콘텐츠 사업**

한글 자료가 매우 부족한 것이 기회입니다.

| 상품 | 가격대 | 난이도 |
| --- | --- | --- |
| 온라인 강의 (인프런/유데미) | 5~15만원 | 중 |
| eBook | 1~3만원 | 하 |
| 블로그 + 애드센스 | 광고 수익 | 하 |
| 유튜브 튜토리얼 | 광고/협찬 | 중 |

커리큘럼 예시: 모니터링 개요 → Docker 설치 → 대시보드 구축 → 슬랙 알림 →
커스텀 플러그인 제작 → 실전 운영

**2) Grafana 대시보드 템플릿 판매**

구성: Telegraf 설정(.conf) + Grafana JSON + 설치 가이드 + 알림 규칙

- 이커머스 운영 팩 / 게임서버 팩 / 공장 설비 팩 / K8s 팩 / DB 진단 팩
- 판매처: Gumroad, Lemon Squeezy, 크몽 등
- 장점: 한 번 제작 후 반복 판매 가능

**3) 한국 특화 커스텀 플러그인**

`EXTERNAL_PLUGINS.md`의 외부 플러그인 목록에 한국 서비스 연동이 거의 없습니다.

- 네이버 클라우드(NCP), KT/LG U+ 클라우드 메트릭
- 국내 PG사(토스페이먼츠, 아임포트) 트랜잭션 모니터
- 카페24/고도몰/메이크샵 쇼핑몰 지표
- 카카오 알림톡 output 플러그인
- 국내 제조사 PLC/HMI 커넥터

수익 모델: 오픈소스 배포로 인지도 확보 → 컨설팅 유입, 또는 기업 라이선스 판매

**4) Telegraf 설정 생성기 (웹 SaaS)**

- 왼쪽: 플러그인 검색/선택 → 가운데: 옵션 폼 → 오른쪽: TOML 실시간 미리보기
- **구현 팁**: 각 플러그인 폴더의 `sample.conf`와 `README.md`를 파싱하면
  248개 입력 폼을 **자동 생성**할 수 있어 개발량이 크게 줄어듭니다.
- 과금: Free(생성/다운로드) / Pro $9(저장·팀공유·검증) / Team $29(다중 프로필·API)

### TIER 2 — 중간 난이도, 높은 수익성

**5) 스마트팩토리 모니터링 SI/컨설팅 (수익성 최상)**

Modbus, OPC UA, S7comm, BACnet을 지원하는 경량 에이전트는 사실상 Telegraf뿐이며,
기존 SCADA 솔루션 대비 원가가 극히 낮습니다.

| 항목 | 단가 |
| --- | --- |
| 초기 구축 (중소 공장) | 1,000~3,000만원 |
| 유지보수 | 월 50~150만원 |
| 추가 설비 연동 | 건당 200~500만원 |

진입 전략: 정부 스마트공장 지원사업 과제로 레퍼런스 확보

**6) 매니지드 모니터링 서비스 (한국형 Datadog)**

| 항목 | Datadog | 자체 서비스 |
| --- | --- | --- |
| 서버당 월 요금 | $15~23 | 약 1만원 |
| 한국어 지원 | 없음 | 있음 |
| 카카오톡/네이버웍스 알림 | 없음 | 있음 |
| 국내 데이터센터 | 없음 | 있음 (규제 대응) |

**7) 어플라이언스(하드웨어) 판매**

라즈베리파이 + Telegraf + InfluxDB + Grafana 프리셋 = "꽂으면 되는 모니터링 박스".
원가 대비 설정·지원 포함 판매, 원격관리 구독 추가 가능.

**8) 기업 대상 구축/마이그레이션 컨설팅**

- Datadog 비용 절감 전환, Zabbix/Nagios 현대화
- 아키텍처 설계 500~1,000만원 / 구축 2,000~5,000만원 /
  기술지원 연간계약 1,200~3,600만원

### TIER 3 — 장기 투자형

**9) AI 기반 이상탐지 SaaS**

- 자연어 질의("어제 새벽에 왜 느려졌나?")로 메트릭 분석
- 이상 징후 예측, 장애 리포트 자동 생성
- MCP 서버 제공으로 LLM이 직접 메트릭 조회
- Telegraf가 수집 문제를 이미 해결했으므로 AI 계층만 얹으면 됨

**10) 산업별 턴키 솔루션 (수직 특화)**

| 업종 | 수집 대상 |
| --- | --- |
| 프랜차이즈 | POS, 냉장고 온도, 네트워크, CCTV 상태 |
| 병원 | 의료장비 가동률, 서버, 백업 상태 |
| 스마트팜 | 온습도, CO2, 급수, 조도 |
| 에너지 | 태양광 인버터, ESS, 전력량계 |
| 게임사 | TPS, 동시접속, 매칭시간, DB 부하 |

업종 솔루션은 범용 모니터링 툴보다 단가를 훨씬 높게 책정할 수 있습니다.

**11) 플러그인 마켓플레이스 운영** — 거래 수수료 20~30% 모델

**12) 교육 인증 사업** — 기업 단체 교육 중심

### 추천 로드맵

```text
0~3개월    한국어 콘텐츠 시작 + 대시보드 템플릿 3종 판매
3~9개월    React 설정 생성기 웹서비스 출시 + 온라인 강의 런칭
9~18개월   매니지드 모니터링 SaaS 베타 또는 스마트팩토리 SI 진출
18개월~    AI 이상탐지 기능 탑재 프리미엄 SaaS
```

**순서의 근거**

- 콘텐츠를 먼저 하는 이유: 수익을 내면서 **시장의 실제 불편함을 학습**
- 템플릿 → 도구 순서: 작은 성공으로 초기 고객 리스트 확보
- SaaS를 나중에: 앞 단계에서 모은 사용자가 초기 유저가 되어 콜드스타트 해소

**하나만 고른다면: Telegraf 설정 생성기 웹서비스**

1. React/PHP 스택과 정확히 일치
2. `sample.conf` 파싱으로 UI 자동 생성 → 개발량 대폭 감소
3. 무료 공개 시 글로벌 트래픽 확보 가능
4. "telegraf config generator" 검색 상위 노출 시 지속적 유입
5. 확보한 사용자가 이후 SaaS의 초기 고객이 됨

---

## 참고 문서 (저장소 내부)

| 문서 | 내용 |
| --- | --- |
| `docs/QUICK_START.md` | 5분 만에 시작하기 |
| `docs/INSTALL_GUIDE.md` | 모든 설치 방법 |
| `docs/CONFIGURATION.md` | 설정 문법 전체 |
| `docs/COMMANDS_AND_FLAGS.md` | CLI 옵션 전체 |
| `docs/EXTERNAL_PLUGINS.md` | 외부 플러그인 작성법 |
| `docs/SECRETSTORES.md` | 비밀정보 관리 |
| `docs/DATA_FORMATS_INPUT.md` | 입력 파서 종류 |
| `docs/PARSING_DATA.md` | 임의 데이터 파싱 가이드 |
| `CONTRIBUTING.md` | 기여 방법 |
| `plugins/inputs/example/` | 플러그인 제작 템플릿 |
