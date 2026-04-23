# hwp2pdf 프로젝트 개요서 (rhwp 기반)

## 1. 프로젝트 개요

본 프로젝트는 Rust 기반 오픈소스 엔진인 rhwp를 활용하여  
HWP/HWPX 문서를 PDF로 변환하는 고속 변환 API 파이프라인을 구축하는 것을 목표로 한다.

초기 단계에서는 비동기 처리, 대규모 분산 처리보다는  
**단일 노드에서 안정적으로 동작하는 고속 변환 API 구현**에 집중한다.

---

## 2. 목표 (Objective)

### 핵심 목표

- rhwp 기반 HWP/HWPX → PDF 변환 기능 구현
- HTTP API 인터페이스 제공
- 단일 요청 기준 빠른 변환 처리 (Low latency)
- 내부 서비스에서 바로 호출 가능한 변환 엔드포인트 제공

### 비목표 (Out of Scope - 초기 단계)

- 대규모 비동기 워커 시스템
- 분산 처리 / 큐 기반 아키텍처
- 완벽한 렌더링 정합성 보장
- OCR 기반 fallback

---

## 3. 기술 스택

- Language: Rust
- Web Framework: Axum (or Actix)
- Conversion Engine: rhwp
- Runtime: Tokio
- Storage (초기): Local filesystem

---

## 4. rhwp 분석

### 역할

rhwp는 HWP/HWPX 문서를 파싱하고 내부 Document Model로 변환한 뒤  
렌더링 파이프라인을 통해 SVG 또는 PDF로 출력하는 엔진이다.

### 내부 처리 흐름

```
HWP/HWPX File
    ↓
Parser (Binary / XML)
    ↓
Document Model (Structured Representation)
    ↓
Rendering Pipeline
    ↓
SVG (per page)
    ↓
PDF (SVG → PDF 변환)
```

### 특징

- Rust 기반 → 높은 성능 및 안정성
- HWP 5.0 및 HWPX 지원
- 문단 / 표 / 텍스트박스 / 이미지 / 수식 렌더링 지원
- PDF 생성은 SVG 기반 렌더링 후 병합 방식

### 제약 사항

- PDF 출력 기능은 공식적으로 안정화 단계가 아님
- 폰트 의존성이 높음 (환경에 따라 렌더링 차이 발생)
- 일부 문서에서 레이아웃 깨짐 가능성 존재

---

## 5. 시스템 아키텍처 (초기 버전)

### 구조

```
[ Client ]
    ↓
[ API Server (Rust) ]
    ↓
[ rhwp Conversion Engine ]
    ↓
[ Local Storage ]
```

### 흐름

1. 클라이언트가 HWP/HWPX 파일 업로드
2. API 서버에서 파일 수신 및 임시 저장
3. rhwp를 호출하여 PDF 변환 수행
4. 결과 PDF 저장
5. 변환 결과 반환

---

## 6. API 설계

### 6.1 파일 업로드 및 변환

```
POST /convert
Content-Type: multipart/form-data
```

#### Request

- file: HWP 또는 HWPX 파일

#### Response (성공)

```json
{
  "status": "success",
  "file_id": "uuid",
  "download_url": "/files/{file_id}.pdf"
}
```

#### Response (실패)

```json
{
  "status": "error",
  "message": "conversion failed"
}
```

---

### 6.2 파일 다운로드

```
GET /files/{file_id}.pdf
```

---

## 7. 변환 처리 방식

### 초기 구현 전략

- rhwp CLI 또는 내부 함수 호출
- 동기 처리 방식 (request-response)
- 단일 요청당 하나의 변환 수행

### 변환 명령 흐름 (예시)

```
rhwp export-pdf input.hwp -o output.pdf
```

또는 내부 호출:

```
HwpDocument::from_bytes(...)
→ render_to_svg_pages()
→ svg_to_pdf()
```

---

## 8. 성능 전략

### 1. 프로세스 단순화

- 불필요한 중간 포맷 제거
- 직접 PDF 생성 경로 사용

### 2. I/O 최소화

- 가능하면 메모리 기반 처리
- 임시 파일 최소화

### 3. 병목 요소

- SVG 렌더링 비용
- 폰트 로딩
- 대용량 문서 처리

---

## 9. 폰트 전략 (중요)

PDF 품질은 폰트에 크게 의존함.

### 대응 전략

- 컨테이너에 한글 폰트 포함
- Noto Sans KR / Noto Serif KR 기본 사용
- rhwp가 참조하는 경로에 폰트 배치

예시:

```
/app/fonts/
    NotoSansKR-Regular.ttf
    NotoSerifKR-Regular.ttf
```

---

## 10. 에러 처리

### 주요 에러 유형

- 파싱 실패
- 렌더링 실패
- 폰트 로딩 실패
- 파일 포맷 오류

### 응답 정책

- 명확한 에러 메시지 반환
- 내부 로그 기록

---

## 11. 확장 계획 (Future Work)

- 비동기 워커 기반 처리
- Job Queue 도입
- 대용량 파일 처리 개선
- PDF 품질 개선
- Markdown/JSON 직접 추출 경로 추가
- 임베딩 파이프라인 연계

---

## 12. 리스크 및 대응

| 리스크 | 설명 | 대응 |
|------|------|------|
| 렌더링 불완전 | 일부 문서 깨짐 | fallback 전략 |
| 폰트 문제 | 환경별 결과 차이 | 폰트 고정 |
| 성능 편차 | 문서별 처리시간 차이 | timeout 설정 |
| rhwp 변경 | upstream 변경 | wrapper 구조 유지 |

---

## 13. 결론

본 프로젝트는 rhwp를 활용하여  
HWP/HWPX 문서를 PDF로 변환하는 API 기반 파이프라인을 구축하는 것을 목표로 한다.

초기 단계에서는 단순하고 빠르게 동작하는 변환 API를 구현하고,  
이후 점진적으로 비동기 처리 및 확장 가능한 구조로 발전시킨다.

핵심 전략은 다음과 같다:

- 단순한 구조로 빠르게 구현
- rhwp를 변환 엔진으로 활용
- PDF 변환을 안정적으로 수행
- 이후 확장 가능한 구조로 진화
