# Sketch — 일정+강제성 자산관리 앱

## 전제 결정 (phase1 하이브리드 권고 반영)

- **데이터 수집**: 오픈뱅킹 이체/조회 API + Android SMS 파싱(`READ_SMS`) + 수동 입력 보완. iOS는 마이데이터·SMS 모두 제약이므로 **수동 입력 + 카드사 앱 연계 안내** 중심.
- **강제성 3층**:
  1. **사전 격리** — 자동이체로 저축·격리 계좌에 선송금 (Qapital 스타일)
  2. **사후 페널티** — 목표 실패 시 사전 동의한 금액을 지정 기부처/계좌로 이체 (Beeminder 스타일)
  3. **행동 마찰** — 쇼핑·배달 앱을 예산 초과 다음날 차단·지연 (Freedom/One Sec 스타일, Android Accessibility)
- **플랫폼**: Android 우선 / iOS 축소판(차단·SMS 없음, 수동+격리+페널티만).
- **일정 축**: 예산 이벤트(월급일·카드결제일·저축 목표 마감일)를 캘린더로 시각화 — "일정+자산" 결합이 본 앱의 고유 가치.
- **개발 규모**: 1~2인(안선민/심보승) 전제. **마이데이터 사업자 미등록** → 마이데이터 API 사용하지 않음. 오픈뱅킹 이체 API(핀테크 사업자 등록 선) + SMS/수동으로 우회.
- **Canva/Aura 시너지 없음**: standalone 모바일 앱. phase7 dispatcher에서 **internal fallback** 또는 신규 destination 등록.

## 데이터 모델 초안 (standalone Prisma)

> 참고: Aura-board(padlet/) 스키마와 무관. standalone 스키마.
> ER 관계: `User` 1:N 거의 모든 모델. `Transaction`은 `Account`·`Category`·`DailyBudget`에 연결. `EnforcementEvent`는 `SavingRule`·`PenaltyPledge`·`BlockedAppRule` 중 하나를 출처로 가짐.

```prisma
model User {
  id                String   @id @default(cuid())
  email             String?  @unique
  phone             String?  @unique
  platform          String   // "android" | "ios"
  createdAt         DateTime @default(now())
  // relations: accounts, budgets, rules, pledges, events, ...
}

model ScheduleEvent {
  id        String   @id @default(cuid())
  userId    String
  type      String   // "salary" | "card_due" | "goal_deadline" | "user"
  title     String
  date      DateTime
  amount    Int?     // 재무 이벤트일 때 금액(KRW)
  recurring String?  // "monthly" | "weekly" | null
}

model Account {
  id           String  @id @default(cuid())
  userId       String
  kind         String  // "checking" | "savings" | "freeze" | "card"
  provider     String  // "openbanking:kb" | "manual" | "toss" ...
  externalId   String? // 오픈뱅킹 fintech_use_num
  nickname     String
  isFrozen     Boolean @default(false)
}

model Transaction {
  id         String   @id @default(cuid())
  userId     String
  accountId  String?
  amount     Int      // +수입/-지출 KRW
  occurredAt DateTime
  merchant   String?
  source     String   // "sms" | "openbanking" | "manual" | "ocr"
  categoryId String?
  dailyBudgetId String?
}

model Category {
  id     String @id @default(cuid())
  userId String
  name   String
  color  String
  icon   String?
}

model MonthlyBudget {
  id              String @id @default(cuid())
  userId          String
  yearMonth       String // "2026-04"
  savingGoal      Int    // 월 저축 목표(KRW)
  totalBudget     Int    // 월 총 지출 상한
  status          String // "active" | "achieved" | "failed"
}

model DailyBudget {
  id          String   @id @default(cuid())
  userId      String
  monthlyId   String
  date        DateTime
  budgetAmount Int     // 오늘의 예산(전날 초과 시 삭감됨)
  spentAmount  Int     @default(0)
  overflow     Int     @default(0) // 초과액, 다음날로 이월 삭감
}

model SavingRule {
  id         String @id @default(cuid())
  userId     String
  trigger    Json   // {type:"daily", time:"23:50"} | {type:"on_income"} | {type:"on_spend", category:"coffee"}
  action     Json   // {type:"transfer", amount:10000, toAccountId:"..."}
  isActive   Boolean @default(true)
}

model PenaltyPledge {
  id            String @id @default(cuid())
  userId        String
  condition     Json   // {type:"monthly_goal_fail", monthlyId:"..."}
  penaltyAmount Int    // KRW
  targetType    String // "charity" | "third_party"
  targetRef     String // 기부처 id 또는 계좌 마스킹
  consentedAt   DateTime
  revokedAt     DateTime?
}

model AssetFreeze {
  id            String @id @default(cuid())
  userId        String
  accountId     String // kind="freeze"
  lockedAmount  Int
  unlockAt      DateTime? // 시간 잠금
  unlockCondition Json?   // {type:"goal_achieved"} 등
}

model BlockedAppRule {
  id         String @id @default(cuid())
  userId     String
  appPackage String // "com.coupang.mobile"
  mode       String // "block" | "delay_10s" | "warn"
  triggerOn  String // "daily_budget_exceeded" | "always" | "weekend"
}

model EnforcementEvent {
  id         String   @id @default(cuid())
  userId     String
  kind       String   // "freeze_transfer" | "penalty_charged" | "app_blocked" | "warning_shown"
  sourceType String   // "SavingRule" | "PenaltyPledge" | "BlockedAppRule"
  sourceId   String
  amount     Int?
  occurredAt DateTime @default(now())
  meta       Json?
}
```

