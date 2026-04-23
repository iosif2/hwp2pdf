# Architecture

**프로젝트**: hwp2pdf · **생성일**: 2026-04-23 · **수정일**: 2026-04-23

## 전체 구조

```
┌─────────────────────────────────────────────────────┐
│                   API Server (Axum)                 │
│                                                     │
│  POST /convert          GET /files/{file_id}.pdf    │
│       │                          │                  │
│  ┌────▼──────────┐      ┌────────▼───────┐          │
│  │ ConvertHandler│      │  FileHandler   │          │
│  └────┬──────────┘      └────────┬───────┘          │
│       │                          │                  │
│  ┌────▼──────────┐      ┌────────▼───────┐          │
│  │ ConvertService│      │  Local Storage │          │
│  └────┬──────────┘      └────────────────┘          │
│       │                                             │
│  ┌────▼──────────┐                                  │
│  │  rhwp Engine  │                                  │
│  └───────────────┘                                  │
└─────────────────────────────────────────────────────┘
```

## 레이어 역할

| 레이어 | 책임 |
|--------|------|
| Handler | HTTP 요청/응답, 입력 검증, 에러 직렬화 |
| Service | 변환 흐름 조율, 임시 파일 생명주기 관리 |
| rhwp Engine | HWP/HWPX 파싱 및 PDF 생성 |
| Local Storage | 변환 결과 PDF 저장 및 제공 |

## 변환 파이프라인

```
HWP/HWPX bytes
    │
    ▼
HwpDocument::from_bytes()   ← 바이너리(HWP 5.0) 또는 XML(HWPX) 파싱
    │
    ▼
Document Model              ← 문단 / 표 / 이미지 / 수식 등 구조화
    │
    ▼
render_to_svg_pages()       ← 페이지별 SVG 렌더링 (폰트 의존)
    │
    ▼
svg_to_pdf()                ← SVG 병합 → PDF 생성
    │
    ▼
output.pdf
```

> rhwp의 PDF 출력 경로는 아직 안정화 단계가 아님. 일부 문서에서 레이아웃 깨짐 발생 가능.

## API

### `POST /convert`

```
Content-Type: multipart/form-data
Body: file=<hwp|hwpx>
```

성공 응답:
```json
{ "status": "success", "file_id": "<uuid>", "download_url": "/files/<uuid>.pdf" }
```

실패 응답:
```json
{ "status": "error", "message": "<사유>" }
```

### `GET /files/{file_id}.pdf`

저장된 PDF 파일 스트리밍 반환.

## 폰트

PDF 렌더링 품질은 폰트 환경에 직접 의존한다.
rhwp가 참조하는 경로에 아래 폰트가 반드시 존재해야 한다:

```
/app/fonts/
    NotoSansKR-Regular.ttf
    NotoSerifKR-Regular.ttf
```

폰트 경로가 다르면 렌더링 결과가 환경마다 달라진다.

## 처리 방식 (초기)

- **동기 처리**: 요청당 하나의 변환, 완료 후 응답
- **단일 노드**: 워커 큐 없음, 분산 없음
- **임시 파일 최소화**: 가능하면 메모리 기반 처리

## 주요 에러 유형

| 에러 | 원인 | 처리 |
|------|------|------|
| 파싱 실패 | 손상된 파일 또는 미지원 포맷 | 400 반환 |
| 렌더링 실패 | rhwp 내부 오류, 복잡한 레이아웃 | 500 반환 |
| 폰트 로딩 실패 | 폰트 경로 누락 | 500 반환 + 로그 |
| 변환 시간 초과 | 대용량 문서 | timeout 설정 필요 |
