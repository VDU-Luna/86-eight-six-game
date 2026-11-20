# 01. Roblox 기술 아키텍처 (Technical Architecture)

- 작성일: 2026-11-20 · 문서 버전: v0.1 · 기준: `main` / `9d2d42b`
- 상태: 공식 문서 확인 사실 / 설계 추론 / 제안 정책을 구분 표기. 모든 제안은 `PROPOSED — NOT APPROVED`.
- 표기: 【사실】 공식 문서로 확인, 【추론】 확인 사실에서 도출, 【제안】 Claude 정책, 【검증 필요】 실측 필요.

---

## 1. Experience / Place 구조

### 기존 결정 (출처: MASTER_GDD §13, DECISIONS_LOG 2026-11-15 모드 서버 분리)
- 하나의 Experience, 모드별 Place. 메인 화면은 공통 진입 Place.
- 오픈월드 서버별 월드 독립. 계정 데이터와 서버 월드 상태 분리 저장.

### 제안 (CHANGE 없음, 구체화 NEW)
| Place | 용도 | 알파 포함 |
|---|---|---|
| `Lobby` | 메인 메뉴, 정비고, 창고, 계정 로드 확인 | 포함 |
| `OpenWorld` | 검증 지역 전투·임무 | 포함 |
| `Story / PvE / PvP` | 전용 Place 자리만 예약 | 미포함(메뉴 '개발 중') |

- Lobby ↔ OpenWorld 이동은 **서버 측 `TeleportService:TeleportAsync()`만 사용**한다. 공식 문서상 클라이언트 측 `Teleport()`는 deprecated이므로 사용하지 않는다【사실】.
- 텔레포트 시 로컬 데이터는 버려진다【사실】. 따라서 계정 데이터는 Lobby 진입 시 DataStore에서 로드하고, 텔레포트 데이터(`TeleportOptions:SetTeleportData`)에는 **비보안 정보(예: 선택한 기체 ID)만** 담는다【사실: 공식 문서가 민감 데이터 전송 금지를 명시】.

### 월드 인스턴스 방식 (D6과 연결)
- 옵션 A: 모든 OpenWorld 서버가 **슬롯 ID**를 받아 해당 슬롯의 스냅샷을 로드(구현 단순, 슬롯 동시 접속 제한 필요).
- 옵션 B: 예약 서버(Reserved Server) 방식 — **【검증 필요】** 알파에서 실제 지원 여부와 제약을 확인해야 함.
- **권장: 옵션 A로 시작.** 슬롯당 동시에 한 서버만 쓰도록 세션 잠금(아래 §5)으로 보장하고, 서버 수 증가 시 옵션 B 검토.

## 2. 서버 / 클라이언트 책임 분리 (핵심)

Roblox 공식 문서 요지【사실】: 서버 변경은 자동 복제되고, 클라이언트 변경은 원칙적으로 서버에 복제되지 않는다(예외: 클라이언트가 소유한 파트 물리, 입력 액션 시스템). Remote는 기본적으로 보안되지 않으므로 서버에서 검증해야 한다. 클라이언트는 원격 이벤트를 임의 빈도·인자로 호출할 수 있다.

| 영역 | 권위 | 클라이언트 역할 | 근거 |
|---|---|---|---|
| 기체 위치·회전(이동) | 탑승자 네트워크 소유 + **서버 검증 보정** | 입력 적용, 즉시 반응 | 반응성. 소유 파트는 악용 가능 → 검증 필요【추론】 |
| 갈고리 부착 판정(레이캐스트, 거리, 사거리) | **서버** | 조준 방향 전송, 연출 | 재화·상태 영향 |
| 갈고리 에너지 소비·해제 | **서버** | 에너지 HUD 표시(예측 가능) | 악용 방지 |
| 탄약·재장전 | **서버** | 발사 연출, 재장전 UI | 자원 영향 |
| 발사체 이동·명중 | **서버** | 투사체 시각 보간 | 다수 투사체 비용 관리 |
| 피해·부위 손상·파괴 | **서버** | HUD 반영, 효과 | 전투 공정성 |
| 대시 시작·종료 | **서버 판정**(이동은 탑승자 소유) | 즉시 연출 | 쿨다운·에너지 검증 |
| 파일럿 체력·부활 | **서버** | 피드백 | 상태 |
| 수리 진행 | **서버 타이머** | 진행 바 | 중단 규칙 판정 |
| 재화·XP·보상·구매 | **서버** | 표시만 | 경제 |
| 기체 소유·업그레이드 | **서버 (DataStore 기반)** | 목록 조회 | 데이터 정합 |
| Legion AI·스폰·웨이브 | **서버** | 연출만 | 성능과 공정성 |

