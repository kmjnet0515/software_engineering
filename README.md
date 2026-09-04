# Ollert

칸반 보드 기반의 팀 협업 툴입니다. 프로젝트를 만들어 팀원을 초대하고, 카드로 할 일을 관리하며, 같은 프로젝트에 접속한 팀원 화면에 변경 사항이 실시간으로 반영됩니다.

2025년 1학기 소프트웨어공학 수업 팀 프로젝트(15조)로 개발했습니다. AWS EC2에 프론트엔드와 백엔드를 배포해 운영했습니다.

---

## 기술 스택

**프론트엔드**
- Next.js 15 (App Router) / React 19 / TypeScript
- Tailwind CSS 4
- NextAuth.js — Google, Kakao 소셜 로그인
- Socket.IO Client — 실시간 동기화
- Recharts — 요약 차트, Framer Motion — 랜딩 페이지 애니메이션

**백엔드**
- Node.js / Express 5 (ESM)
- MySQL (mysql2/promise, 커넥션 풀)
- Socket.IO — 변경 알림 브로드캐스트
- bcryptjs — 비밀번호 해싱
- Nodemailer — 이메일 인증 및 알림 발송
- node-cron — 마감일 알림 스케줄러
- multer-s3 / AWS SDK v3 — 첨부파일 업로드
- OpenAI API — 챗봇

**인프라**
- AWS EC2 (프론트 3000 포트 / 백엔드 5001 포트), AWS S3, MySQL
- 프론트엔드·백엔드 각각 Dockerfile 구성

---

## 주요 기능

### 회원 / 인증
- 이메일 회원가입: 6자리 인증 코드를 메일로 발송해 이메일 소유를 확인한 뒤에만 비밀번호를 설정할 수 있습니다.
- 비밀번호는 bcrypt로 해싱해 저장하고, 8자 이상 + 특수문자 포함 규칙을 프론트·백엔드 양쪽에서 검증합니다.
- Google, Kakao 소셜 로그인 (NextAuth). 소셜 계정은 비밀번호 찾기 대상에서 제외됩니다.
- "내 정보 저장" 선택 시 UUID 토큰을 발급해 1시간 동안 자동 로그인되고, 만료된 토큰은 재로그인 시점에 정리됩니다.
- 인증 코드 재발송을 통한 비밀번호 재설정

### 프로젝트
- 프로젝트 생성 시 `할 일` / `진행 중` / `완료` 컬럼이 자동으로 만들어집니다.
- 생성자는 owner, 초대로 들어온 사람은 member로 구분되며 owner만 로그·역할 관리 메뉴에 접근할 수 있습니다.
- UUID 초대 링크를 생성해 클립보드로 복사합니다. 링크는 1회용이고, 이미 참여 중인 사용자는 중복 가입되지 않습니다.
- 프로젝트 삭제 시 owner는 프로젝트 자체를, member는 자신의 참여 정보만 삭제됩니다.

### 보드
- 컬럼 / 카드 추가·삭제·이름 수정
- HTML5 Drag and Drop API로 카드를 다른 컬럼으로 이동합니다. 라이브러리 없이 `dataTransfer`로 카드 id를 넘기고, 드롭 시점에 컬럼 id를 갱신한 뒤 소켓으로 다른 접속자에게 알립니다.

### 카드 상세 (모달)
- 상세 설명, 담당자, 시작일 / 마감일 지정
- 담당자로 지정되면 해당 사용자에게 안내 메일이 발송됩니다.
- 댓글 작성·수정·삭제 (작성자 본인만 수정·삭제 가능)
- 댓글에 파일 첨부 — S3에 업로드하고 반환된 URL을 댓글에 연결합니다.

### 4가지 프로젝트 뷰
- **요약** — 컬럼별 카드 개수 집계와 파이 차트. 컬럼 개수에 맞춰 HSL 색상을 균등 분할해 생성하므로 컬럼을 추가해도 색이 겹치지 않습니다.
- **타임라인** — 카드의 시작일~마감일을 막대로 표시하는 간트 형태. 좌우 끝에 도달하면 날짜 범위를 30일씩 자동 확장하고, Ctrl + 휠로 하루 칸 너비를 20~80px 범위에서 조절합니다.
- **캘린더** — 월간 달력에 기간 카드를 막대로 표시합니다. 휠로 이전·다음 달 이동, 연·월 텍스트를 클릭해 직접 입력할 수 있습니다.
- **채팅** — 프로젝트별 실시간 채팅. 메시지는 DB에 저장되고 접속 시 이전 대화를 불러옵니다.

