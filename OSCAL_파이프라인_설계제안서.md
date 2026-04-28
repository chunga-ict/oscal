# OSCAL Compliance Intelligence Pipeline — 프로덕션 아키텍처 설계 제안서

**대상 환경:** 한국 기업 (K-ISMS-P, ISO27001, PCI DSS, SOC2, GDPR)  
**PoC 기반:** `index.html` (1,184줄 단일 HTML)  
**작성일:** 2026-04-28

---

## 1. 핵심 방향

PoC의 **개념 설계는 그대로 유지**, 7대 구조 문제를 해결하며 프로덕션 수준으로 전환합니다.

---

## 2. 기술 스택

| 레이어 | 선택 | 이유 |
|--------|------|------|
| 프론트엔드 | **Next.js 14 + TypeScript** | Babel Standalone CDN 제거, SSR, 타입 안전성 |
| UI | **shadcn/ui + Tailwind CSS** | PoC 다크 테마 유지, 접근성 확보 |
| 백엔드 | **FastAPI (Python 3.12)** | IBM Trestle 필수, Pydantic으로 OSCAL 스키마 검증, async |
| OSCAL 엔진 | **IBM Compliance Trestle 3.x** | SSP 자동 조립, Component import, SAR 생성 |
| OSCAL 검증 | **oscal-cli (NIST 공식)** | 생성된 JSON의 스키마 적합성 보장 |
| DB | **PostgreSQL 16 (JSONB)** | OSCAL 문서 전체를 JSONB로 저장하면서 SQL 쿼리 가능 |
| 파일 스토어 | **MinIO (S3 호환)** | 온프레미스 배포, OSCAL JSON 원본 저장 |
| 태스크 큐 | **Celery + Redis** | GAP 분석·증거 수집 등 비동기 파이프라인 |
| 실시간 통신 | **SSE (Server-Sent Events)** | `setInterval` 시뮬레이션 → 실제 서버 푸시 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────────┐
│            Next.js 14 + TypeScript (프론트엔드)           │
│  파이프라인 · 갭분석 · 솔루션매핑 · OSCAL뷰어 · 모니터링 │
└──────────────────┬───────────────────────────────────────┘
                   │ HTTPS / SSE
┌──────────────────▼───────────────────────────────────────┐
│               Kong Gateway (인증·라우팅·속도제한)         │
└──────┬───────────────────────┬───────────────────────────┘
       │                       │
┌──────▼───────┐   ┌───────────▼─────────────────────────┐
│  Auth API    │   │   Core API + OSCAL Pipeline Engine   │
│  FastAPI     │   │   FastAPI + Celery Workers           │
│  JWT·MFA·RBAC│   │                                     │
│              │   │  Worker 1: GAP Analysis              │
│              │   │  Worker 2: OSCAL Generator (Trestle) │
│              │   │  Worker 3: Evidence Collector        │
│              │   │  Worker 4: SSP Assembler             │
└──────┬───────┘   └───────────┬─────────────────────────┘
       │                       │
