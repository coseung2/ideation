# 행사 신청 보드 — 설계 노트 (v3, 공개 보드 방식 확정)

> 작성일: 2026-04-12 (v2 대체)
> 관련: `tablet-performance-roadmap.md`, `drawing-board-library-roadmap.md`, `implementation-roadmap.md`
> 동기: 실제 패들렛처럼 **QR만 찍으면 로그인 없이 접속 가능한 공개 보드**. 담당 교사가 설문 개설, 교내 누구나 희망 제출.
> 예시: 버스킹 오디션, 학생회 임원 선거, 체육대회 계주, 캠페인 참가자
> 구조: `Board.layout = "event-signup"` + 공개 접근 모드

---

## 핵심 방향

- **QR은 보드 접속 링크** (로그인 아님)
- 보드는 **완전 공개** — 링크/QR만 있으면 누구나 접근
- 학생 신원은 폼에 직접 입력 (이름·학년·반·학번)
- 담당 교사가 사후 심사·검증
- 기존 Student·Classroom 인증 인프라 **우회**

---

## Board 확장

```prisma
model Board {
  // ... 기존 ...
  layout      String @default("freeform")   // + "event-signup"
  accessMode  String @default("classroom")  // "classroom" | "public-link"
  accessToken String?                        // public-link일 때 URL hash (재발급 가능)

  // event-signup 메타
  eventPosterUrl      String?
  applicationStart    DateTime?
  applicationEnd      DateTime?
  eventStart          DateTime?
  eventEnd            DateTime?
  venue               String?
  maxSelections       Int?
  videoPolicy         String?            // "required" | "optional" | "none"
  videoProviders      String?            // JSON: ["youtube","stream"]
  maxVideoDurationSec Int?
  maxVideoSizeMb      Int?
  allowTeam           Boolean  @default(false)
  maxTeamSize         Int?
  customQuestions     String?  @db.Text  // JSON 폼 스키마
  announceMode        String?            // "public" | "by-name-search"
  requireApproval     Boolean  @default(false) // 교사 승인 대기 상태 사용 여부

  // 기본 신원 필드 사용 여부 (폼에 자동 포함)
  askName             Boolean  @default(true)
  askGradeClass       Boolean  @default(true)
  askStudentNumber    Boolean  @default(true)
  askContact          Boolean  @default(false)
}
```

**접근 경로**: `/b/:slug?t=:accessToken` — 토큰 불일치 시 404. 교사가 "링크 재발급" 누르면 `accessToken` 재생성 → 예전 QR 무효.

## Submission 확장

```prisma
model Submission {
  id          String   @id @default(cuid())
  boardId     String
  userId      String?                 // ⭐ optional — 공개 신청은 null
  // 공개 신청자 정보 (userId null일 때 채워짐)
  applicantName     String?
  applicantGrade    String?           // "6" 같은 학년
  applicantClass    String?           // "3반"
  applicantNumber   String?           // 학번·출석번호
  applicantContact  String?           // 선택
  ipHash            String?           // 중복 신청 throttling용 (해시만 저장)

  // 기존 필드
  content       String @default("")
  linkUrl       String?
  fileUrl       String?
  status        String @default("submitted") // submitted|approved|rejected|waitlist|withdrawn|pending_approval
  feedback      String?
  grade         String?

  // event-signup 전용
  teamName       String?
  teamMembers    String?  @db.Text   // JSON: [{name, grade, class, number}]
  answers        String?  @db.Text   // JSON: customQuestions 응답
  videoUrl       String?
  videoProvider  String?
  videoId        String?
  videoThumbnail String?
  scoreAvg       Float?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  board    Board              @relation(fields: [boardId], references: [id], onDelete: Cascade)
  user     User?              @relation(fields: [userId], references: [id])
  reviews  SubmissionReview[]

  @@index([boardId, status])
  @@index([userId])
  @@index([ipHash])
}

model SubmissionReview {
  id           String @id @default(cuid())
  submissionId String
  reviewerId   String   // 교사/심사위원 User
  score        Int
  comment      String?  @db.Text
  createdAt    DateTime @default(now())

  submission Submission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  reviewer   User       @relation(fields: [reviewerId], references: [id])

  @@unique([submissionId, reviewerId])
  @@index([submissionId])
}
```

