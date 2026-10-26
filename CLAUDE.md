# CMS서초영재관 문자시트 — 프로젝트 구조 문서

Claude Code가 이 프로젝트를 이해하고 이어서 작업할 수 있도록 작성된 문서입니다.

---

## 1. 프로젝트 개요

- **앱 이름**: CMS서초영재관 문자시트
- **URL**: https://dooocsaaa76-creator.github.io/bloomath/
- **GitHub**: https://github.com/dooocsaaa76-creator/bloomath
- **배포 방식**: GitHub Pages (단일 HTML 파일)
- **구현**: 단일 파일 `index.html` (약 3650+ 줄), 외부 JS/CSS 없음 (Firebase SDK 제외)

---

## 2. Firebase 구성

```js
apiKey: "AIzaSyB5kXJJ4pQtU5i6sqE3RXOUu2DiHRg87WA"
authDomain: "bloomath-message.firebaseapp.com"
projectId: "bloomath-message"
appId: "1:968918734392:web:2bc5c4df537eaaf64bb8cd"
```

- **Firebase SDK**: v10.12.2 (모듈식, gstatic CDN에서 로드)
- **플랜**: Spark (무료) — PITR(특정 시점 복구) 불가
- **컬렉션**: `bloomath`
  - 문서 `pl`: PL(개별진도반) 전체 데이터
  - 문서 `schedule`: 일반 문자 발송 스케줄 데이터 (별도)

---

## 3. PL 데이터 모델 (Firestore `bloomath/pl`)

```js
{
  terms: [
    {
      id: "uuid",
      name: "2026년 9월분기 (영재관)",
      venue: "영재관",   // "영재관" | "프리미엄관"
      classes: [
        { id: "uuid", name: "월1부", day: "월", time: "3:00~6:00", teacher: "홍길동", plType: "elementary" }
        // plType: "elementary"(초등PL) | "secondary"(중등PL) | undefined(자동판별)
      ]
    }
  ],
  classes: {
    "<classId>": {
      students: [{ id: "<studentId>", name: "홍길동" }],
      sessions: {},
      currentDate: ""
    }
  },
  students: {
    "<studentId>": {
      name: "홍길동",
      school: "서초초등학교",
      grade: "초5",
      history: [],
      sentLog: {}
    }
  }
}
```

---

## 4. 핵심 규칙: 학생 ID 공유

**같은 이름의 학생은 반드시 하나의 studentId를 공유**한다.

- 학생이 여러 반에 배정되어도 `students` 컬렉션에는 1개 레코드만 존재
- `classes[classId].students`에는 `{ id: 공유ID, name: "이름" }` 형태로 저장
- 문자 발송 이력(`history`, `sentLog`)이 모든 반에서 연동됨
- 새 학생 추가 시 `nameToId` 맵 패턴 사용:

```js
const nameToId = {};
Object.entries(pl_db.students).forEach(([id, s]) => {
  if (s.name) nameToId[s.name] = id;
});
// 이름이 이미 있으면 기존 ID 재사용, 없으면 새 ID 생성
if (!nameToId[name]) {
  const newId = uid();
  nameToId[name] = newId;
  pl_db.students[newId] = { name, school, grade, history: [], sentLog: {} };
}
```

---

## 5. 관(Venue) 구조

### 영재관
- **초등PL**: 월~금 1부/2부 (3:00~6:00, 6:00~9:00)
- **중등PL & 주말단과**: 수A, 수B, 금, 토1~3부, 일
- 학생 명단 테이블이 초등/중등 2개로 분리 표시

### 프리미엄관
- 단일 PL 테이블 (월~금 1부/2부)

---

## 6. 초등PL / 중등PL 자동 판별 로직 (`isSecondary`)

```js
const isSecondary = cls =>
  cls.plType === 'secondary' ||
  (cls.plType !== 'elementary' && (
    ['토','일'].includes(cls.day) ||
    (cls.day === '수' && (cls.name === '수A' || cls.name === '수B' || cls.time.startsWith('5'))) ||
    (cls.day === '금' && cls.time.startsWith('7'))
  ));
```

- `plType` 필드가 있으면 우선 적용 (반추가 시 명시적 선택)
- 없으면 요일/시간/이름 조합으로 자동 판별