## 사용자 흐름

### 페르소나 A — "소비통제가 절실한" 메인 유저 (20~30대, 안선민이 말한 타깃)

1. **온보딩**: 월 저축 목표(예: 100만), 월 총예산, 일일 예산 자동 산출 확인. 페널티 약정(목표 실패 시 5만원 기부 자동이체) 동의. 오픈뱅킹 연동(주계좌·저축계좌·카드).
2. **일상 추적**: Android SMS 파싱으로 카드 지출이 `Transaction`에 자동 유입 → 오늘의 `DailyBudget.spentAmount` 실시간 갱신. iOS는 수동·OCR.
3. **당일 초과**: 일일 예산 초과 시 푸시 경고("오늘 2.3만 초과") + 다음날 `DailyBudget.budgetAmount` 자동 삭감.
4. **사전 격리**: 매일 23:50 `SavingRule` 트리거로 일일 예산 잔여액을 격리 계좌(`AssetFreeze`)로 자동이체.
5. **월말 실패**: 저축목표 미달 시 `PenaltyPledge` 실행 → 기부처 자동이체, `EnforcementEvent` 기록.
6. **행동 마찰**: 예산 초과 다음날은 쿠팡·배민 앱 실행 시 10초 지연 + "오늘은 예산 초과일" 경고 (`BlockedAppRule`).
7. **일정 탭**: 월급일·카드결제일·목표 마감일이 `ScheduleEvent`로 달력에 표시 → 현금흐름 시각화.

### 페르소나 B — "일정 위주" 라이트 유저

1. 기본 **캘린더** 용도로 진입. `ScheduleEvent` 수동 등록. 강제성·예산 기능은 기본 Off.
2. 수동으로 지출 몇 건 기록 → 앱이 "예산 기능을 켜면 자동 격리 추천" 프롬프트 제시 (opt-in).
3. 사용자가 자발적으로 강제성 1층(격리)만 활성화. 페널티·앱차단은 여전히 Off. 단계적 온보딩.

## 모바일 성능·안정성 체크

- [ ] 앱 cold start < 1.5s (Android 중저가, Galaxy A 시리즈 기준)
- [ ] SMS 파싱 백그라운드 작업 하루 평균 배터리 소모 < 2% (WorkManager + SMS BroadcastReceiver, 파싱은 on-arrival 이벤트 기반)
- [ ] 오픈뱅킹 API 호출 하루 상한 관리 (기관별 quota, 조회는 1시간 1회 캐시)
- [ ] 오프라인에서도 수동 입력·캘린더 열람 가능 → 온라인 복귀 시 sync (SQLite 로컬 + WorkManager sync job)
- [ ] 금융 데이터 로컬 암호화 — AES-256, Android Keystore / iOS Keychain. DB는 SQLCipher.
- [ ] 민감정보(계좌번호·카드번호 전체) 서버 저장 금지 — 서버는 사용자 id·익명 트랜잭션 메타만.
- [ ] Accessibility 서비스(앱 차단용) 사용자 재동의 주기 관리 — Android가 주기적으로 권한 해제 가능.
- [ ] 페널티 자동이체 실행 시 사용자 푸시 사전 고지 (법적/UX 리스크 완화, 24시간 취소 window).

## 강제성 3층 조합 모듈 매핑

| 층 | 메커니즘 | 구현 모듈 (Prisma) | 의존 외부 |
|---|---|---|---|
| 1 사전 격리 | 자동이체로 저축계좌 선송금 | `SavingRule` + `AssetFreeze` + `EnforcementEvent` | 오픈뱅킹 이체 API (핀테크 사업자 등록) |
| 2 사후 페널티 | 목표 실패 시 지정 이체 | `PenaltyPledge` + `EnforcementEvent` | 오픈뱅킹 이체 API + 기부처 수취 계약(또는 제3자 계좌) |
| 3 행동 마찰 | 쇼핑·배달 앱 차단·지연 | `BlockedAppRule` + `EnforcementEvent` | Android Accessibility Service / iOS Screen Time Family Controls |