### 2.1 기체 이동의 권한 모델 (상세 제안)
- **제안 P-MOVE**: 탑승자의 클라이언트가 기체 본체의 위치·속도를 시뮬레이션하고 네트워크 소유를 갖는다. 서버는 매 샘플(예: 0.1초 간격, 【검증 필요】)마다 다음을 검사한다.
  1. 최대 속도 상한 (기체별 데이터 값 + 여유율)
  2. 순간 이동 거리 상한 (텔레포트 감지)
  3. 에너지 상태와 이동 능력 일치 (에너지 0일 때 대시/갈고리 이동 불가)
  4. 대시 중 방향 고정 규칙 준수
  5. 이탈 시 마지막 유효 위치로 **서버가 보정 명령** (부드러운 보간, 튕김 최소화)
- 위반이 반복되면 해당 플레이어의 기체를 서버 소유로 전환하고 로그를 남긴다.
- **대안 비교**
  - 대안 1 (서버 완전 시뮬레이션): 악용 0에 가깝지만 입력 지연이 모바일에서 체감될 가능성이 높음. **알파의 조작감 목표와 충돌하므로 기각하되 비교 측정용으로 프로토타입 1개는 유지**.
  - 대안 2 (제안 P-MOVE): 반응성 우선, 검증 로직 비용 중간. **채택**.
  - 대안 3 (완전 클라이언트 신뢰): 기각(공식 문서가 명시적으로 위험 경고).

## 3. 네트워크 Remote 설계

### 원칙【사실+제안】
- 공식 문서 요지: `RemoteEvent`/`RemoteFunction`은 신뢰성·순서 보장. `UnreliableRemoteEvent`는 비신뢰·비순서이며 빈번한 갱신에 적합. 클라이언트가 임계치를 넘게 보내면 서버가 나중에 처리하며 순서 보존.
- **제안**: 상태 변화(피해, 재화, 탑승, 부착 성공)는 `RemoteEvent`. 위치 스냅샷·투사체 연출·Legion 위치 보간은 `UnreliableRemoteEvent`.
- **검증 항목 규칙**: 모든 클라이언트→서버 요청은 (a) 타입·길이·범위 검사, (b) 컨텍스트 권한(탑승 중인가, 쿨다운 끝났나, 에너지 충분한가), (c) 호출 빈도 제한을 통과해야 한다. 공식 문서가 강조한 대로 큰 문자열·테이블 대체 공격을 막는다.
- 허니팟 Remote(사용되지 않는 원격)를 1~2개 두고 호출 시 로그 + 세션 종료 대상으로 한다(악용 탐지, 비용 낮음).

### 대역폭 예산 (초기 가설)
- 플레이어당 클라이언트→서버 입력·상태: 최대 15 msg/s, 평균 5 msg/s.
- 서버→클라이언트 스냅샷: Feldress 최대 8대 + Legion 근거리 40기 기준 10Hz 보간용 요약, 기체당 약 80 bytes 가설【검증 필요】.
- 실제 수치는 `08`의 네트워크 측정으로 확정.

## 4. 물리 시뮬레이션과 네트워크 복제

【사실】 Roblox 물리 엔진은 서버와 클라이언트 간에 연산을 분산한다. 서버는 앵커 파트 소유를 바꿀 수 없고, 비앵커 파트는 근접도와 기기 성능에 따라 자동으로 클라이언트에 소유가 넘어갈 수 있다. 개발자는 `BasePart:SetNetworkOwner()`로 직접 지정할 수 있다.

