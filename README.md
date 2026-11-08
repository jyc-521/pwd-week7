# Ajou Campus Foodmap API

아주대학교 주변 맛집 정보를 제공하는 Ajou Campus Foodmap의 백엔드 API 서버입니다.

맛집 목록, 상세 정보, 인기 맛집을 조회하고 새 맛집을 등록할 수 있습니다. 맛집 데이터는 Supabase의 PostgreSQL 데이터베이스에 저장합니다. 이전 버전은 데이터를 서버 메모리에 보관했기 때문에 서버가 재시작되면 새로 등록한 맛집이 사라졌습니다. 이번 버전은 데이터베이스에 저장하므로 서버를 재시작하거나 다시 배포해도 데이터가 그대로 남아 있습니다.

실전웹서비스개발 7주차 과제로 진행했습니다.

## 주요 기능

1. 맛집 전체 목록을 조회합니다.
2. ID로 맛집 상세 정보를 조회합니다. 없는 ID를 요청하면 404를 반환합니다.
3. 평점이 높은 순서로 인기 맛집을 조회합니다.
4. 새 맛집을 등록합니다. ID는 데이터베이스가 자동으로 생성합니다.

## 동작 구조

클라이언트가 Express 서버에 요청을 보내면, 서버는 Supabase JS SDK를 통해 Supabase Data API에 요청하고 PostgreSQL의 `restaurants` 테이블에서 데이터를 읽거나 저장합니다.

```text
React 클라이언트 → Express (/api/restaurants) → Supabase JS SDK → Data API → PostgreSQL
```

제보와 좋아요 기능은 기존과 같이 브라우저의 LocalStorage를 사용합니다. 이미지는 파일을 저장하지 않고 URL 문자열만 데이터베이스에 저장합니다.

## 배포 주소

| 구분 | 주소 |
| --- | --- |
| 서버 GitHub | https://github.com/jyc-521/pwd-week7 |
| 클라이언트 GitHub | https://github.com/jyc-521/pwd-week7-client |
| Render API | https://pwd-week6-jyc-521.onrender.com |
| Vercel 서비스 | https://pwd-week5-brown.vercel.app |

## 사용 기술

서버는 Node.js 24.x, TypeScript, Express로 작성했습니다. 데이터베이스는 Supabase(PostgreSQL)를 사용하고 `@supabase/supabase-js`로 연결합니다. 테스트에는 Jest, Supertest, PGlite를 사용했습니다. 서버는 Render에, 클라이언트는 Vercel에 배포합니다.

## 데이터베이스 설계

맛집 정보는 `restaurants` 테이블 하나에 저장합니다. 테이블 정의와 권한 설정은 `supabase/schema.sql`에 있고, 처음 데이터 3건은 `supabase/seed.sql`로 입력합니다.

| 열 | 자료형 | 제약조건 |
| --- | --- | --- |
| id | bigint | 기본 키, 자동 생성 |
| name | text | NOT NULL |
| category | text | NOT NULL |
| location | text | NOT NULL |
| price_range | text | NOT NULL, 기본값 '정보 없음' |
| rating | numeric | NOT NULL, 기본값 0 |
| description | text | NOT NULL, 기본값 '' |
| recommended_menu | text[] | NOT NULL, 기본값 빈 배열 |
| likes | integer | NOT NULL, 기본값 0 |
| image | text | NOT NULL, 기본값 '' |

`id`는 기본 키이므로 중복이나 NULL을 허용하지 않고, 값을 넣지 않으면 데이터베이스가 번호를 자동으로 붙입니다. 추천 메뉴는 여러 개이므로 문자열 배열(`text[]`)로 저장합니다.

테이블에는 RLS를 적용하고 `anon`, `authenticated` 역할의 권한을 회수했습니다. 따라서 브라우저에서 데이터베이스에 직접 접근할 수 없고, 서버용 Secret key를 가진 Express 서버만 데이터베이스에 접근합니다.

데이터베이스의 열 이름은 snake_case이고 API 응답은 camelCase입니다. 두 형식의 변환은 `src/models/restaurant.model.ts`에서 처리합니다.

## 프로젝트 구조

```text
pwd-week7/
├── server.ts
├── src/
│   ├── app.ts
│   ├── config/supabase.ts
│   ├── controllers/restaurants.controller.ts
│   ├── services/restaurants.service.ts
│   ├── models/restaurant.model.ts
│   ├── routes/restaurants.routes.ts
│   ├── types/
│   ├── middleware/
│   ├── utils/
│   └── data/restaurants.json
├── supabase/
│   ├── schema.sql
│   └── seed.sql
├── scripts/
│   ├── check-db.ts
│   └── copy-data.ts
└── tests/
```

## API

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| GET | /health | 서버 상태 확인 |
| GET | /api/restaurants | 맛집 목록 조회 |
| GET | /api/restaurants/popular?limit=N | 인기 맛집 조회 (기본 5개) |
| GET | /api/restaurants/:id | 맛집 상세 조회 |
| POST | /api/restaurants | 맛집 등록 |
| GET | /api/restaurants/sync-demo | 원본 JSON 비교용 데모 |
| POST | /api/restaurants/reset-demo | 원본 3건으로 초기화 |

성공 응답은 `{ "data": ... }` 형식이고, 오류 응답은 `{ "error": { "message": ... } }` 형식입니다. 등록에 성공하면 201을, 없는 맛집을 조회하면 404를 반환합니다. `reset-demo`를 호출하면 추가로 등록한 맛집이 모두 삭제됩니다.

## 실행 방법

먼저 `.env.example`을 복사해 `.env` 파일을 만들고, 본인의 Supabase 프로젝트 URL과 Secret key를 입력합니다. `.env` 파일은 Git에 올리지 않습니다.

```bash
cp -n .env.example .env
npm install --include=dev
npm run dev
```

`.env`에는 다음 값을 입력합니다.

| 변수 | 값 |
| --- | --- |
| PORT | 3000 |
| NODE_ENV | development |
| SUPABASE_URL | Supabase 프로젝트 기본 URL |
| SUPABASE_SECRET_KEY | Supabase 서버용 Secret key |

그 밖의 명령은 다음과 같습니다.

| 명령 | 설명 |
| --- | --- |
| npm run typecheck | 타입 검사 |
| npm test | 테스트 실행 |
| npm run build | 배포용 빌드 |
| npm run db:check | Supabase 연결 확인 |
| npm start | 빌드한 서버 실행 |

## 확인 결과

타입 검사, 테스트(Jest 28건, SQL 6건), 빌드를 모두 통과했습니다.

`POST /api/restaurants`로 새 맛집을 등록한 뒤 서버를 완전히 종료하고 다시 실행했을 때, 등록한 맛집(id 4)이 그대로 조회되는 것을 확인했습니다.

기존 React 클라이언트를 로컬 서버와 연결했을 때 데이터베이스의 맛집 4건이 화면에 표시되는 것을 확인했습니다.

Render에 배포한 뒤 공개 주소에서도 같은 4건과 id 4 맛집이 조회되었고, Vercel에 배포된 클라이언트에서도 Render API를 통해 같은 데이터가 표시되는 것을 확인했습니다.