---

## 교사 흐름 (행사 개설)

```
1. "새 보드" → 레이아웃 "행사 신청"
2. accessMode = "public-link" 자동 설정 (토글 가능)
3. 기본 정보: 제목·설명·포스터(Canva)
4. 일정 + 선발 인원
5. 신원 필드 체크: 이름·학년·반·학번·연락처 중 필요한 것
6. 영상 정책
7. 팀 허용
8. 폼 빌더 (추가 질문)
9. 공지 방식: 공개 명단 / 본인 이름 검색
10. 교사 승인 대기 옵션
11. 게시 → 링크 + QR 자동 생성
    링크: /b/busking-2026?t=a9f3xk...
    QR: 포스터 삽입용 PNG/SVG
12. 수정·재발급·마감 버튼
```

---

## 학생 흐름 (신청)

```
1. 포스터·단톡·복도 QR 스캔
2. 바로 보드 페이지 로드 (로그인 없음)
3. 행사 상세 (포스터·일정·설명·선발 인원)
4. "신청하기" 클릭
5. 폼 작성
   - 이름·학년·반·학번 (교사 설정에 따라)
   - 커스텀 답변
   - 팀 허용 시 팀원 정보 직접 입력
   - 영상 업로드
6. 제출 → 확인 번호 또는 본인 제출 토큰 발급
7. 본인 제출 토큰으로 나중에 상태 확인·수정 가능 (링크: /b/.../my?mt=mytoken)
```

**확인 토큰 이유**: 로그인 없으니 "내 신청 찾기"를 학생 기기/쿠키에 저장된 토큰으로 연결. 분실 시 이름·학번 입력해 복구 가능한 경로 제공.

---

## 영상 업로드 경로

- **Cloudflare Stream Direct Creator Upload**: 기본 추천. 로그인 없이도 서명 URL만으로 업로드 가능
- **YouTube 비공개 링크**: 학생이 계정 있으면
- **Vercel Blob**: 아주 짧은 영상

서명 URL은 서버에서 1회성 발급 → 남용 방지.

---

## 교사 심사 흐름

기존 v2 설계 유지. 리스트·개별 상세·점수·상태 일괄 변경. 태블릿은 개별 심사만, 일괄은 데스크톱.

"교사 승인 대기" 옵션 켜면:
- 새 제출은 `pending_approval` 상태
- 교사가 "목록에 노출" 체크해야 `submitted`로 전환
- 스팸·장난 제출 자동 숨김 효과

---

## 결과 공지

- **공개 명단**: 보드 상단 합격자 영역
- **본인 이름 검색**: 학생이 본인 이름·학번 입력하면 결과 표시 (명단 노출 없음)
- 본인 확인 토큰 있으면 "내 신청 결과" 페이지로 바로

---

## 악용·스팸 방어

| 리스크 | 대응 |
|---|---|
| QR 유출·외부 접근 | `accessToken` 기반 URL. 재발급 시 예전 QR 무효 |
| 장난 신청 | 교사 승인 대기(`requireApproval=true`). 의심 건 삭제 |
| 같은 기기 반복 신청 | `ipHash` + 쿠키 기반 throttling (1시간 5건 제한 등) |
| 개인정보 유출 | 공개 명단 모드면 이름·학번 노출 명시 동의 체크. 기본값은 본인 검색 모드 |
| 영상 업로드 남용 | 서명 URL 1회성, 크기·길이 상한 강제 |
| 마감 후 제출 | 서버 시각 기준 잠금 |

---

## 공개 vs 학급 전용 — 혼용 시나리오

같은 담당 교사가 두 종류의 보드 운영 가능:
- 학급 공지 보드: `accessMode = "classroom"` (기존)
- 버스킹 오디션 보드: `accessMode = "public-link"` (누구나)

