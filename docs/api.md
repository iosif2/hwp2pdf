# API

**프로젝트**: hwp2pdf · **생성일**: 2026-04-23 · **수정일**: 2026-04-23

Base URL: `http://localhost:3000`

---

## `POST /convert`

HWP 또는 HWPX 파일을 업로드하여 PDF로 변환한다.

### Request

```
POST /convert
Content-Type: multipart/form-data
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `file` | binary | ✅ | `.hwp` 또는 `.hwpx` 파일 |

#### curl 예시

```bash
curl -X POST http://localhost:3000/convert \
  -F "file=@document.hwp"
```

### Response

#### 성공 `200 OK`

```json
{
  "status": "success",
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "download_url": "/files/550e8400-e29b-41d4-a716-446655440000.pdf"
}
```

#### 실패 — 잘못된 파일 형식 `400 Bad Request`

```json
{
  "status": "error",
  "message": "unsupported file format: only .hwp and .hwpx are accepted"
}
```

#### 실패 — 파싱 오류 `422 Unprocessable Entity`

```json
{
  "status": "error",
  "message": "failed to parse document: corrupted or invalid HWP structure"
}
```

#### 실패 — 변환 오류 `500 Internal Server Error`

```json
{
  "status": "error",
  "message": "conversion failed: rendering error on page 3"
}
```

---

## `GET /files/{file_id}.pdf`

변환 완료된 PDF 파일을 다운로드한다.

### Request

```
GET /files/550e8400-e29b-41d4-a716-446655440000.pdf
```

#### curl 예시

```bash
curl -O http://localhost:3000/files/550e8400-e29b-41d4-a716-446655440000.pdf
```

### Response

#### 성공 `200 OK`

```
Content-Type: application/pdf
Content-Disposition: attachment; filename="550e8400-e29b-41d4-a716-446655440000.pdf"

<binary PDF data>
```

#### 실패 — 파일 없음 `404 Not Found`

```json
{
  "status": "error",
  "message": "file not found"
}
```

---

## 전체 흐름 예시

```bash
# 1. 변환 요청
$ curl -s -X POST http://localhost:3000/convert \
    -F "file=@report.hwp" | jq
{
  "status": "success",
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "download_url": "/files/550e8400-e29b-41d4-a716-446655440000.pdf"
}

# 2. PDF 다운로드
$ curl -O http://localhost:3000/files/550e8400-e29b-41d4-a716-446655440000.pdf
```

---

## 에러 코드 요약

| HTTP 상태 | 원인 |
|-----------|------|
| `400` | 파일 누락, 미지원 확장자 |
| `422` | 파일 파싱 실패 (손상 또는 비정상 구조) |
| `500` | rhwp 렌더링 실패, 폰트 로딩 실패 등 내부 오류 |
| `404` | 존재하지 않는 `file_id` |