【제안】
- **Feldress는 "단일 본체 + 가상 서스펜션" 구조**를 권장한다. 다리 수준 물리 링크(모든 다리를 개별 제약으로 연결)는 알파에서 금지.
- 본체 파트 1개 + 제약 최소 구성(【검증 필요】: 공식 구성 권장 대비 실제 비용 실측).
- 부품(무장·센서)은 **앵커 또는 용접된 시각 모델**로 두고 물리 시뮬레이션에서 제외한다. 손상·파괴 시 시각 효과만 바꾼다.
- 투사체는 **물리 파트를 만들지 않는다.** 서버가 프레임 단위 레이캐스트로 진행을 계산하고, 클라이언트에는 발사 이벤트만 보낸다.
- 폭발 파편 등 물리 연출은 **클라이언트 전용 장식**(서버 판정 없음)으로 한정하고 상한을 둔다.

## 5. 데이터 저장 (DataStore · 세션 잠금 · 자동 저장)

### 공식 사실 (확인일 2026-11-20)【사실】
- 데이터스토어 쓰기 한도는 **키당 최근 60초 기준** 약 4 MB/분, 읽기 약 25 MB/분으로 공식 문서에 명시되어 있다(`Data store error codes and limits`).
- 요청 큐는 키별로 관리되며 큐당 30개 제한. 키가 쓰로틀되면 요청이 대기한다.
- 공식 권장: 메모리에 로드 후 로컬 복사본을 갱신하고 주기적·퇴장 시·서버 종료 시·중요 체크포인트에서 저장. 자동 저장 주기는 세션 잠금 만료보다 짧게.
- 재시도는 `pcall` + 지수 백오프 + 지터, 키별 순서 보장. 실패한 쓰기는 성공 여부가 불확실할 수 있음.
- `BindToClose`는 서버 종료 시 **30초 이내**에 모든 바인딩 함수가 끝나야 하며, 초과 시 종료된다.
- `PlayerRemoving`은 서버 종료 시 함께 발생하지만 핸들러가 동기 실행된다(`ServiceProvider` 문서 요지).

### 제안 데이터 구조
| 저장소 | 키 | 내용 | 저장 주기 |
|---|---|---|---|
| `Account_v1` | UserId | 레벨, XP, 화폐, 재료, 기체 소유·업그레이드·무장 해금, 설정 | 60~180초 + 퇴장 + 종료 + 구매 직후 |
| `WorldSlot_v1` | SlotId | 기지 상태, 점령 지역 목록, 시설 상태, 웨이브 번호, 서버 사이클 번호 | 300초 + 종료 + 점령 변경 시 |
| `Ops_v1` | 날짜/기능 | 텔레메트리 요약, 악용 로그 | 쓰기 빈도 낮게 |

- **요청 예산**: 60~180초 주기 저장으로 동시 접속 8명 기준 한도 여유가 충분하다는 것이 공식 가이드의 방향이다. 실제 여유는 **【검증 필요】** Data Stores Observability Dashboard로 측정.
- **세션 잠금**: 한 계정은 한 시점에 한 서버에서만 쓰기 가능. 잠금 키에는 서버 ID와 만료 시각을 기록하고, 만료 전 갱신한다. 잠금 실패 시 로드 거부 + 재시도 안내.
- **서버 종료 처리 순서**: 종료 신호 → 모든 플레이어 상태 저장을 병렬 실행 → 저장 완료 대기(상한 25초, BindToClose 30초 이내) → 실패분은 로그와 재시도 큐 기록.
- **스키마 버전**: 필드마다 `v` 번호와 마이그레이션 함수. 데이터 손상 시 자동 덮어쓰기 금지, 백업 키 보관.

## 6. MemoryStore 사용 판단

【사실】 MemoryStore는 TTL 기반 임시 데이터로 빠른 접근에 적합하며, 메모리 쿼터(64 KB + 1.2 KB × 사용자 수 등 공식)와 요청 쿼터(1000 + 120 × 접속자 수/분)가 있다. 쿼터를 넘으면 메모리 증가 요청이 실패한다.

【제안】
- **알파에서 사용**: 서버 간 잠금 보조, 파티 초대 같은 짧은 상태, 월드 슬롯 점유 정보(TTL 기반).
- **알파에서 사용 안 함**: 계정 데이터, 월드 진행 원본, 재화 장부. 영속 데이터는 DataStore만.
- 쿼터 초과 시 기능 저하 경로: 파티 초대는 실패 안내, 슬롯 점유는 DataStore 잠금으로 폴백.