---

## 7. 기본 반 구성 (`PL_DEFAULT_CLASSES`, line ~2157)

```js
const PL_DEFAULT_CLASSES = {
  '영재관': {
    elementary: [/* 월~금 1부/2부 */],
    secondary: [/* 수A/수B/금/토1~3부/일 */]
  },
  '프리미엄관': [/* 월~금 1부/2부 */]
};
```

분기 생성 시 자동으로 기본 반이 추가됨.

---

## 8. 페이지 라우팅 (SPA 방식)

```js
view = { page: 'pl-home' | 'pl-terms' | 'pl-class' | 'pl-roster' | 'pl-student', plTermId, plClassId, plStudentId }
```

`render()` 함수가 `view.page`에 따라 해당 렌더 함수 호출.

---

## 9. 주요 전역 함수

| 함수 | 설명 |
|------|------|
| `savePLDB()` | Firestore `bloomath/pl`에 `pl_db` 전체 저장 |
| `uid()` | 랜덤 단축 ID 생성 |
| `getPLTerm(id)` | `pl_db.terms`에서 분기 찾기 |
| `getPLClassData(id)` | `pl_db.classes[id]` 반환 |
| `sortClassesByTime(cls)` | 요일/시간 순 정렬 |
| `buildPLRosterTable(termId, term, classes, title)` | 학생 명단 HTML 테이블 생성 |
| `window.createPLTerm()` | 새 분기 생성 |
| `window.addPLClass(termId)` | 반 추가 (plType 저장 포함) |
| `window.editPLClass(termId, classId)` | 반 정보 수정 모달 |
| `window.deletePLClass(termId, classId)` | 반 삭제 |
| `window.copyPLStudents()` | 이전 분기 학생 복사 |
| `window.plGo(page, opts)` | 페이지 이동 |

---

## 10. UI 컴포넌트 패턴

- **Toast**: `showToast(msg, 'success'|'error')`
- **확인 다이얼로그**: `showConfirm(msg, callback)`
- **모달**: `document.getElementById('modalTitle')`, `document.getElementById('modalContent')` 직접 조작 후 `openModal()` / `closeModal()`

---

## 11. 반추가 UI (line ~2253)

영재관 분기에서는 **구분 드롭다운** 표시:
```html
<select id="plNewClassType">
  <option value="elementary">초등PL</option>
  <option value="secondary">중등PL</option>
</select>
```
프리미엄관에서는 표시 안 됨 (단일 유형).

---

## 12. 진행 중인 작업 이력

| 날짜 | 작업 |
|------|------|
| 2026-10 | Firebase 데이터 손실 발생 (영재관 classes 필드 대부분 소실) |
| 2026-10 | PL명단 CSV + 배치생 엑셀로 복구 도구(`restore_pl_v2.html`) 제작 |
| 2026-10 | 학생 ID 공유 방식 확인 및 name→ID 맵 패턴 확립 |
| 2026-10 | 반추가 UI에 초등PL/중등PL 선택 드롭다운 추가 |

---

## 13. 복구 도구

- **`/mnt/user-data/outputs/restore_pl_v2.html`**: 브라우저에서 열어 Firebase에 영재관 2026년 9월분기 PL 데이터 복원
  - "미리보기" → 변경사항 확인
  - "복원시작" → 실제 저장
  - "기존영재관 classes 정리후복원" → 중복 데이터 정리 후 복원

---

## 14. 개발 시 주의사항

1. **단일 HTML 파일**: `index.html` 하나에 모든 HTML/CSS/JS 포함
2. **GitHub Pages 배포**: `main` 브랜치 push 시 자동 배포
3. **Firebase 직접 호출**: SDK import는 gstatic CDN URL 사용 (npm 번들 없음)
4. **학생 ID 공유 필수**: 새 학생 추가 시 항상 `nameToId` 맵 확인 후 처리
5. **`pl_db` 전역 객체**: Firestore에서 로드한 데이터를 메모리에 유지, 수정 후 `savePLDB()` 호출
6. **반 plType 필드**: 2026-10 이후 추가된 필드. 기존 반들은 `undefined`이므로 자동판별 로직이 여전히 필요
