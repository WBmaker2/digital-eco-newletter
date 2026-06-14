# Firebase 학급 게시판 구현 계획

## 목표

실제 교실에서 여러 학생이 각자 기기로 작성한 환경 뉴스가 같은 학급 게시판에 모이도록, 현재 브라우저 `localStorage` 저장 방식을 Firebase Firestore 기반 공유 저장소로 확장한다.

## 현재 한계

- `localStorage`는 각 브라우저/기기 안에만 저장된다.
- 학생 A가 발행한 뉴스는 학생 B나 교사용 PC에서 자동으로 보이지 않는다.
- 브라우저 데이터 삭제, 시크릿 모드, 기기 변경 시 게시글이 사라질 수 있다.

## 권장 구조

- 정적 호스팅: 기존 GitHub Pages 유지
- 백엔드 저장소: Firebase Firestore
- Firebase 프로젝트: `digital-eco-newletter-wbmaker2`
- 공유 단위: 학급 코드
- 컬렉션 경로: `classes/{classCode}/reports`
- 실시간 조회: Firestore `onSnapshot()` 구독
- 발행: Firestore `addDoc()`
- 삭제: Firebase Auth 교사 로그인 후 등록된 교사 UID만 허용
- fallback: Firebase 설정이 비어 있거나 연결 실패 시 기존 `localStorage` 모드로 동작

## 데이터 모델

```json
{
  "themeId": "acid-rain",
  "themeTitle": "산성비와 숲 파괴",
  "themeShort": "산성비",
  "themeColor": "green",
  "headline": "산성비에 약해진 숲, 우리가 지켜요",
  "reporter": "5학년 김초록 기자",
  "body": "피해 상황과 기후 행동 다짐",
  "imageUrl": "data:image/svg+xml...",
  "caption": "산성비로 약해진 숲을 살펴보는 현장 사진",
  "classCode": "green-earth-class",
  "createdAt": "serverTimestamp 또는 ISO 문자열",
  "updatedAt": "serverTimestamp 또는 ISO 문자열"
}
```

## 구현 단계

1. `index.html`에 Firebase 설정 영역 추가
   - `window.ECO_NEWS_FIREBASE_CONFIG` 값이 채워지면 Firestore 모드
   - 설정이 비어 있으면 localStorage 데모 모드
2. 학급 코드 UI 추가
   - 학생과 교사가 같은 학급 코드를 입력하면 같은 게시판을 조회
   - 기본값은 `green-earth-class`
3. 저장소 계층 분리
   - `createLocalBoardStore()`
   - `createFirestoreBoardStore()`
   - 앱 나머지 코드는 동일한 `boardStore.subscribe/add/delete` 인터페이스 사용
4. 게시판 실시간 동기화
   - Firestore 모드: `onSnapshot(query(... orderBy("createdAt", "desc")))`
   - localStorage 모드: 즉시 렌더링 + 같은 브라우저 저장
5. 교사용 삭제 흐름
   - 교사용 관리 영역에서 Firebase Auth 이메일/비밀번호 로그인
   - 로그인한 사용자의 UID가 `firestore.rules`의 교사 UID 목록과 일치할 때만 삭제 버튼 표시
   - Firestore Rules도 같은 UID 조건으로 삭제를 강제해 클라이언트 우회 삭제를 차단
6. 검증
   - Firebase 설정 없음: 기존 localStorage 발행/조회/삭제 유지
   - Firebase 설정 있음: 두 브라우저/기기에서 같은 학급 코드로 실시간 동기화 확인

## Firebase 콘솔 설정 메모

1. Firebase 프로젝트 생성
2. Web 앱 등록 후 `firebaseConfig` 확보
3. Firestore Database 생성
4. 운영 전 Security Rules 적용
5. `index.html`의 `window.ECO_NEWS_FIREBASE_CONFIG`에 설정값 입력

## 적용 Rules 요약

현재 `firestore.rules`는 게시글 생성 시 필드와 글자 수를 검사하고, 읽기는 수업 게시판 공유를 위해 허용한다. 삭제는 Firebase Auth로 로그인한 교사 UID만 허용한다.

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isTeacher() {
      return request.auth != null &&
        request.auth.uid in ["교사_UID"];
    }

    match /classes/{classCode}/reports/{reportId} {
      allow read: if true;
      allow create: if
        request.resource.data.headline is string &&
        request.resource.data.reporter is string &&
        request.resource.data.body is string &&
        request.resource.data.classCode == classCode;
      allow update: if false;
      allow delete: if isTeacher();
    }
  }
}
```

교사 계정을 추가하려면 다음 중 하나가 필요하다.

- Firebase Auth에 교사 계정 생성 후 `firestore.rules`의 UID 목록에 추가
- 장기 운영 시 Cloud Functions 또는 커스텀 클레임으로 교사 권한 관리 자동화

## 참고한 공식 문서

- Firebase Web SDK 설정: https://firebase.google.com/docs/web/setup
- Firestore 실시간 업데이트: https://firebase.google.com/docs/firestore/query-data/listen
- Firestore 데이터 추가: https://firebase.google.com/docs/firestore/manage-data/add-data