## 7. 텔레포트와 장애 복구

【사실】 `TeleportService:TeleportAsync()`는 플레이어 배열과 `TeleportOptions`를 받는다. `TeleportInitFailed` 이벤트와 `TeleportResult` 열거형(Failure, GameNotFound, GameFull, Unauthorized, Flooded, IsTeleporting 등)으로 실패를 식별할 수 있다. `Player.OnTeleport`로 `TeleportState` 단계(`Failed`, `InProgress` 등)를 관찰할 수 있다.

【제안 복구 규칙】
- 텔레포트 실패 시 로비에 머물고 재시도 1회 후 안내 표시. 재화·기체는 변하지 않음.
- 텔레포트 후 OpenWorld에서 로드 실패(잠금 충돌 등) 시 "데이터 대기" 화면 후 로비 복귀.
- 텔레포트 직전 저장이 실패하면 **텔레포트를 보류**한다(데이터 손실 방지 우선).

## 8. 스트리밍 (StreamingEnabled)

【사실】 Instance streaming은 Workspace 자손만 대상이다. ReplicatedStorage 등은 streaming 대상이 아니며 접속 시 전체 복제된다. 제약으로 연결된 인스턴스는 함께 스트리밍된다. `Model`의 `ModelStreamingMode`(Atomic, Persistent, PersistentPerPlayer, Nonatomic 등)로 동작을 지정한다. RemoteEvent로 보낸 인스턴스 참조는 클라이언트에 아직 없을 수 있다(공식 문서 경고).

【제안】
- `StreamingEnabled` 사용. 검증 지역은 약 2×2 km 가설이므로 **지역 내 전부를 항상 로드하지 않고** 구역 단위 스트리밍.
- 기체 모델은 `Atomic`, 기지·거점 구조물은 `Persistent`(알파 내 건물 수 제한), Legion은 `Nonatomic`.
- 템플릿(기체·무장·Legion)은 **ReplicatedStorage**에 두고 필요 시 클론.
- Remote로 Legion이나 기체 인스턴스를 참조할 때는 **ID 기반**으로 보내고, 클라이언트는 `WaitForChild`에 타임아웃을 둔다(공식 권장).
- 임무 시작 전 `RequestStreamAroundAsync` 등으로 주변 선로딩 (【검증 필요】: 알파 지역 크기에서 호출 비용).

## 9. 성능 예산 (초기 가설, 측정 후 조정)

| 항목 | 서버 목표 | 클라이언트 목표(PC / 모바일) | 측정 도구 |
|---|---|---|---|
| 서버 프레임 시간 | 평균 ≤ 16ms, p95 ≤ 25ms (60Hz 대비 여유) | — | Developer Console, MicroProfiler(【검증 필요】 서버 측 캡처 방식) |
| 활성 Feldress | 8대 (최대 16대 부하) | — | 로그 집계 |
| 활성 투사체 | 동시 ≤ 200 | 표시 ≤ 200 | 카운터 |
| Legion 활성 개체 | 근거리 ≤ 40, 중거리 ≤ 80, 원거리 추상 무제한 | 근거리 ≤ 40 표시 | AI 틱 시간 |
| 물리 파트 | 기체당 ≤ 25 (앵커 제외) | ≤ 3,000 (모바일 ≤ 1,500) | 인스턴스 카운트 |
| 메모리 | 서버 ≤ 1.5 GB 가설 | 모바일 ≤ 600 MB 가설 | 메모리 프로파일 |
| 네트워크 | 플레이어당 서버→클라 ≤ 25 KB/s 가설 | — | 네트워크 로그 |
| 프레임 | — | PC 60fps, 모바일 30fps 이상 가설 | MicroProfiler |

> 수치는 모두 **플랫폼 제약이 아니라 게임 내 튜닝 목표**다. 공식 플랫폼 한도와 혼동하지 않는다(PART K §7).

## 10. 보안 체크리스트 (알파 필수)

