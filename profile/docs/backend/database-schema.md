# 데이터베이스 설계 및 스키마

<details>
<summary><b>목차</b></summary>

- [테이블 명세](#테이블-명세)
  - [Memo (메모)](#memo-메모)
  - [Tag (태그)](#tag-태그)
  - [File_Metadata (파일 원본)](#file_metadata-파일-원본)
  - [Memo_File (파일 첨부 내역)](#memo_file-파일-첨부-내역)
  - [Memo_Tag (메모-태그 매핑)](#memo_tag-메모-태그-매핑)
- [참조 무결성 및 삭제 연쇄 (Cascade)](#참조-무결성-및-삭제-연쇄-cascade)
  - [계층별 삭제 방어선 구축](#계층별-삭제-방어선-구축)
- [비정규화 및 DB 트리거](#비정규화-및-db-트리거)
- [상태 관리 및 DDL 전략](#상태-관리-및-ddl-전략)
  - [환경별 DDL 정책](#환경별-ddl-정책)
  - [기본값(Default) 이중화 제약](#기본값default-이중화-제약)

</details>

---

## 테이블 명세

### Memo (메모)

사용자가 작성한 메모 본문과 메타데이터를 저장하는 테이블.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| `id` | UUID | PK | 자동 생성 | 고유 식별자 |
| `user_id` | UUID | NOT NULL | — | 소유자 (Keycloak sub) |
| `content` | TEXT | — | — | 메모 본문 (Vault를 통해 암호화 저장 가능) |
| `is_pinned` | BOOLEAN | NOT NULL | `false` | 고정 여부 |
| `should_encrypt_content` | BOOLEAN | NOT NULL | `false` | 암호화 활성화 플래그 |
| `file_count` | INTEGER | NOT NULL | `0` | 첨부 파일 수 (통계용 비정규화) |
| `created_at` | TIMESTAMPTZ | NOT NULL | `now()` | 생성 시각 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `now()` | 수정 시각 |

[Memo.java](https://github.com/ellen24k-memo/backend/blob/main/src/main/java/io/github/ellen24k/memo_back/domain/Memo.java)

---

### Tag (태그)

사용자별로 생성한 태그 정보를 저장하는 테이블.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| `id` | UUID | PK | 자동 생성 | 고유 식별자 |
| `user_id` | UUID | NOT NULL | — | 소유자 ID |
| `name` | VARCHAR(50) | NOT NULL | — | 태그명 |
| `type` | VARCHAR(20) | NOT NULL, CHECK | `NORMAL` | 상태 (`NORMAL` / `FEATURED`) |
| `created_at` | TIMESTAMPTZ | NOT NULL | `now()` | 생성 시각 |

**제약조건 (Unique Constraint)**
<br/>`(user_id, name)` 조합에 Unique 인덱스를 걸어 해당 사용자 계정 스코프 내에서 태그명의 중복 생성 방어

[Tag.java](https://github.com/ellen24k-memo/backend/blob/main/src/main/java/io/github/ellen24k/memo_back/domain/Tag.java)

---

### File_Metadata (파일 원본 테이블)

MinIO에 업로드된 원본 파일을 관리하며, **CAS(Content-Addressable Storage)** 패턴을 구현한 테이블.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| `file_hash` | VARCHAR | **PK** | — | 파일 내용의 SHA-256 해시값 |
| `minio_object_name` | VARCHAR | NOT NULL | — | MinIO에 저장된 실제 오브젝트명 식별 키 |
| `original_filename` | VARCHAR | NOT NULL | — | 최초 업로드 시 파일명 |
| `content_type` | VARCHAR | — | — | MIME 타입 |
| `size` | BIGINT | NOT NULL | — | 파일 용량(bytes) |
| `reference_count` | INTEGER | NOT NULL | `0` | 해당 파일을 참조 중인 `Memo_File` 의 개수 |
| `created_at` | TIMESTAMPTZ | NOT NULL | — | 최초 업로드 시각 |

[FileMetadata.java](https://github.com/ellen24k-memo/backend/blob/main/src/main/java/io/github/ellen24k/memo_back/domain/FileMetadata.java)

---

### Memo_File (파일 첨부 매핑 테이블)

실제 메모(`Memo`)와 메모에 첨부된 파일(`File_Metadata`) 사이의 논리적 연결(N:1 다대일 매핑)을 담당하는 테이블.<br/>
메모 단위에서 동일 파일 저장 시 이 테이블에 참조 레코드를 생성할 뿐 실제 스토리지에 파일이 중복 저장되지는 않는다.

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| `id` | UUID | PK | 고유 식별자 |
| `memo_id` | UUID | FK → memo(id), NOT NULL | 소속 메모 |
| `file_hash` | VARCHAR | NOT NULL | 물리 파일의 무결성 참조키 (`File_Metadata`) |
| `file_name` | VARCHAR | NOT NULL | 사용자가 메모 컨텍스트에서 지정한 개별 파일명 |
| `content_type` | VARCHAR | — | MIME 타입 |
| `size` | BIGINT | NOT NULL | 파일 크기 |
| `minio_object_name` | VARCHAR | NOT NULL | MinIO 식별 키 통로 |

[File.java](https://github.com/ellen24k-memo/backend/blob/main/src/main/java/io/github/ellen24k/memo_back/domain/File.java)

---

### Memo_Tag (메모-태그 매핑 조인 테이블)

메모(`Memo`)와 태그(`Tag`) 간 다대다(M:N) 관계 설정을 위해 JPA의 `@ManyToMany` 속성이 도출한 조인 테이블. 

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| `memo_id` | UUID | FK → memo(id), NOT NULL | 메모 ID |
| `tag_id` | UUID | FK → tag(id), NOT NULL | 태그 ID |

복합 기본키 PK `(memo_id, tag_id)`를 맵핑하여 동일 태그를 중복 할당하지 않도록 한다.

---

## 참조 무결성 및 삭제 연쇄 (Cascade)

### 계층별 삭제 로직

| 계층 | 동작 방식 및 역할 |
|---|---|
| **애플리케이션 레이어 (Service)** | MinIO `reference_count`를 검증하여 실제 참조가 0일 때만 물리 파일을 삭제. |
| **JPA 프레임워크 (Entity)** | `cascade = ALL`, `orphanRemoval = true` 설정으로 부모 삭제 시 연관 컬렉션 생명주기를 영속성 컨텍스트 내에서 동기화. |
| **DB 물리 계층 (DDL)** | 외래키에 `ON DELETE CASCADE` 강제 적용. |

---

## 비정규화 및 DB 트리거

목록 조회 시 발생하는 `COUNT()` 서브쿼리 병목(N+1)을 제거하기 위해 `memo` 테이블에 `file_count` 컬럼을 도입 (비정규화). 

비정규화로 인한 데이터 파편화를 막고 백엔드 동시성(Lock) 제어 부담을 줄이고자 **PostgreSQL 트리거**로 동기화를 처리하였다.

**동작 방식 (`trg_memo_file_count`)**
*   `memo_file` 테이블 내역에 데이터 **INSERT** ➔ 연관된 `memo.file_count` **+1 증가**
*   `memo_file` 테이블 내역에 데이터 **DELETE** ➔ 연관된 `memo.file_count` **-1 감소**

```sql
CREATE OR REPLACE FUNCTION trg_memo_file_count()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE memo
       SET file_count = file_count + 1,
           updated_at = now()
     WHERE id = NEW.memo_id;
    RETURN NEW;

  ELSIF TG_OP = 'DELETE' THEN
    UPDATE memo
       SET file_count = GREATEST(file_count - 1, 0),
           updated_at = now()
     WHERE id = OLD.memo_id;
    RETURN OLD;
  END IF;

  RETURN NULL;
END;
$$;

-- INSERT 트리거
CREATE TRIGGER memo_file_after_insert
AFTER INSERT ON memo_file
FOR EACH ROW
EXECUTE FUNCTION trg_memo_file_count();

-- DELETE 트리거
CREATE TRIGGER memo_file_after_delete
AFTER DELETE ON memo_file
FOR EACH ROW
EXECUTE FUNCTION trg_memo_file_count();
```

---

## 상태 관리 및 DDL 전략

### 개발/운영 환경별 DDL 정책

| 환경 | DDL-Auto 속성 |
|---|---|
| **dev** | `update` |
| **prod** | `none` |

### 기본값(Default) 이중화

```java
// 1. 애플리케이션 레벨
private Boolean isPinned = false;
```
```sql
-- 2. DB 물리 제약 레벨
columnDefinition = "boolean not null default false"
```

JPA를 통한 삽입 외에도, 외부 스크립트나 배치 도구를 이용한 수동 삽입 시에도 무결성을 보장한다.