### 실시간 동기화
Socket.IO로 메시지 내용을 직접 주고받는 대신, `isChanged` / `isModalChanged` **신호만 브로드캐스트**하고 각 클라이언트가 REST API로 최신 상태를 다시 조회하는 방식을 사용했습니다. 소켓과 DB의 상태가 어긋날 여지를 없애고, 화면마다 필요한 데이터만 가져오도록 하기 위한 선택입니다.

### 마감 알림
`node-cron`으로 매일 자정에 마감 7일 전 / 1일 전인 카드를 조회해 담당자에게 메일을 보냅니다.

### AI 챗봇
화면 우측 하단 버튼으로 열리는 챗봇입니다. "프로젝트 이동" 같은 키워드가 포함되면 해당 페이지로 라우팅하고, 그 외 입력은 OpenAI API(gpt-3.5-turbo)로 전달해 답변합니다.

### 활동 로그
컬럼·카드 추가/삭제, 드래그 이동, 카드 저장, 댓글 변경, 권한 변경을 `logs` 테이블에 기록하고 owner만 조회할 수 있습니다.

---

## 시스템 구성

```
[ 브라우저 ]
     |  REST (fetch)          |  WebSocket
     v                        v
[ Next.js (EC2:3000) ]   [ Express (EC2:5001) ]
     |                        |
     |  NextAuth              +--> MySQL      (사용자 / 프로젝트 / 카드 / 채팅 / 로그)
     +--> Google / Kakao      +--> AWS S3     (댓글 첨부파일)
                              +--> Gmail SMTP (인증코드 / 마감 알림)
                              +--> OpenAI API (챗봇)
```

## 데이터베이스

| 테이블 | 설명 |
| --- | --- |
| `user_info` | 사용자 계정, 비밀번호 해시, 이메일 인증 상태 |
| `projects` | 프로젝트 이름, 설명, 생성자 |
| `project_members` | 프로젝트 참여자와 역할 (owner / member) |
| `column_table` | 보드 컬럼 |
| `card_table` | 카드 (제목, 설명, 담당자, 시작일, 마감일) |
| `comment_table` | 카드 댓글, 첨부파일 URL |
| `chat_messages` | 프로젝트별 채팅 메시지 |
| `logs` | 활동 로그 |
| `invite_tokens` | 1회용 초대 링크 토큰 |
| `login_tokens` | 자동 로그인 토큰 (1시간 만료) |

## API

`backend/server.js`에 REST 엔드포인트 약 50개를 구현했습니다.

| 분류 | 주요 엔드포인트 |
| --- | --- |
| 인증 | `/api/request-verification`, `/api/verify-code`, `/api/signup`, `/api/tryLogin`, `/api/socialLogin`, `/api/changePassword` |
| 자동 로그인 | `/api/createLoginToken`, `/api/tokenLogin`, `/api/deleteLoginToken` |
| 프로젝트 | `/api/createProject`, `/api/showProjects`, `/api/updateProject`, `/api/deleteProject`, `/api/checkOwner` |
| 초대 / 권한 | `/api/createInviteLInk`, `/api/acceptInvite`, `/api/showProjectUsernameRole`, `/api/changeRole` |
| 보드 | `/api/createColumn`, `/api/showColumn`, `/api/deleteColumn`, `/api/createCard`, `/api/showCard`, `/api/dragCard` |
| 카드 상세 | `/api/setCardManager`, `/api/setStartEndDate`, `/api/setCard_desc`, `/api/getDescCardManagerStartEndDate` |
| 댓글 / 파일 | `/api/addComment`, `/api/getComments`, `/api/editComment`, `/api/deleteComment`, `/api/upload` |
| 채팅 / 로그 / 챗봇 | `/api/setChat`, `/api/getChat`, `/api/writeLog`, `/api/getLog`, `/api/analyze` |

---

## 실행 방법

### 백엔드

```bash
cd backend
npm install
npm start          # http://localhost:5001
```

`backend/.env.local`:

```
SERVER_HOST=            # MySQL 호스트
SERVER_USERNAME=
SERVER_PASSWORD=
SERVER_DATABASE_NAME=
EMAIL=                  # 발신용 Gmail 주소
PASSWORD=               # Gmail 앱 비밀번호
OPENAI_API_KEY=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET_NAME=
```

### 프론트엔드

```bash
cd frontend
npm install
npm run dev        # http://localhost:3000
```

`frontend/.env.local`:

```
NEXTAUTH_URL=
NEXTAUTH_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
KAKAO_CLIENT_ID=
KAKAO_CLIENT_SECRET=
```

### Docker

```bash
docker build -t ollert-backend ./backend
docker build -t ollert-frontend ./frontend
```

---