1. 모든 Remote 입력 검증(타입, 범위, 컨텍스트, 빈도).
2. 재화·XP·기체 소유·업그레이드·수리 완료는 **서버에서만** 상태 변경.
3. 탑승 상태, 에너지, 쿨다운, 탄약은 서버 기준 값만 사용.
4. 텔레포트 데이터에 민감 정보 금지.
5. 허니팟 Remote 1~2개.
6. 로그: 검증 실패 횟수, 텔레포트 실패, 저장 실패를 요약 기록.
7. 클라이언트 스크립트에 숨긴 수치(밸런스·확률)를 신뢰하지 않음. 확률 판정은 서버.

## 11. 서버 종료·재접속

- 종료 시 §5의 순서 준수.
- 재접속: 세션 잠금이 남아 있으면 **만료까지 대기 또는 강제 인수 금지**. 만료 시간을 60초 등으로 짧게 두고, 같은 서버 재접속은 잠금 재획득.
- 월드 슬롯 스냅샷이 있으면 새 서버가 로드. 없으면 초기 상태로 시작하고 로그 기록.
- 알파에서 **진행 중 전투 상태(투사체, Legion 위치 등)는 저장하지 않는다**. 재접속 시 전투는 초기화되고 계정 데이터만 유지.

## 12. 기존 결정과의 정합성 점검

| 항목 | 기존 결정 | 본 아키텍처 | 판정 |
|---|---|---|---|
| 서버별 월드 독립 | 채택 | 슬롯 단위 독립 | KEEP |
| 계정/월드 데이터 분리 | 채택 | Account / WorldSlot 분리 | KEEP |
| 서버 재시작 시 월드 복구 | 아키텍처에서 확정 | 알파는 최소 스냅샷만 | CHANGE (C-12) |
| 메인 화면 공통 진입 | 채택 | Lobby Place | KEEP |
| 텔레포트 | 미정 | 서버 측 호출, 실패 규칙 | NEW |

## 부록 A. 확인한 Roblox 공식 문서 (확인일 2026-11-20)

- Network ownership, movement validation, and physics — create.roblox.com (네트워크 소유권과 이동 검증 문서)
- Network ownership — create.roblox.com (BasePart 소유권과 `SetNetworkOwner`)
- Client-server model / Onboarding for developers — https://create.roblox.com/docs/en-us/projects/client-server.md
- Securing the client-server boundary — https://create.roblox.com/docs/en-us/scripting/security/client-server-boundary.md
- Remote events and callbacks — create.roblox.com (신뢰성·순서·큐 제한)
- Server-side detection and consequencing — create.roblox.com (허니팟)
- Security and cheat mitigation tactics — create.roblox.com
- Instance streaming — create.roblox.com (범위, Stream in/out, ModelStreamingMode)
- Streaming techniques — https://create.roblox.com/docs/en-us/workspace/streaming/techniques.md
- Teleport between places / TeleportService / TeleportResult / TeleportState — create.roblox.com
- DataStoreService / DataModel:BindToClose — create.roblox.com
- Implement player data and purchasing systems — create.roblox.com
- Best practices for data stores — create.roblox.com
- Data store error codes and limits — create.roblox.com (쓰기 4 MB/분, 읽기 25 MB/분, 큐 30)
- Memory stores / Memory store sorted map / Extended Services — create.roblox.com
- Mobile input / Adaptive design guidelines / Wireframe your layouts — create.roblox.com

> 참고: 검색 결과에서 페이지 제목만 확인된 항목은 `(create.roblox.com)` 도메인과 제목으로 표기했습니다. 실제 URL은 알파 착수 전 재확인 필요.

## 부록 B. 검증 필요 목록 (아키텍처)

- V-A1 P-MOVE 검증 샘플링 주기와 보정 방식의 체감 품질
- V-A2 Feldress 파트 구성 비용(제약 수 대비 프레임 시간)
- V-A3 DataStore 실제 쿼터 여유(8인 기준 실측)
- V-A4 Reserved Server 방식 지원 여부와 제약
- V-A5 세션 잠금 구현 방식(자체 구현 vs 검증된 외부 모듈 및 라이선스)
- V-A6 StreamingEnabled에서 기체 Atomic 모델의 실제 스트리밍 비용