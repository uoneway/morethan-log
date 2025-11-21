# jung-geun님의 morethan-log 확장 기능 분석

<https://github.com/morethanmin/morethan-log> 버전에 비해
<https://github.com/jung-geun/morethan-log> 에서 추가된 주요 기능들에 대한 사용법 가이드입니다.

### 커밋 분석 결과

  총 26개 커밋을 분석한 결과를 KEEP (유지), REMOVE (제거), PARTIAL (부분 유지) 로 분류합니다.

  ---
  REMOVE (제거 권장) - 개인정보/광고 관련

  | 커밋      | 내용                                 | 제거 이유                 |
  |---------|------------------------------------|-----------------------|
  | b9cc817 | site.config.js 업데이트                | 개인 블로그 설정 (이름, 프로필 등) |
  | 729e254 | Ads.txt 추가, Instagram 핸들 업데이트      | 개인 광고 및 SNS 정보        |
  | 4c21366 | Ads.txt → ads.txt 이름 변경            | 광고 관련                 |
  | 4b214c5 | Google Ads 스크립트 추가                 | 광고 코드                 |
  | 4e36c48 | ads.txt sitemap 추가                 | 광고 관련                 |
  | ccdde58 | Google Analytics 활성화 (site.config) | 개인 GA ID              |
  | 84a7837 | README 업데이트                        | 개인 프로젝트 문서 (선택적)      |

  ---
  KEEP (유지 권장) - 기능적 업데이트

  | 커밋      | 내용                             | 유지 이유                       |
  |---------|--------------------------------|-----------------------------|
  | d94428d | @vercel/analytics 통합           | Vercel 분석 기능 (유용)           |
  | 6244a62 | blog scheme 'system' 변경        | 다크모드 시스템 설정 연동              |
  | a17f3fa | GA 컴포넌트, record map 최적화        | 부분 유지: 최적화 코드만              |
  | 2bf925e | API 에러 핸들링/재시도 로직 개선           | 안정성 향상                      |
  | 1444e54 | getPostBySlug 함수 구현            | 핵심 기능 개선                    |
  | a651e97 | usePostsQuery 개선               | 데이터 핸들링 개선                  |
  | 8caa318 | Notion 공식 API 클라이언트 통합, 캐싱     | 핵심 기능 업그레이드                 |
  | b1b3294 | Noto Color Emoji 제거            | 스타일링 개선                     |
  | fc95f9f | 포스트 필터링/정규화 개선                 | API 통합 개선                   |
  | 0d48d1d | 디버그 로깅 유틸리티                    | 개발 편의성                      |
  | 3e43cac | 이미지 핸들링/캐싱 메커니즘 개선             | 중요: Notion 이미지 URL 만료 문제 해결 |
  | 0beb2d3 | 이미지 프록시 URL 처리 개선              | 이미지 안정성                     |
  | 6b18e95 | SpeedInsights 컴포넌트 추가          | 성능 분석 (선택적)                 |
  | 93b8301 | @vercel/speed-insights 패키지     | 성능 분석                       |
  | 9ab347e | GitHub Actions 워크플로우, Jest 테스트 | 매우 유용: CI/CD 및 테스트 인프라      |
  | d74b4d5 | 코드 구조 정리, 중복 제거                | 코드 품질                       |
  | 7bcff57 | KaTeX 수학 렌더링 기능                | 유용: 수식 지원                   |
  | 61c45cd | Notion 블록 요약 기능                | 기능 추가                       |
  | 3a14870 | 이미지 프록시 개선, API 테스트            | 안정성 및 테스트                   |
  | 6b781da | Notion API 색상 지원               | 기능 확장                       |
  | 1be7341 | Notion API 통합 테스트              | 테스트 코드                      |
  | 489c48e | customMapImageUrl 개선           | 이미지 처리 개선                   |

  ---
  권장 작업 순서

  1. site.config.js: 직접 수정하여 본인 정보로 교체
  2. ads.txt 삭제: public/ads.txt 파일 삭제
  3. Google Ads 코드 제거: MetaConfig 컴포넌트에서 광고 스크립트 제거
  4. GA ID 설정: 본인의 Google Analytics ID로 교체하거나 비활성화

---

## 기능 분석

### 1. 환경 변수 설정

#### 필수 환경 변수

`.env.local` 파일을 생성하고 다음 값들을 설정하세요:

```env
# Notion Integration Token (필수)
# https://www.notion.so/my-integrations 에서 생성
NOTION_TOKEN=ntn_xxxxxxxxxxxxxxxxxxxx

# Notion Database ID (필수)
# UUID 형식 (하이픈 포함)
NOTION_DATASOURCE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Revalidation 시크릿 토큰 (선택)
# API를 통한 캐시 갱신에 사용
TOKEN_FOR_REVALIDATE=your-secret-token

# ISR Revalidation 주기 설정 (선택, 기본값: 6시간)
REVALIDATE_HOURS=6

# Google Analytics (선택)
NEXT_PUBLIC_GOOGLE_MEASUREMENT_ID=G-XXXXXXXXXX

# Utterances 댓글 (선택)
NEXT_PUBLIC_UTTERANCES_REPO=username/repo
```