보드 리스트에서 두 모드 시각 구분 (아이콘·뱃지).

---

## 태블릿(갤럭시 탭 S6 Lite) 고려

- 학생 진입: 단일 폼 페이지 — 가벼움
- 영상 업로드: Cloudflare Stream TUS 재개 업로드
- 교사 심사 리스트: 가상화 (100건+ 대비)
- 공개 URL이라 SSR 캐시 활용 가능 (보드 메타만, 개별 제출은 아님)

---

## 작업 분할

| 단계 | 내용 |
|---|---|
| ES-1 | Board.accessMode/accessToken + Submission.userId optional + 공개 신청 필드 |
| ES-2 | 레이아웃 "event-signup" + 행사 개설 폼 |
| ES-3 | 폼 빌더 (커스텀 질문) |
| ES-4 | QR 생성 + 링크 재발급 |
| ES-5 | 공개 신청 폼 (로그인 없이 접근) + 확인 토큰 발급 |
| ES-6 | YouTube 링크 경로 |
| ES-7 | Cloudflare Stream 서명 URL + 업로드 |
| ES-8 | 교사 심사 탭 + 승인 대기 모드 |
| ES-9 | 복수 심사위원 |
| ES-10 | 결과 공지 (공개 명단 / 본인 검색) + 확인 토큰 조회 |
| ES-11 | 스팸 방어 (ipHash throttling) |
| ES-12 | 팀 신청 (폼에서 팀원 정보 입력) |
| ES-13 | (보너스) 관객 투표 |
| ES-14 | (보너스) 타임테이블 |
| ES-15 | (보너스) Canva Autofill 당선자 포스터 |

1차 배포: ES-1 ~ ES-11

---

## 수용 기준 핵심

- [ ] 학생이 QR 스캔 후 로그인 없이 3분 이내 신청 완료
- [ ] 교사가 링크 재발급 시 예전 QR 즉시 무효
- [ ] 갤럭시 탭 S6 Lite에서 영상 업로드(1분) 성공
- [ ] 승인 대기 모드에서 스팸 제출 교사 뷰에만 노출
- [ ] 본인 확인 토큰 분실 시 이름·학번으로 복구 가능
- [ ] 공개 명단 노출 전 신청자 명시 동의

---

## 미결

- 학번·학년 검증 방법 (학교 학적 DB 연동은 나중, 당분간 자기 기입 신뢰)
- 이름 동명이인 식별 (학년·반·학번 조합으로 충분?)
- 공개 명단 모드에서 개인정보보호법 고지 문구
- 초등 저학년 QR 스캔·타자 숙련도 (교사·학부모 도움 허용)
- ~~학부모 대리 신청 케이스 지원 여부~~ — **Seed 7 범위 밖**. 학부모는 `/parent/*`에서 **자녀 본인 Submission만 열람(read-only)**. 대리 신청은 현행 공개 보드 폼을 학부모가 직접 기입(로그인 없이)하는 경로 유지. Seed 7 PWA는 신청 기능 비제공

---

## 파킹

- 카카오 알림톡 결과 공지 (학부모 연결) — Seed 7 v2+ 파킹과 연계
- QR 스캐너 앱 내장 (타 QR 앱 필요 없이)
- 학교 SSO 연동 (Classting 등)
- 연합 행사 여러 학교 공동 공개 보드

---

## 학부모 열람 범위 (Seed 7 통합, 2026-04-12)

본 로드맵의 행사 보드는 공개 접속(QR·토큰) 모델이므로 학부모도 동일 URL로 접근 가능하지만, **인증된 학부모 PWA(`/parent/*`) 경로의 열람 규칙**은 **Seed 7 `seed_37b35654542f`** (`plans/parent-viewer-roadmap.md`)의 **자녀 범위 매트릭스 §5**로 일원화됐다.