┌──────▼───────────────────────▼───────────────────────────┐
│   PostgreSQL 16 (JSONB) │ Redis 7 │ MinIO (OSCAL 파일)   │
└──────────────────────────────────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────────────┐
│           External Integrations                           │
│   Okta · MS Graph · Tenable · Splunk · HashiCorp Vault   │
│   NVD CVE 피드 · KISA 취약점 DB · NIST OSCAL 카탈로그    │
└──────────────────────────────────────────────────────────┘
```

---

## 4. PoC 7대 문제 해결 매핑

| # | PoC 문제 | 해결 |
|---|---------|------|
| 1 | `Math.random()` UUID | Python `uuid.uuid4()` (RFC 4122) |
| 2 | `SOLUTIONS_DB` 하드코딩 | PostgreSQL DB + Admin API |
| 3 | Babel Standalone CDN | Next.js + Turbopack 빌드 |
| 4 | `setInterval` 시뮬레이션 | FastAPI SSE + Redis pub/sub |
| 5 | 인증 없음 | JWT + TOTP MFA + RBAC |
| 6 | 새로고침 시 상태 소실 | PostgreSQL 영속성 + React Query 캐시 |
| 7 | OSCAL 스키마 검증 없음 | oscal-cli CI 파이프라인 통합 |

---

## 5. 핵심 추가 기능

### 5-1. K-ISMS-P OSCAL 카탈로그 (최우선 과제)

현재 NIST 800-53 OSCAL 카탈로그는 공식 제공되지만 **K-ISMS-P OSCAL 카탈로그는 존재하지 않습니다.**  
80개 인증기준을 OSCAL 형식으로 직접 구축하면 경쟁사가 단기간에 복제 불가능한 기술 자산이 됩니다.

```json
{
  "id": "kisms-2.2.1",
  "title": "정보보호 정책의 승인",
  "props": [
    { "name": "regulation", "value": "K-ISMS-P" },
    { "name": "category",   "value": "정책및조직" }
  ]
}
```

### 5-2. 자동화 파이프라인 (Celery 체인)

```python
pipeline = chain(
    build_compliance_profile.s(project_id),  # S1: Profile 생성
    run_gap_analysis.s(),                    # S2: GAP 분석
    discover_solutions.s(),                  # S3: 솔루션 매핑
    generate_oscal_components.s(),           # S4: Component 생성
    generate_api_contracts.s(),              # S5: Contract 생성
    run_auto_verification.s(),               # S6: 자동 검증
    update_monitoring_state.s(),             # S7: 모니터링 갱신
)

# S7 → S2 피드백 루프 (매일 오전 6시)
@beat.task(crontab(hour=6))
def check_monitoring_and_rerun_gap(): ...

# 15분마다 API Contract 검증 (PoC의 "verification-sla": "15min" 실제 구현)
@beat.task(run_every=900)
def verify_active_contracts(): ...
```

---

## 6. 데이터 모델

### 6-1. 핵심 엔티티 관계

```
Organization ─── Project ─── GapFinding ─── Solution
                    │              │              │
               OscalDocument  PoamItem      ApiContract
                    │
                MinIO (파일 원본)
```

### 6-2. 주요 엔티티

| 엔티티 | 핵심 필드 |
|--------|----------|
| `Organization` | id(UUID4), name, applicable_regulations[], compliance_profile(JSONB) |
| `Project` | id, organization_id, target_regulations[], status, oscal_ssp_id |
| `GapFinding` | id, domain_code, control_id, regulation_ref, severity, status |
| `Solution` | id, name, tier, deployment_type, api_type, domain_coverage[], control_ids[] |
| `ApiContract` | id, gap_finding_id, solution_id, contract_document(JSONB), status |
| `OscalDocument` | id, document_type, content(JSONB), storage_key(MinIO), schema_valid |
| `ComplianceEvent` | id, project_id, event_type, severity, occurred_at |

### 6-3. UUID 전략

| 상황 | 방법 |
|------|------|
| 일반 엔티티 | `uuid.uuid4()` — RFC 4122 Type 4 |
| 동일 솔루션+도메인 조합 (재현 필요) | `uuid.uuid5(NAMESPACE_URL, f"{domain}-{solution_id}")` |
| PoC 방식 (`Math.random()`) | **전면 제거** |

---

## 7. REST API 구조

```
/api/v1/
├── auth/
│   ├── POST   /login
│   ├── POST   /mfa/verify
│   └── POST   /token/refresh
│
├── organizations/{org_id}/
│   ├── GET    /profile
│   ├── PUT    /profile
│   └── GET    /compliance-score
│
├── projects/{project_id}/
│   ├── POST   /pipeline/run           ← 전체 파이프라인 실행
│   │
│   ├── gap-findings/
│   │   ├── GET    /                   ← 갭 목록 (severity/domain/status 필터)
│   │   └── PATCH  /{gap_id}           ← 상태 업데이트
│   │
│   ├── solutions/
│   │   └── GET    /recommend          ← 도메인별 자동 추천
│   │
│   ├── oscal/
│   │   ├── POST   /components/generate
│   │   ├── POST   /ssp/assemble
│   │   ├── POST   /poam/generate
│   │   └── POST   /validate/{doc_id}
│   │
│   ├── contracts/
│   │   ├── POST   /generate
│   │   └── POST   /{id}/send          ← 벤더 전달
│   │
│   └── monitoring/
│       ├── GET    /stream             ← SSE 이벤트 스트림
│       └── GET    /scores             ← 도메인별 준수율
│
└── admin/
    ├── solutions/                     ← SOLUTIONS_DB 관리
    └── catalogs/                      ← K-ISMS-P 카탈로그 관리