---

### 2. 이미지 프록시 API

#### 문제 해결

Notion의 이미지 URL은 AWS S3 presigned URL로, 일정 시간 후 만료됩니다.
이미지 프록시는 이 문제를 해결합니다.

#### 작동 방식

- `/api/image-proxy?url=<notion-image-url>` 엔드포인트 제공
- Notion 이미지를 프록시하여 캐싱 (1년)
- 만료된 URL 자동 갱신 시도
- 실패 시 placeholder SVG 반환

#### 자동 적용

코드베이스 내에서 Notion 이미지 URL은 자동으로 프록시 URL로 변환됩니다.
별도 설정 불필요.

#### 디버깅

에러 발생 시 로그 파일 생성:

- `image-proxy-errors.jsonl` - 프록시 실패 로그

Slack 알림 설정 (선택):

```env
SLACK_WEBHOOK=https://hooks.slack.com/services/xxx/yyy/zzz
```

---

### 3. Revalidation API

#### 용도

Notion 데이터베이스 업데이트 후 사이트 캐시를 강제 갱신합니다.

#### 사용법

**전체 페이지 갱신:**

```
GET https://your-site.com/api/revalidate?secret=<TOKEN_FOR_REVALIDATE>
```

**특정 페이지 갱신:**

```
GET https://your-site.com/api/revalidate?secret=<TOKEN_FOR_REVALIDATE>&path=/your-post-slug
```

#### GitHub Actions 자동화

`.github/workflows/revalidate.yml`이 12시간마다 자동으로 revalidation 실행합니다.

**필요한 GitHub Secrets:**

- `REVALIDATE_URL`: 사이트 URL (예: `https://your-blog.com`)
- `REVALIDATE_SECRET`: `TOKEN_FOR_REVALIDATE` 값
- `DISCORD_WEBHOOK` (선택): 성공/실패 알림용

---

### 4. GitHub Actions CI/CD

#### 테스트 워크플로우 (`.github/workflows/test.yml`)

**트리거:**

- `main`, `develop` 브랜치 push/PR

**실행 내용:**

1. ESLint 검사
2. TypeScript 타입 체크
3. Jest 테스트 실행
4. 빌드 검증

**로컬에서 실행:**

```bash
npm run lint          # ESLint
npm run type-check    # TypeScript 검사
npm run test          # Jest 테스트
npm run test:coverage # 커버리지 리포트
```

---

### 5. KaTeX 수학 렌더링

#### 용도

Notion의 수학 수식 블록을 렌더링합니다.

#### 사용법

Notion에서 `/equation` 블록 또는 인라인 수식을 사용하면 자동으로 렌더링됩니다.

**예시:**

- 인라인: `$E = mc^2$`
- 블록: 수식 블록 생성

별도 설정 불필요, 자동 적용됩니다.

---

### 6. Vercel Analytics & Speed Insights

#### 용도

- **Analytics**: 페이지뷰, 방문자 분석
- **Speed Insights**: Core Web Vitals 성능 측정

#### 설정

Vercel에 배포 시 자동 활성화됩니다.
Vercel 대시보드에서 Analytics 탭 확인.

로컬 개발 환경에서는 비활성화됩니다.

---

### 8. Notion 공식 API 통합

#### 개선 사항

- `@notionhq/client` 공식 SDK 사용
- 향상된 에러 핸들링 및 재시도 로직
- API 응답 캐싱
- 포스트 필터링 및 정규화 개선

#### 설정

`NOTION_TOKEN` 환경 변수만 설정하면 자동 적용됩니다.

---

### 9. 디버그 API

#### 개발용 엔드포인트

```
GET /api/debug/inspect-slug?slug=<post-slug>
```

특정 포스트의 캐싱 상태와 메타데이터를 확인할 수 있습니다.

**주의:** 프로덕션에서는 비활성화하거나 인증 추가를 권장합니다.

---

### 설정 체크리스트

- [ ] `.env.local` 파일 생성
- [ ] `NOTION_TOKEN` 설정
- [ ] `NOTION_DATASOURCE_ID` 설정
- [ ] `site.config.js` 개인정보 수정
- [ ] (선택) `TOKEN_FOR_REVALIDATE` 설정
- [ ] (선택) GitHub Secrets 설정 (CI/CD용)
- [ ] (선택) Google Analytics ID 설정
- [ ] (선택) Utterances repo 설정

---

### 제거 권장 항목

원본 fork에서 개인정보/광고 관련 코드를 제거해야 합니다:

1. `public/ads.txt` - 삭제
2. `src/components/GoogleAnalytics/index.tsx` - GA ID 교체 또는 삭제
3. `site.config.js` - 개인정보 교체
4. MetaConfig의 Google Ads 스크립트 - 제거

자세한 내용은 커밋 분석 결과를 참조하세요.