| 축 | 값 |
|---|---|
| 엔티티 | `Board(layout="event-signup")` · `Submission` |
| 학부모 학급 이벤트 메타 열람 | 자녀 학급의 이벤트 보드 메타(제목·설명·포스터·일정·장소·선발 인원)는 **노출 허용** |
| 서버 필터 (메타) | `Board.classroomId = child.classroomId` 중 `layout=="event-signup"` AND `accessMode IN ("classroom","public-link")` |
| 학부모 Submission 열람 | **자녀 본인 Submission·피드백만** — 타 학생 Submission 응답에서 제거 |
| 서버 필터 (Submission) | `Submission.userId = child.userId` 또는 `applicantName+grade+class+number` 매칭 → 자녀만 |
| 참가자 명단·득점 | 학부모 뷰에서 **항상 비노출** (공개 명단 모드여도 `/parent/*`에서는 마스킹). SubmissionReview·scoreAvg 필드 제거 |
| 팀 제출 | 팀 멤버 중 자녀가 있으면 해당 팀 Submission만 열람 가능, 팀 다른 멤버의 개인 정보는 마스킹 |
| 공개 보드 QR 경로 | `/parent/*` 외 공개 URL(`/b/:slug?t=...`)은 기존 규칙 유지 (학부모도 공개 접속 가능) |
| 진입점 | `/parent/child/[id]/events` — 자녀 학급 이벤트 카드 리스트 + 본인 신청 상태 |

구현 작업은 Seed 7의 **PV-7 (자녀 범위 서버 필터)** 에서 처리한다. 본 로드맵의 ES-1~ES-11은 공개 보드·교사 심사 로직만 담당하며, `/parent/*` 전용 Submission 필터는 Seed 7 측에서 구현.

---

## Submission 엔티티 공유 — assignment-board (Seed 11, 2026-04-14)

`Submission`은 event-signup(Seed 3)과 assignment-board(Seed 11) **둘이 재사용**한다. v1 충돌 없음:

| 축 | event-signup (Seed 3) | assignment-board (Seed 11) |
|---|---|---|
| `Submission.status` 허용 값 | `submitted` · `approved` · `rejected` · `waitlist` · `withdrawn` · `pending_approval` | **사용하지 않음** (상태는 `AssignmentSlot.submissionStatus`에 분리) |
| 상태 네임스페이스 | `Submission.status` | `AssignmentSlot.submissionStatus` (별도 slot 엔티티) |
| 상태 값 | 위 6종 | `assigned` · `viewed` · `submitted` · `returned` · `reviewed` · `orphaned` |
| 참조 방향 | `Board(layout="event-signup") ← Submission` | `Board(layout="assignment") ← AssignmentSlot.submissionId → Submission` |
| 콘텐츠 필드 공유 | `content` · `linkUrl` · `fileUrl` · `feedback` · `updatedAt` — 공통 재사용 | 동일 |
| `feedback` 필드 | 교사 심사 코멘트 | v1 반려 사유 저장처로도 사용 가능 (`AssignmentSlot.returnReason`과 이중화 허용 — slot은 메타용, Submission은 콘텐츠용) |

**Zod 검증 규칙**: `Board.layout` 기반 분기로 `Submission.status` 허용 enum을 제한 (assignment-board 로드맵 §12 R2 완화). 서버 레벨에서 layout mismatch 시 422.

동일한 `Submission` 테이블을 두 레이아웃이 공유하므로 마이그레이션 시 충돌 없이 점진 확장 가능. `SubmissionHistory` 엔티티 승격은 양 로드맵 공통 v2 파킹.

---

### 변경 로그
| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-12 | `seed_37b35654542f` | §미결 "학부모 대리 신청"을 Seed 7 범위 밖(read-only)으로 해소. §파킹 "카카오 알림톡 학부모 연결"을 Seed 7 v2 파킹과 연계로 표기. §학부모 열람 범위 절 신규 추가. |
| 2026-04-14 | `seed_38c34e91bf28` | §Submission 엔티티 공유 절 신규 추가 — assignment-board(Seed 11)와 Submission 재사용 네임스페이스 분리 합의. status enum 충돌 없음(slot 레벨 필드로 분리). |
