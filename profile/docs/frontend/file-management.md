# 파일 관리 상세

<details>
<summary><b>목차</b></summary>

- [파일 타입 정의](#파일-타입-정의)
  - [서버 응답 타입](#서버-응답-타입)
  - [UI 상태 타입](#ui-상태-타입)
  - [파일 제약 조건](#파일-제약-조건)
- [파일 업로드](#파일-업로드)
  - [전체 흐름](#전체-흐름)
  - [클라이언트 검증](#클라이언트-검증)
  - [배치 업로드 전략](#배치-업로드-전략)
  - [업로드 프로그레스 추적](#업로드-프로그레스-추적)
- [파일 다운로드](#파일-다운로드)
- [파일 삭제](#파일-삭제)
  - [삭제 후 캐시 무효화 (3단 갱신)](#삭제-후-캐시-무효화-3단-갱신)
- [FileManagementPage UI 로직](#filemanagementpage-ui-로직)
  - [파일 상태 관리](#파일-상태-관리)
  - [주요 핸들러](#주요-핸들러)
  - [유틸리티 함수](#유틸리티-함수)

</details>

---

## 파일 타입 정의

[file.ts](../../src/types/file.ts)

### 서버 응답 타입

| 인터페이스 | 필드 | 설명 |
|---|---|---|
| `FileResponse` | `id`, `memoId`, `fileName`, `minioObjectName`, `fileHash`, `fileSize`, `contentType`, `createdAt` | 서버에서 반환하는 파일 메타데이터 |
| `FileListResponse` | `files: FileResponse[]`, `totalCount` | 파일 목록 래퍼 |
| `DownloadUrlResponse` | `downloadUrl: string` | Pre-signed 다운로드 URL |

### UI 상태 타입

```typescript
interface FileItem {
  id: string;              // 클라이언트 생성 고유 ID
  name: string;            // 파일명
  size: number;            // 파일 크기 (bytes)
  type?: string;           // MIME 타입
  uploadProgress?: number; // 업로드 진행률 (0~100)
  isUploading?: boolean;   // 업로드 중 여부
  error?: string;          // 에러 메시지
  backendId?: string;      // 서버 File ID
  minioObjectName?: string; // MinIO 오브젝트 키
}
```

`FileItem`은 업로드 전·중·후·실패 상태를 하나의 타입으로 표현한다. `backendId`가 없으면 서버에 아직 저장되지 않은 로컬 상태를 의미한다.

### 파일 제약 조건

[file.ts](../../src/types/file.ts)

| 상수 | 값 |
|---|---|
| `MAX_FILE_SIZE` | 500MB |
| `ALLOWED_EXTENSIONS` | txt, doc, docx, pdf, xls, xlsx, ppt, pptx, jpg, jpeg, png, gif, zip, rar |

---

## 파일 업로드

### 전체 흐름

```mermaid
sequenceDiagram
    participant User
    participant FMP as FileManagementPage
    participant Hook as useUploadFile
    participant FS as file.ts
    participant AX as Axios Client
    participant BE as Backend
    participant MinIO

    User->>FMP: 파일 선택 (드래그 또는 클릭)
    FMP->>FMP: handleFileUpload(files[])
    
    loop 각 파일에 대해
        FMP->>FMP: validateFile(file)
        alt 검증 실패
            FMP->>FMP: toast.error() 표시
        else 검증 성공
            FMP->>FMP: FileItem 생성 (isUploading: true)
        end
    end

    alt 파일 1개 & 업로드 중 아님
        FMP->>FMP: executeUpload(newItems) 직접 호출
    else 파일 2개 이상
        FMP->>FMP: setPendingQueue에 추가
        Note over FMP: 사용자가 '업로드 시작' 버튼 클릭 시 startUpload()
    end
    
    loop 각 유효 파일에 대해 (순차)
        FMP->>Hook: uploadMutation.mutateAsync({memoId, file, onProgress})
        Hook->>FS: uploadFile(memoId, file, onProgress)
        FS->>FS: FormData 생성
        FS->>AX: POST /file/memo/{memoId} (multipart)
        
        Note over AX: onUploadProgress 콜백
        AX-->>FMP: onProgress(percent) → UI 진행률 갱신
        
        AX->>BE: multipart 전송
        BE->>BE: 해시 계산, 중복 검사
        BE->>MinIO: putObject (신규 파일만)
        BE-->>AX: FileResponse
        AX-->>FMP: FileResponse
        
        FMP->>FMP: 성공한 FileItem 로컬 상태에서 제거
    end

    FMP->>FMP: 배치 완료
    FMP->>FMP: queryClient.invalidateQueries<br/>(fileKeys.byMemo + memoKeys.detail + memoKeys.lists)
```

### 클라이언트 검증

[FileManagementPage.tsx](../../src/pages/FileManagementPage.tsx)

```mermaid
flowchart TD
    A["validateFile(file)"] --> B{"확장자 허용?\n(ALLOWED_EXTENSIONS)"}
    B -->|No| C["'허용되지 않는 파일 형식' 에러"]
    B -->|Yes| D{"파일 크기 초과?\n(> MAX_FILE_SIZE)"}
    D -->|Yes| E["'파일 크기 제한 초과' 에러"]
    D -->|No| F{"이미 추가된 파일?\n(name + size 중복)"}
    F -->|Yes| G["'이미 추가된 파일' 에러"]
    F -->|No| H["null (검증 통과)"]
```

검증 실패 시 해당 파일에 대한 `toast.error()`만 표시하고, 나머지 유효한 파일은 계속 처리한다.

### 배치 업로드 전략

```mermaid
flowchart TD
    A["handleFileUpload(selectedFiles)"] --> B["각 파일에 대해\nFileItem 생성 + 검증"]
    B --> C{"파일 수?"}
    C -->|"1개 & 업로드 중 아님"| D["executeUpload(newItems) 직접 호출"]
    C -->|"2개 이상"| E["setPendingQueue에 추가"]
    E --> F["사용자: '업로드 시작' 클릭"]
    F --> G["startUpload() → executeUpload(pendingQueue)"]
    D & G --> H["for...of 순차 실행"]
    H --> I["각 파일 업로드 완료/실패 후\n성공 항목은 로컬 상태에서 제거"]
    I --> J["전체 배치 완료"]
    J --> K["queryClient.invalidateQueries\n(한 번에 캐시 무효화)"]
```

> [!IMPORTANT]
> 파일 업로드는 **순차 실행**된다. 파일 업로드가 끝날 때마다 무효화하면 배치 도중 파일 목록이 반복 refetch되기 때문에 `useUploadFile`의 `onSuccess`에는 캐시 무효화를 두지 않는다. 캐시 무효화는 `executeUpload`가 전체 배치를 완료한 뒤 한 번에 수행된다.

### 업로드 프로그레스 추적

```mermaid
flowchart LR
    A["Axios onUploadProgress"] --> B["progressEvent.loaded / progressEvent.total"]
    B --> C["percent = Math.round(loaded * 100 / total)"]
    C --> D["onProgress(percent)"]
    D --> E["FileItem.uploadProgress = percent"]
    E --> F["UI 프로그레스 바 갱신"]
```

---

## 파일 다운로드

[useFileQueries.ts](../../src/hooks/api/useFileQueries.ts)

`downloadFile` 함수는 Hook이 아닌 **독립 함수**로 구현되어 있다. 다운로드는 캐싱이 필요 없는 일회성 동작이기 때문이다.

```mermaid
sequenceDiagram
    participant User
    participant FMP as FileManagementPage
    participant DL as downloadFile()
    participant FS as file.ts
    participant AX as Axios Client
    participant BE as Backend
    participant MinIO

    User->>FMP: 다운로드 버튼 클릭
    FMP->>DL: downloadFile(fileId, filename)
    DL->>FS: getDownloadUrl(fileId, filename)
    FS->>AX: GET /file/{fileId}/download-url?filename={name}
    AX->>BE: 요청 (JWT 포함)
    BE->>MinIO: generatePresignedDownloadUrl<br/>(30분 만료)
    MinIO-->>BE: Pre-signed URL
    BE-->>AX: { downloadUrl: "https://..." }
    AX-->>FS: DownloadUrlResponse
    FS-->>DL: "https://..." (URL 문자열)
    DL->>DL: window.open(downloadUrl, "_blank")
    Note over DL: 새 탭에서 브라우저 네이티브 다운로드 시작
```

| 단계 | 설명 |
|---|---|
| Pre-signed URL 요청 | 백엔드가 MinIO에서 30분 유효 Pre-signed URL 생성 |
| 브라우저 다운로드 | `window.open`으로 새 탭에서 URL 오픈 → `Content-Disposition`에 의해 자동 다운로드 |

---

## 파일 삭제

```mermaid
sequenceDiagram
    participant User
    participant FMP as FileManagementPage
    participant Hook as useDeleteFile
    participant FS as file.ts
    participant BE as Backend
    participant MinIO

    User->>FMP: 삭제 버튼 클릭
    FMP->>Hook: deleteFileMutation.mutate({fileId, memoId})
    Hook->>FS: deleteFile(fileId)
    FS->>BE: DELETE /file/{fileId}
    
    BE->>BE: File 조회 + 소유권 검증
    BE->>BE: FileMetadata.referenceCount--
    
    alt referenceCount == 0
        BE->>MinIO: removeObject (물리 삭제)
        BE->>BE: FileMetadata DB 삭제
    end
    
    BE->>BE: File 엔티티 DB 삭제
    BE-->>FS: 200 OK

    Note over Hook: onSuccess 캐시 무효화
    Hook->>Hook: invalidateQueries(fileKeys.byMemo)
    Hook->>Hook: invalidateQueries(memoKeys.detail)
    Hook->>Hook: resetQueries(memoKeys.lists)
```

### 삭제 후 캐시 무효화 (3단 갱신)

| 대상 | 전략 | 이유 |
|---|---|---|
| `fileKeys.byMemo(memoId)` | `invalidateQueries` | 해당 메모의 파일 목록 갱신 |
| `memoKeys.detail(memoId)` | `invalidateQueries` | 메모 상세의 `fileCount` 갱신 |
| `memoKeys.lists()` | `resetQueries` | 메모 목록 카드의 파일 아이콘 갱신 |

업로드와 달리 삭제는 단건 요청이므로, `useDeleteFile`의 `onSuccess`에서 즉시 무효화한다.

---

## FileManagementPage UI 로직

[FileManagementPage.tsx](../../src/pages/FileManagementPage.tsx)

### 파일 상태 관리

```mermaid
stateDiagram-v2
    [*] --> 대기중 : FileItem 생성

    대기중 --> 업로드중 : startUpload() / executeUpload()
    대기중 --> [*] : validateFile() 실패 (toast.error, FileItem 미생성)

    업로드중 --> 업로드중 : onProgress(percent)
    업로드중 --> 완료 : 업로드 성공 (로컬 상태에서 제거 후 서버 목록 반영)
    업로드중 --> 에러 : 업로드 실패

    에러 --> [*] : 에러 메시지 표시
```

### 주요 핸들러

| 핸들러 | 동작 |
|---|---|
| `handleFileSelect` | `<input type="file">` onChange → `handleFileUpload`에 파일 배열 전달 |
| `handleFileUpload` | 파일 검증 → FileItem 생성 → 1개면 `executeUpload` 직접 호출, 2개 이상이면 `pendingQueue`에 추가 |
| `startUpload` | `pendingQueue` 전체를 `executeUpload`에 전달 (업로드 시작 버튼) |
| `executeUpload` | 순차 업로드 → 완료 후 캐시 무효화 |
| `handleFileDelete` | `backendId` 존재 시 `deleteFileMutation.mutate`, 없으면 로컬 상태 및 `pendingQueue`에서 제거 |
| `handleClearAll` | 업로드 중(`isUploading`)이면 실행하지 않음. `backendId`가 있는 파일(서버에 저장된 파일)만 `deleteFileMutation` 호출 |

### 유틸리티 함수

| 함수 | 동작 |
|---|---|
| `formatFileSize(bytes)` | `bytes` → 사람이 읽을 수 있는 형식 (`B`, `KB`, `MB`) |
| `getFileIcon(fileName)` | 확장자 기반 아이콘 컴포넌트 반환 (phosphor-icons) |
| `validateFile(file)` | 확장자·크기·중복 검증 → 에러 메시지 또는 `null` |