## 알려진 리스크

| 리스크 | 영향 | 완화 |
|---|---|---|
| 마이데이터 미등록 → 국내 조회 커버리지 낮음 | 초기 UX가 토스·뱅크샐러드보다 불편 | SMS+수동 하이브리드, iOS는 카드앱 화면 안내, 주요 카드사 3~5개만 파서 우선 지원 |
| Play Store `READ_SMS` 권한 거절 | Android 핵심 자동화 기능 불가 | 가계부/금융 카테고리 포지셔닝, 심사용 데모 영상·privacy statement 준비, fallback으로 사용자 수동 SMS 복붙·알림창 읽기 권한(`NotificationListener`) 대체 경로 |
| 페널티 자동이체의 법적 리스크(전자금융거래법상 위임결제) | 서비스 중단·제재 | 사전 동의 약정+유저 24h 해지권 명시, 우선 **기부처 수취** 구조로 법률 자문 부담 최소화, 3자 계좌 송금은 v2로 연기 |
| iOS Screen Time API 제약 | 차단 기능 Android 대비 약함 | iOS는 "설정 가이드+경고" 수준으로 축소, 핵심 가치는 격리+페널티로 커버. Family Controls 엔터프라이즈 프로파일은 MVP 범위 외 |
| 사용자 이탈(강제성 피로) | LTV 하락, 리텐션 저하 | 강제성 난이도 단계 선택(1층→2층→3층 opt-in), 성공 시 시각 보상(streak·그래프)과 해제 조건 명확화 |
| 보안 사고 시 신뢰 붕괴 | 앱 폐업 수준 치명타 | 로컬 우선 저장, 서버 민감정보 최소화, 외부 보안 감사(최소 1회) 후 정식 런칭 |
| 오픈뱅킹 이체 수수료/일일 한도 | 일일 격리 빈도 제한 | 일일 단위가 아닌 주/월 격리 default, 사용자가 원할 때만 일일 모드 활성화 |
| 공동 개발자 2인의 역할 과중 | 런칭 지연 | MVP는 1층(격리)+SMS 파싱+캘린더만, 2·3층은 v1.1·v1.2로 분할 |

## 미결 질문 (phase3 인터뷰 재료)

1. **강제성 MVP 범위**: 3층 모두 MVP인가, 1층(사전 격리)만으로 먼저 런칭 후 2·3층을 v1.1·v1.2로 확장인가? *(phase1에서도 핵심 trade-off로 남음)*
2. **페널티 이체 대상**: 지정 기부처만 허용(법적 안전) vs 사용자가 지정한 제3자 계좌도 허용(UX 자유도) — 어느 쪽을 MVP에 채택? *(phase1 미결 질문 재등장 — 법적 리스크)*
3. **iOS 우선순위**: iOS 축소판을 MVP에 포함 vs Android 단독 런칭 후 iOS v2 — 시장 커버리지 vs 개발비 trade-off?
4. **SMS 파싱 실패 사용자 대비책**: Play Store가 `READ_SMS` 승인 거절 시 `NotificationListener` 대체 경로를 MVP에 함께 구현할 것인가, 런칭 후 핫픽스로 미룰 것인가? *(phase1 미결 질문 확장)*
5. **일정 탭의 깊이**: 단순 사용자 등록 이벤트만 vs 재무 이벤트(월급일·카드결제일·목표 마감일) 자동 주입까지 — MVP 범위? "일정+자산"의 고유 가치를 만드는 지점이므로 결정 필요.
6. **오픈뱅킹 일일 격리 빈도**: 일일 격리 vs 주/월 단위 — 오픈뱅킹 이체 API 수수료·한도 실측 필요. MVP default는 어느 쪽? *(phase1 미결 질문 재등장)*
7. **수익 모델**: (a) 무료 + 페널티 수수료 1~3% (b) 구독 월 3~5천원 (c) 무료 + 제휴 금융상품 추천 — 어느 가정으로 phase3 이후 설계할 것인가? Beeminder형 (a)가 강제성 철학과 가장 일치하나 한국 유저 수용도는 불명.

## Canva/Aura 시너지

본 아이디어는 Aura-board·padlet과 **무관한 standalone 모바일 앱**. 시너지 없음.

→ phase7 dispatcher 처리 후보:
- **internal fallback** (`destinations/internal/`) — 당장 외부 프로젝트 INBOX 없음
- **신규 destination 등록** — 실제 착수 결정 시 새 프로젝트 폴더 생성 후 INBOX/ 배송