```

### 벤더 표준 엔드포인트 (PoC API Contract 실제 구현)

```
GET  /oscal/compliance-status        ← 통제 준수 상태 (SSP 형식)
POST /oscal/evidence/collect         ← 감사 증거 자동 수집 (SAR 형식)
POST /oscal/gap-remediation          ← 갭 조치 트리거 (POA&M 연동)
```

---

## 8. RBAC 역할 설계

K-ISMS-P 인증기준 2.2.x (정보보호 역할 및 책임) 반영:

| 역할 | 권한 |
|------|------|
| `SuperAdmin` | 플랫폼 전체 관리 |
| `OrgAdmin` | 조직 내 모든 기능 |
| `ComplianceOwner` | 프로젝트 관리, 갭 승인, 보고서 생성 |
| `SecurityAnalyst` | 갭 분석, 솔루션 선택, OSCAL 생성 |
| `Auditor` | 읽기 전용 + 감사 증거 다운로드 |
| `Vendor` | 특정 API Contract 조회만 허용 |

**인증 흐름:** LDAP/SAML 연동 → TOTP MFA → JWT 발급 (Access 15분, Refresh 7일) → Kong에서 검증

---

## 9. 배포 아키텍처

### 컨테이너 구성

| 서비스 | 기술 | 리소스 |
|--------|------|--------|
| frontend | Next.js | 2 replicas, 512MB |
| api-core | FastAPI | 3 replicas, 1GB |
| api-auth | FastAPI | 2 replicas, 512MB |
| worker-gap | Celery | 2 replicas, 2GB |
| worker-oscal | Celery + Trestle | 2 replicas, 4GB |
| worker-evidence | Celery | 3 replicas, 1GB |
| postgres | PostgreSQL 16 | primary + replica |
| redis | Redis 7 | primary |
| minio | MinIO | 분산 4 nodes |
| kong | Kong Gateway | 2 replicas |

### 환경 구성

| 환경 | 구성 |
|------|------|
| dev | Docker Compose, 단일 노드, 로컬 MinIO |
| staging | k3s, 실제 솔루션 API 샌드박스 연동 |
| prod | EKS 또는 온프레미스 K8s, HA 구성 |

### 한국 기업 배포 고려사항

- **온프레미스 우선:** 금융/의료 기업의 클라우드 외부 SaaS 불가 환경 → MinIO로 데이터 주권 확보
- **인터넷 단절 환경:** KISA 취약점 DB, NVD 미러링 (Celery Beat 야간 동기화)
- **개인정보 보호법:** PostgreSQL `pgcrypto`로 PII 암호화, 한국 리전 데이터 격리

---

## 10. 개발 로드맵

### Phase 1 — 기반 구축 (8주)

| 주차 | 작업 |
|------|------|
| 1-2 | 모노레포 설정 (Next.js + FastAPI), Docker Compose 환경, Alembic DB 마이그레이션 |
| 3-4 | JWT + MFA + RBAC 인증 서비스, Kong Gateway 통합 |
| 5-6 | FastAPI CRUD API (조직·프로젝트·갭·솔루션), React Query 연동 |
| 7-8 | **K-ISMS-P OSCAL Catalog 1차 구축** ← 최우선, UUID 체계 전환 |

**산출물:** 하드코딩 데이터 제거 + 실제 DB 기반 동작하는 대시보드

### Phase 2 — OSCAL 파이프라인 엔진 (10주)

| 주차 | 작업 |
|------|------|
| 1-3 | IBM Trestle 통합, Component.json 생성 서비스, oscal-cli 검증 CI 통합 |
| 4-5 | API Contract 생성 엔진 (buildAPIContract Python 이관 + UUID 교체) |
| 6-7 | Celery 태스크 체인, MinIO 파일 저장 파이프라인 |
| 8-10 | **Okta + Tenable.io 실제 API 통합** (엔드투엔드 첫 증거 수집) |

**산출물:** Okta MFA 상태를 자동 수집해 OSCAL SAR로 저장하는 엔드투엔드 파이프라인

### Phase 3 — 실시간 모니터링 & 보고 (8주)

| 주차 | 작업 |
|------|------|
| 1-3 | SSE 기반 실시간 이벤트 스트림, Redis pub/sub, S7→S2 피드백 루프 |
| 4-5 | NVD CVE 피드 연동, KISA 취약점 DB 미러링 |
| 6-7 | PDF/Excel 감사 보고서 자동 생성 (WeasyPrint + OSCAL SAR) |
| 8 | K-ISMS-P 인증 심사용 증거 패키지 자동 생성 |

**산출물:** 실제 준수율이 실시간으로 변동하는 살아있는 대시보드

### Phase 4 — 엔터프라이즈 (6주)

| 주차 | 작업 |
|------|------|
| 1-2 | 멀티테넌시 (PostgreSQL Row-Level Security), Kong 테넌트 라우팅 |
| 3-4 | SAML/OIDC SSO (기업 AD/Azure AD), LDAP 그룹 → RBAC 자동 매핑 |
| 5-6 | K8s 프로덕션 배포, Prometheus + Grafana, 부하 테스트 |

---

## 11. PoC 재활용 전략

### 그대로 가져갈 것

| PoC 자산 | 재활용 방법 |
|----------|------------|
| 7단계 파이프라인 구조 | 프론트엔드 상태 머신의 기반 |
| `buildOSCALComponent` JSON 구조 | Python 백엔드 OSCAL 생성 로직의 명세서 |
| `x-oscal-metadata` 확장 필드 설계 | 벤더 계약서 표준으로 그대로 채택 |
| `ctrlMap` (도메인→컨트롤 매핑) | DB `Solution.control_ids` 초기 시드 데이터 |
| `SOLUTIONS_DB` 20개 솔루션 | DB 시드 데이터로 직접 이관 |
| 다크 테마 색상 팔레트 (`C` 객체) | Tailwind CSS 커스텀 테마 변수로 변환 |
| GitHub Actions 워크플로우 템플릿 | `.github/workflows/oscal-verify.yml`로 이식 |

### 교체할 것

| PoC 문제 코드 | 교체 내용 |
|--------------|----------|
| `Math.random().toString(36)` | `uuid.uuid4()` (Python 백엔드) |
| `SOLUTIONS_DB`, `GAP_DATA` 하드코딩 | PostgreSQL DB + FastAPI REST |
| `<script type="text/babel">` | Next.js + Turbopack 빌드 |
| `setInterval` 시뮬레이션 | FastAPI SSE + Redis pub/sub |
| 인라인 스타일 객체 (`s`, `C`) | Tailwind CSS + shadcn/ui 컴포넌트 |

---

## 12. 기술 리스크 및 완화 방안

| 리스크 | 가능성 | 영향도 | 완화 방안 |
|--------|--------|--------|----------|
| IBM Trestle Python 3.12 호환성 이슈 | 중간 | 높음 | Trestle 전용 Python 3.11 Docker 컨테이너 격리 |
| K-ISMS-P OSCAL 카탈로그 작성 공수 과소평가 | 높음 | 매우 높음 | Phase 1 최우선 착수, KISA 협력 검토 |
| 벤더 API Contract 이행 거부 | 높음 | 높음 | 계약 조항에 OSCAL 엔드포인트 구현 의무화, 수동 폴백 |
| 대용량 OSCAL JSON JSONB 성능 | 낮음 | 중간 | 문서 원본은 MinIO, DB에는 메타데이터만 저장 |
| 멀티테넌트 데이터 격리 위반 | 낮음 | 치명적 | PostgreSQL RLS + 서비스 레이어 tenant_id 필터 의무화 |

---

## 13. 우선순위 결론

1. **K-ISMS-P OSCAL 카탈로그** — 시장 차별화의 핵심, 모든 후속 작업의 기반
2. **실제 API 통합 (Okta 1개)** — 데모 수준을 넘어서는 결정적 증거
3. **PoC UI 자산 최대 활용** — 재작성 최소화, UX 개념과 색상 체계 이식

---

*문서 생성: Claude Sonnet 4.6 / 2026-04-28*
