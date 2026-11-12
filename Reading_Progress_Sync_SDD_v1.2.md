# Software Design Document (SDD)

# Reading Progress Sync

**Version:** 1.2\
**Status:** Agreed Design\
**Document Type:** Software Design Document\
**Date:** 2026-11-12

------------------------------------------------------------------------

## 0. Revision History

| Version | Thay đổi chính |
|---------|----------------|
| 1.1 | Thiết kế link-first, Chrome Extension + Safari Web Extension |
| 1.2 | Thống nhất các điểm còn mở (xem bảng dưới) |

**Các quyết định đã thống nhất trong v1.2:**

| # | Vấn đề ở v1.1 | Quyết định v1.2 |
|---|---------------|-----------------|
| D1 | Định danh item dựa trên `user_id + normalized_source_url`, nhưng form Add chỉ có một ô URL nên URL chapter trở thành định danh; thêm chapter mới tạo item mới | Reading Item được định danh bằng **UUID**. Bỏ `source_url`. URL chỉ là **Progress URL**. Cập nhật tiến độ luôn thông qua item đã có |
| D2 | Last Write Wins theo timestamp server cho phép request offline gửi muộn ghi đè tiến độ mới hơn | **Optimistic concurrency bằng `version`**. Request kèm `expected_version`; nếu server có version mới hơn thì trả `409 Conflict`, người dùng chọn giữ bản nào |
| D3 | Unique constraint trên cột `normalized_source_url` không tồn tại | Bỏ unique constraint theo URL (hệ quả của D1) |
| D4 | `note` nằm ở cả `reading_items` và `reading_progress` | Chỉ giữ **một note ở Reading Item** |
| D5 | Xử lý trùng lặp mâu thuẫn giữa FR-01 và mục 10.5 | Khi Add, nếu nghi trùng thì **cảnh báo và gợi ý Update**; người dùng quyết định |
| D6 | Response API lúc lồng `progress`, lúc phẳng | **Lồng `progress` ở mọi API** |
| D7 | Safari Web Extension cần Mac, Xcode, Apple Developer account | iPhone dùng **Web App (PWA)**. Safari Web Extension chuyển sang Future Improvements |
| D8 | Chỉ có paste thủ công | Chrome Extension thêm nút **"Dùng URL tab hiện tại"** (quyền `activeTab`), vẫn giữ ô paste |
| D9 | Chưa rõ đối tượng người dùng | Dùng cá nhân + vài người quen; đăng ký có thể tắt bằng `ALLOW_REGISTRATION` |
| D10 | Chưa rõ nơi deploy | **VPS + Docker Compose + Caddy** (HTTPS tự động), có tên miền |

------------------------------------------------------------------------

## 1. Introduction

### 1.1. Purpose

Reading Progress Sync là hệ thống lưu trữ và đồng bộ vị trí đọc của
người dùng giữa nhiều thiết bị.

Hệ thống được thiết kế theo hướng **link-first**: người dùng chủ động
cung cấp URL của vị trí đang đọc (paste, hoặc bấm nút lấy URL tab hiện
tại trên Chrome). Hệ thống không phân tích HTML/DOM, không tự nhận diện
website và không có adapter riêng cho từng website.

Ví dụ:

1.  Người dùng đọc một bộ manga trên Chrome ở PC đến Chapter 100.
2.  Người dùng mở extension, bấm "Dùng URL tab hiện tại" (hoặc paste
    URL), nhập Title, chọn Content Type rồi Save.
3.  Đọc tiếp đến Chapter 105, người dùng mở item đó trong Library và bấm
    Update Progress với URL mới.
4.  Người dùng mở Web App trên iPhone (đã "Thêm vào Màn hình chính").
5.  Library hiển thị item với tiến độ Chapter 105.
6.  Người dùng chọn Continue Reading và được mở đúng URL đã lưu.

Cùng cơ chế này dùng được cho sách điện tử, web novel, bài viết và các
nội dung đọc trực tuyến khác.

### 1.2. Objectives

-   Đồng bộ Reading Library giữa Chrome trên PC và iPhone.
-   Người dùng chủ động thêm và cập nhật nội dung bằng URL.
-   Không phụ thuộc vào website cụ thể hoặc cấu trúc HTML.
-   Hỗ trợ nhiều loại nội dung: Manga, Manhwa, Webtoon, Book, Novel,
    Article, Other.
-   Lưu URL của vị trí đọc gần nhất để tiếp tục trên thiết bị khác.
-   Không bao giờ để một cập nhật cũ ghi đè âm thầm lên tiến độ mới hơn.
-   Quy trình thêm và cập nhật đơn giản, tối đa vài thao tác.
-   Không cần máy Mac hoặc tài khoản Apple Developer.

### 1.3. Out of Scope (MVP)

-   Đọc nội dung trực tiếp bên trong extension hoặc web app.
-   Download hoặc lưu nội dung truyện/sách.
-   Phân tích nội dung website để lấy title, chapter, page, cover.
-   Tự động theo dõi người dùng chuyển chapter (đọc DOM, lắng nghe thay
    đổi URL, đọc lịch sử duyệt web).
-   Tự động thêm nội dung người dùng truy cập.
-   Tìm kiếm nội dung trên nhiều website.
-   Đồng bộ bookmark, tab, lịch sử trình duyệt.
-   Nhận diện cùng một nội dung đăng trên nhiều website.
-   Lưu lịch sử các vị trí đọc trước đây.
-   Safari Web Extension và ứng dụng iOS native.
-   Xác minh email, quên mật khẩu.
-   Phát hành công khai trên Chrome Web Store.

### 1.4. Terminology

| Term | Description |
|------|-------------|
| Reading Item | Một nội dung người dùng theo dõi tiến độ (manga, book, novel, article...). Định danh bằng UUID |
| Content Type | Loại nội dung do người dùng chọn |
| Reading Library | Danh sách Reading Item của người dùng |
| Reading Progress | Vị trí đọc gần nhất của một Reading Item |
| Progress URL | URL dẫn tới vị trí đọc gần nhất |
| Progress Version | Số nguyên tăng dần mỗi lần progress thay đổi, dùng để phát hiện xung đột |
| Source Host | Hostname trích từ Progress URL, chỉ để hiển thị |
| Chrome Extension | Client trên Chrome desktop (Manifest V3) |
| Web App (PWA) | Client web dùng trên iPhone (và mọi trình duyệt khác), có thể "Thêm vào Màn hình chính" |
| Shared Core | Thư viện JavaScript dùng chung giữa Chrome Extension và Web App |
| Sync Queue | Hàng đợi local chứa các thay đổi chưa gửi được lên Backend |

------------------------------------------------------------------------

## 2. Overall Description

### 2.1. Product Perspective

Hệ thống gồm bốn thành phần:

1.  **Chrome Extension:** thêm item, cập nhật tiến độ, xem Library trên
    PC.
2.  **Web App (PWA):** cùng chức năng, dùng trên iPhone qua Safari hoặc
    bất kỳ trình duyệt nào.
3.  **Backend API (FastAPI):** xác thực, lưu trữ, xử lý xung đột.
4.  **Database (PostgreSQL):** tài khoản, Reading Item, Reading
    Progress.

Hai client dùng chung **Shared Core** và cùng một tài khoản.

### 2.2. Supported Platforms

| Platform | Browser | Client |
|----------|---------|--------|
| Windows/macOS/Linux desktop | Google Chrome (và Chromium: Edge, Brave...) | Chrome Extension |
| iPhone | Safari, Chrome iOS... | Web App (PWA) |
| Desktop/Android khác | Mọi trình duyệt hiện đại | Web App (PWA) |

Lưu ý:

-   Trình duyệt trên iOS (kể cả Chrome iOS) không hỗ trợ Chrome
    Extension.
-   Safari Web Extension cần Xcode trên macOS và tài khoản Apple
    Developer, nên không thuộc MVP (xem mục 14).
-   PWA trên iOS bắt buộc phục vụ qua HTTPS.

### 2.3. User Characteristics

-   Người dùng cá nhân và một nhóm nhỏ người quen.
-   Đọc trên nhiều website và nhiều thiết bị.
-   Muốn tự chọn nội dung cần lưu, không muốn bị thu thập lịch sử duyệt
    web.
-   Đọc nhiều loại nội dung khác nhau.
-   Không muốn chuyển sang một ứng dụng đọc khác.

### 2.4. Assumptions and Constraints

-   Có Internet khi cần đồng bộ; khi offline, thay đổi được xếp hàng
    chờ.
-   Người dùng cung cấp URL hợp lệ.
-   Website nguồn cho phép truy cập URL đã lưu. Hệ thống không đảm bảo
    URL còn hoạt động.
-   Title, content type, progress label, note do người dùng nhập hoặc
    chỉnh sửa.
-   Hostname trích từ URL chỉ để hiển thị.
-   Không cần quyền đọc nội dung trang hoặc theo dõi duyệt web. Quyền
    `activeTab` chỉ dùng để đọc URL và tiêu đề tab khi người dùng bấm
    nút.
-   Backend chạy trên VPS có tên miền và HTTPS.

------------------------------------------------------------------------

## 3. Functional Requirements

### 3.1. FR-01: Add Reading Item

**Description:** Thêm một nội dung vào Reading Library.

**Preconditions:** Người dùng đã đăng nhập.

**Main Flow:**

1.  Người dùng mở Chrome Extension popup hoặc Web App và chọn Add Item.
2.  Người dùng cung cấp URL bằng một trong các cách:
    -   **Chrome Extension:** bấm "Dùng URL tab hiện tại". URL và tiêu đề
        của tab đang mở được điền sẵn vào ô URL và ô Title.
    -   Paste URL vào ô URL.
    -   **Web App:** bấm nút "Dán" (đọc clipboard sau khi người dùng bấm)
        hoặc paste thủ công.
3.  Client validate URL.
4.  Người dùng nhập hoặc sửa các trường tùy chọn: Title, Content Type,
    Progress Label, Note, Cover Image URL.
5.  Client chạy **kiểm tra trùng lặp** (xem Business Rules).
6.  Người dùng bấm Save.
7.  Client gửi request tạo item lên Backend, kèm `id` do client sinh
    (UUID v4) để đảm bảo idempotency.
8.  Backend tạo Reading Item và Reading Progress ban đầu (`version = 1`).
9.  Client báo lưu thành công.

**Required Data:** Progress URL.

**Optional Data:** Title, Content Type, Progress Label, Note, Cover URL.

**Business Rules:**

-   Người dùng phải chủ động thêm nội dung.
-   Hệ thống không truy cập URL để lấy metadata.
-   "Dùng URL tab hiện tại" chỉ đọc `tab.url` và `tab.title` tại thời
    điểm bấm; không đọc DOM, không chạy content script.
-   Nếu không có title, UI hiển thị Source Host; người dùng có thể sửa
    sau.
-   Nếu không chọn Content Type, dùng `other`.
-   URL được lưu nguyên trạng sau validation; không thay đổi path hoặc
    query.
-   **Kiểm tra trùng lặp (chỉ cảnh báo, không chặn):** trước khi lưu,
    client so sánh với Library đã tải:
    1.  Trùng **Progress URL** (so sánh theo dạng chuẩn hóa ở mục 6.4),
        hoặc
    2.  Cùng **Source Host** và **Title** giống nhau (không phân biệt
        hoa/thường, bỏ khoảng trắng thừa).

    Nếu có item khớp, hiển thị: *"Có vẻ bạn đã lưu nội dung này: <title>
    — <progress label>"* với hai lựa chọn:
    -   **Cập nhật tiến độ item đó:** chuyển sang Update Progress với URL
        và label đã nhập.
    -   **Vẫn tạo mới.**
-   Nếu Backend nhận lại request tạo với `id` đã tồn tại của cùng người
    dùng (do retry), Backend trả về item hiện có, không tạo bản ghi mới.

### 3.2. FR-02: Update Reading Progress

**Description:** Cập nhật vị trí đọc của một item đã có.

**Main Flow:**

1.  Người dùng mở Library và chọn một Reading Item.
2.  Người dùng chọn Update Progress.
3.  Người dùng cung cấp URL mới (nút tab hiện tại, nút Dán, hoặc paste).
4.  Người dùng có thể sửa Progress Label.
5.  Client gửi request kèm `expected_version` là version mà client đang
    thấy.
6.  Backend cập nhật nếu version khớp (xem FR-03).
7.  Thiết bị khác thấy tiến độ mới khi tải lại Library.

**Business Rules:**

-   Progress chỉ thay đổi khi người dùng thao tác rõ ràng.
-   Không tự phát hiện chapter/page mới.
-   Progress Label là mô tả tự do, không bắt buộc là số.
-   Không có label thì vẫn lưu URL.
-   Không gửi request nếu URL và label không đổi.
-   Source Host được tính lại từ Progress URL mới.

### 3.3. FR-03: Progress Synchronization

**Description:** Đồng bộ Library và Progress giữa các thiết bị của cùng
tài khoản, không để cập nhật cũ ghi đè cập nhật mới.

**Synchronization Policy — Optimistic Concurrency:**

-   Mỗi Reading Progress có `version` (số nguyên, bắt đầu từ 1).
-   Mỗi request cập nhật progress gửi kèm `expected_version`.
-   Backend cập nhật nguyên tử:

    ``` sql
    UPDATE reading_progress
    SET progress_url = :url, progress_label = :label,
        version = version + 1, updated_at = now()
    WHERE reading_item_id = :item_id AND version = :expected_version
    RETURNING *;
    ```

-   Nếu cập nhật thành công (1 dòng), Backend trả progress mới với
    `version` mới.
-   Nếu không có dòng nào được cập nhật (version đã thay đổi):
    -   Nếu progress hiện tại **giống hệt** dữ liệu gửi lên (URL và
        label), đây là một **retry** của request đã thành công. Backend
        trả `200` với `updated: false`.
    -   Ngược lại, Backend trả `409 Conflict` kèm progress hiện tại trên
        server.
-   Khi nhận `409`, client hiển thị:
    *"Tiến độ đã được cập nhật từ thiết bị khác: <label server> (<thời
    gian>). Bạn muốn giữ bản nào?"*
    -   **Giữ bản trên server:** bỏ thay đổi local.
    -   **Dùng bản của tôi:** gửi lại với `expected_version` = version
        mới nhận được.
-   `updated_at` do Backend gán; client không quyết định timestamp.

**Metadata (title, content type, cover, note):** dùng Last Write Wins
theo thứ tự Backend nhận request. Rủi ro xung đột thấp và hậu quả nhỏ.

Lý do chọn version thay vì so sánh chapter: người dùng có thể chủ động
quay lại đọc vị trí cũ. Vì vậy không mặc định chapter lớn hơn là đúng.
Hệ thống chỉ cần đảm bảo không ghi đè **âm thầm**.

### 3.4. FR-04: Reading Library

Mỗi item hiển thị: Cover (nếu có), Title (hoặc Source Host), Content
Type, Source Host, Progress Label (nếu có), thời gian cập nhật gần nhất.

**Main Flow:**

1.  Người dùng mở extension hoặc Web App.
2.  Client hiển thị ngay Library từ cache local (nếu có).
3.  Client gọi `GET /reading-items` và cập nhật lại danh sách.
4.  Người dùng chọn item để Continue Reading, Update, Edit hoặc Remove.

**Business Rules:**

-   Chỉ hiển thị item của tài khoản hiện tại.
-   Mặc định sắp xếp theo thời gian cập nhật gần nhất, giảm dần. Thời
    gian này là giá trị lớn hơn giữa `progress.updated_at` và
    `updated_at` của item.
-   Lọc được theo Content Type.
-   Không có item thì hiển thị empty state.

### 3.5. FR-05: Continue Reading

**Main Flow:**

1.  Người dùng chọn Continue Reading trên một item.
2.  Client mở Progress URL:
    -   **Chrome Extension:** `chrome.tabs.create({ url })`.
    -   **Web App:** mở link ngoài (`<a href target="_blank"
        rel="noopener">`). Trên iPhone, link mở trong Safari.

**Business Rules:**

-   Mở chính xác URL đã lưu.
-   Không suy luận URL từ title hoặc label.
-   URL hỏng thì trình duyệt hoặc website hiển thị lỗi; người dùng có
    thể cập nhật lại URL.

### 3.6. FR-06: Edit Reading Item

Các trường có thể sửa: Title, Content Type, Cover URL, Note, Progress
URL, Progress Label.

**Business Rules:**

-   Chỉ chủ sở hữu được sửa.
-   Sửa Progress URL hoặc Progress Label được xử lý như **Update
    Progress** (FR-02), tức là có version check.
-   Sửa metadata dùng `PATCH /reading-items/{id}` và không làm thay đổi
    progress.

### 3.7. FR-07: Remove Reading Item

1.  Người dùng chọn Remove; client yêu cầu xác nhận.
2.  Client gửi `DELETE`.
3.  Backend xóa item và progress (cascade).
4.  Client cập nhật danh sách.

**Business Rules:** chỉ xóa item của mình. Xóa item đã bị xóa trước đó
trả `404` và client coi như thành công.

### 3.8. FR-08: Content Type Management

| Code | Display Name |
|------|--------------|
| `manga` | Manga |
| `manhwa` | Manhwa |
| `webtoon` | Webtoon |
| `book` | Book |
| `novel` | Novel |
| `article` | Article |
| `other` | Other |

-   Content Type chỉ dùng để phân loại và lọc.
-   Không ảnh hưởng cách lưu URL hoặc đồng bộ.
-   Danh sách định nghĩa ở Backend (enum/validation) và Shared Core; thêm
    loại mới không cần đổi kiến trúc.
-   Có thể sửa sau khi tạo.

### 3.9. FR-09: Settings and Account

Cả hai client đều có:

-   Sign in / Sign out; Register (ẩn nếu server tắt đăng ký).
-   Trạng thái đồng bộ: Synced / Pending (n) / Failed / Offline.
-   Nút Sync now.
-   Thông báo lỗi khi không thể đồng bộ.

Riêng Chrome Extension có thêm cấu hình **Backend URL** (mặc định là
domain production). Web App luôn dùng cùng origin với API.

------------------------------------------------------------------------

## 4. Non-functional Requirements

### 4.1. Performance

| ID | Requirement |
|----|-------------|
| NFR-01 | Extension không ảnh hưởng tốc độ tải trang (không có content script) |
| NFR-02 | `GET /reading-items` phản hồi < 2 giây trong điều kiện bình thường |
| NFR-03 | Cập nhật progress phản hồi < 2 giây |
| NFR-04 | Không gửi request nếu dữ liệu không đổi |
| NFR-05 | Popup và Web App hiển thị Library từ cache ngay khi mở, rồi làm mới nền |

### 4.2. Reliability

-   Không mất dữ liệu đã được Backend xác nhận.
-   Mất mạng thì thay đổi vào Sync Queue và được retry với exponential
    backoff.
-   Request tạo item idempotent nhờ `id` do client sinh.
-   Request cập nhật progress không bao giờ ghi đè âm thầm bản mới hơn
    (version check).
-   Lỗi validation không làm hỏng client.
-   Backup PostgreSQL hằng ngày (`pg_dump`, giữ tối thiểu 7 bản).

### 4.3. Security

-   HTTPS bắt buộc ở production (Caddy tự cấp chứng chỉ).
-   Mọi API nghiệp vụ yêu cầu access token.
-   Người dùng chỉ truy cập dữ liệu của mình: mọi query đều lọc theo
    `user_id` lấy từ token.
-   Mật khẩu băm bằng Argon2 (hoặc bcrypt).
-   Refresh token lưu trên server ở dạng hash, có thể thu hồi khi logout.
-   Đăng ký tài khoản bật/tắt bằng `ALLOW_REGISTRATION`.
-   Rate limiting cho `/auth/login` và `/auth/register`.
-   **Quyền Chrome Extension tối thiểu:** `storage`, `activeTab`, host
    permission cho domain Backend. Không có content script, không có
    quyền `tabs` rộng, `history` hay `<all_urls>`.
-   Web App dùng Content Security Policy chặt, không tải script bên thứ
    ba.
-   Chỉ chấp nhận URL `http`/`https`. Backend không gửi request tới URL
    người dùng cung cấp. Nếu tương lai cần làm vậy, phải chống SSRF.
-   Không thu thập nội dung trang.

### 4.4. Maintainability

-   Tách UI, domain logic, API client và platform adapter.
-   Shared Core không gọi trực tiếp API của trình duyệt (`chrome.*`,
    `window`); những phần đó được truyền vào qua adapter.
-   Không có website-specific adapter.
-   API có version (`/api/v1`). Thay đổi schema qua Alembic migration.
-   Backend có logging có cấu trúc, gồm request id và user id. Không log
    token hay mật khẩu.

### 4.5. Compatibility

-   Chrome Extension: Manifest V3.
-   Web App: Safari iOS 16.4+ và các trình duyệt hiện đại; có manifest
    PWA và service worker.
-   Shared Core là JavaScript ES Modules thuần, không phụ thuộc
    framework.

------------------------------------------------------------------------

## 5. System Architecture

### 5.1. Overview

**Clients:** Chrome Extension, Web App (PWA). Cả hai dùng Shared Core
(Reading Item Manager, URL Validator, Duplicate Checker, Sync Client,
Sync Queue, API Client).

**Backend:** FastAPI gồm Auth Module, Reading Item Module, Progress Sync
Module.

**Database:** PostgreSQL.

**Edge:** Caddy (HTTPS, reverse proxy `/api` tới FastAPI, phục vụ file
tĩnh của Web App).

### 5.2. High-level Architecture

``` mermaid
flowchart TD
    subgraph PC["PC"]
        Chrome["Chrome"] --> CE["Chrome Extension"]
    end

    subgraph Phone["iPhone"]
        Safari["Safari"] --> PWA["Web App (PWA)"]
    end

    subgraph Core["Shared Core (JS)"]
        Manager["Reading Item Manager"]
        Validator["URL Validator"]
        Dup["Duplicate Checker"]
        SyncC["Sync Client + Queue"]
        ApiC["API Client"]
    end

    CE --> Core
    PWA --> Core

    subgraph Server["VPS (Docker Compose)"]
        Caddy["Caddy (HTTPS)"]
        API["FastAPI"]
        DB[("PostgreSQL")]
        Static["Web App static files"]
    end

    ApiC -->|HTTPS /api/v1| Caddy
    PWA -.->|load app| Caddy
    Caddy --> API
    Caddy --> Static
    API --> DB
```

### 5.3. Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| Chrome Extension | Popup UI, lấy URL tab hiện tại, mở tab, lưu trữ bằng `chrome.storage` |
| Web App (PWA) | UI responsive cho mobile, nút Dán, lưu trữ bằng IndexedDB/localStorage, service worker cache app shell |
| Reading Item Manager | Nghiệp vụ create/update/edit/remove ở client |
| URL Validator | Validate và chuẩn hóa URL để so sánh, trích hostname |
| Duplicate Checker | Tìm item nghi trùng trong Library local |
| Sync Client + Queue | Gửi thay đổi, retry, xử lý 409, trạng thái đồng bộ |
| API Client | Gọi HTTP, gắn token, tự refresh token |
| FastAPI | Auth, CRUD, version check, validation |
| PostgreSQL | Lưu trữ |
| Caddy | HTTPS, routing, phục vụ Web App |

### 5.4. Repository Structure

``` text
reading_sync_extension/
├── docs/
│   └── Reading_Progress_Sync_SDD_v1.2.md
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── db.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── security.py
│   │   └── routers/
│   │       ├── auth.py
│   │       └── reading_items.py
│   ├── alembic/
│   ├── tests/
│   ├── pyproject.toml
│   └── Dockerfile
├── core/                      # Shared Core (ES modules, không dùng browser API)
│   ├── content-types.js
│   ├── url-validator.js
│   ├── duplicate-checker.js
│   ├── reading-item-manager.js
│   ├── api-client.js
│   ├── sync-queue.js
│   └── tests/
├── extension/                 # Chrome Extension (MV3)
│   ├── manifest.json
│   ├── background/service-worker.js
│   ├── platform/              # adapter: storage, openUrl, currentTab
│   ├── popup/
│   └── options/
├── web/                       # Web App (PWA)
│   ├── index.html
│   ├── manifest.webmanifest
│   ├── sw.js
│   ├── platform/              # adapter: storage, openUrl, clipboard
│   └── app/
├── scripts/
│   └── build.(js|ps1)         # copy core/ vào extension/ và web/ khi build
└── deployment/
    ├── docker-compose.yml
    ├── Caddyfile
    └── .env.example
```

Chrome Extension không import được file nằm ngoài thư mục của nó. Vì
vậy script build sao chép `core/` vào `extension/core/` và `web/core/`
(các thư mục này nằm trong `.gitignore`).

------------------------------------------------------------------------

## 6. Detailed Design

### 6.1. URL Input

**Input (conceptual):**

``` javascript
{
  id: "uuid-v4-generated-by-client",
  progressUrl: "https://example.com/series/example/chapter-100",
  title: "Example Series",
  contentType: "manga",
  progressLabel: "Chapter 100",
  note: null,
  coverUrl: null
}
```

**Các nguồn URL:**

| Nguồn | Client | Ghi chú |
|-------|--------|---------|
| "Dùng URL tab hiện tại" | Chrome Extension | `chrome.tabs.query({active: true, currentWindow: true})` với quyền `activeTab`; điền sẵn URL và title |
| Nút "Dán" | Web App | `navigator.clipboard.readText()` sau khi người dùng bấm; iOS có thể hiện popup xác nhận |
| Paste thủ công | Cả hai | Luôn có |

**Design decisions:** không phân tích DOM, không cần danh sách website
hỗ trợ, không tự lấy chapter/cover, không sửa URL theo heuristic.

### 6.2. Reading Item Model

| Field | Ghi chú |
|-------|---------|
| id | UUID, **định danh duy nhất**; client có thể sinh |
| title | Tùy chọn |
| content_type | Mặc định `other` |
| source_host | Trích từ Progress URL hiện tại, chỉ để hiển thị |
| cover_url | Tùy chọn |
| note | Tùy chọn, **note duy nhất của item** |
| progress | Progress URL, label, version, updated_at |
| created_at, updated_at | Do Backend quản lý |

Không có `source_url` và không có unique constraint theo URL. Cùng một
truyện chỉ có một item vì người dùng cập nhật item đó qua Update
Progress. Duplicate Checker cảnh báo khi người dùng có vẻ đang thêm lại.

### 6.3. Reading Progress

| Field | Ghi chú |
|-------|---------|
| progress_url | Bắt buộc |
| progress_label | Tùy chọn, tự do |
| version | Số nguyên, Backend tăng mỗi lần cập nhật |
| updated_at | Backend gán |

Ví dụ:

| Content Type | Progress URL | Progress Label |
|--------------|--------------|----------------|
| Manga | URL chapter | Chapter 100 |
| Book | URL trang sách/reader | Page 120 |
| Novel | URL phần truyện | Vol 3 - Ch 12 |
| Article | URL bài viết | Part 2 |

### 6.4. URL Validation and Normalization

**Validation (client và Backend):**

-   Không rỗng; tự cắt khoảng trắng đầu và cuối.
-   Parse được theo chuẩn WHATWG URL.
-   Scheme chỉ `http` hoặc `https`.
-   Độ dài tối đa 2048 ký tự (cấu hình được).
-   Backend luôn validate lại.
-   Không request tới URL để kiểm tra, không crawl.

**Chuẩn hóa chỉ dùng để so sánh trùng lặp, không áp dụng cho URL lưu
trữ:**

-   Scheme và hostname viết thường.
-   Bỏ port mặc định (`:80`, `:443`).
-   Bỏ fragment (`#...`).
-   Bỏ dấu `/` cuối path (trừ path `/`).
-   **Giữ nguyên query parameters.**

**Source Host:** hostname viết thường, bỏ tiền tố `www.`.

### 6.5. Sync Client and Queue

**Trách nhiệm:**

-   Gửi request, gắn token, tự refresh khi nhận `401`.
-   Cập nhật lạc quan (optimistic UI): ghi vào cache local trước, đánh
    dấu `pending`.
-   Lỗi mạng hoặc `5xx` thì giữ trong Sync Queue và retry với backoff
    (1s, 2s, 4s... tối đa 5 phút). Retry ngay khi có sự kiện `online`
    hoặc khi mở client.
-   Gộp thay đổi: nhiều lần Update Progress chờ trên cùng một item chỉ
    giữ lần cuối.
-   `409` thì dừng retry item đó và hiện hộp thoại xung đột (FR-03).
-   `4xx` khác thì đánh dấu `failed` và báo lỗi cho người dùng.
-   Không báo "đã đồng bộ" trước khi Backend xác nhận.

**Lưu trữ queue:** Chrome Extension dùng `chrome.storage.local`, Web App
dùng IndexedDB.

### 6.6. UI States (dùng chung cho hai client)

-   **Library:** danh sách item, lọc theo Content Type, sắp xếp theo thời
    gian cập nhật gần nhất, badge trạng thái sync của từng item.
-   **Add Item:** URL (kèm nút tab hiện tại hoặc nút Dán), Title, Content
    Type, Progress Label, Cover URL, Note, Save; cảnh báo trùng lặp.
-   **Update Progress:** URL (kèm nút tab hoặc Dán), Progress Label,
    Update.
-   **Edit Item:** metadata và progress.
-   **Conflict dialog:** so sánh bản local với bản server.
-   **Empty / Loading / Error / Offline.**
-   **Login / Register / Settings.**

Popup extension rộng khoảng 380px; Web App ưu tiên thiết kế cho mobile.

### 6.7. Platform Integration

| Adapter | Chrome Extension | Web App |
|---------|------------------|---------|
| storage | `chrome.storage.local` | IndexedDB / localStorage |
| openUrl | `chrome.tabs.create` | `window.open` / link `_blank` |
| getCurrentTab | `chrome.tabs.query` (activeTab) | Không có |
| readClipboard | Không cần | `navigator.clipboard.readText` |
| backendUrl | Cấu hình trong Options | Cùng origin |

------------------------------------------------------------------------

## 7. Data Design

### 7.1. Entity Relationship

``` mermaid
erDiagram
    USERS ||--o{ READING_ITEMS : owns
    USERS ||--o{ REFRESH_TOKENS : has
    READING_ITEMS ||--|| READING_PROGRESS : has

    USERS {
        uuid id PK
        string email UK
        string password_hash
        datetime created_at
    }

    REFRESH_TOKENS {
        uuid id PK
        uuid user_id FK
        string token_hash
        datetime expires_at
        datetime revoked_at
        datetime created_at
    }

    READING_ITEMS {
        uuid id PK
        uuid user_id FK
        string title
        string content_type
        string source_host
        text cover_url
        text note
        datetime created_at
        datetime updated_at
    }

    READING_PROGRESS {
        uuid reading_item_id PK
        text progress_url
        string progress_label
        int version
        datetime updated_at
    }
```

### 7.2. Users

| Field | Type | Description |
|-------|------|-------------|
| id | UUID | PK |
| email | VARCHAR(320) | Unique, lưu dạng viết thường |
| password_hash | VARCHAR | Argon2/bcrypt |
| created_at | TIMESTAMPTZ | |

### 7.3. Refresh Tokens

| Field | Type | Description |
|-------|------|-------------|
| id | UUID | PK |
| user_id | UUID | FK users, ON DELETE CASCADE |
| token_hash | VARCHAR | SHA-256 của refresh token |
| expires_at | TIMESTAMPTZ | |
| revoked_at | TIMESTAMPTZ | Nullable; đặt khi logout hoặc rotate |
| created_at | TIMESTAMPTZ | |

Refresh token được **rotate** mỗi lần dùng: token cũ bị thu hồi và một
token mới được cấp.

### 7.4. Reading Items

| Field | Type | Description |
|-------|------|-------------|
| id | UUID | PK (client có thể cung cấp) |
| user_id | UUID | FK users, ON DELETE CASCADE |
| title | VARCHAR(500) | Nullable |
| content_type | VARCHAR(20) | CHECK thuộc danh sách FR-08, mặc định `other` |
| source_host | VARCHAR(255) | Trích từ progress_url |
| cover_url | TEXT | Nullable, đã validate http/https |
| note | TEXT | Nullable, tối đa 2000 ký tự |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | Thời gian sửa metadata |

Index: `(user_id, updated_at DESC)`.

### 7.5. Reading Progress

| Field | Type | Description |
|-------|------|-------------|
| reading_item_id | UUID | PK và FK reading_items, ON DELETE CASCADE |
| progress_url | TEXT | Bắt buộc, tối đa 2048 ký tự |
| progress_label | VARCHAR(200) | Nullable |
| version | INTEGER | NOT NULL, bắt đầu từ 1 |
| updated_at | TIMESTAMPTZ | |

Dùng `reading_item_id` làm PK để đảm bảo quan hệ 1-1.

### 7.6. Data Retention

Chỉ lưu trạng thái mới nhất. Không lưu lịch sử URL, lịch sử progress,
thời lượng đọc, lịch sử duyệt web, nội dung trang. Xóa item thì progress
bị xóa theo (cascade).

------------------------------------------------------------------------

## 8. API Design

Base URL: `/api/v1`. Mọi API nghiệp vụ yêu cầu `Authorization: Bearer
<access_token>`.

### 8.1. Common Formats

**Reading Item (dạng chuẩn dùng trong mọi response):**

``` json
{
  "id": "6f1c...",
  "title": "Example Manga",
  "content_type": "manga",
  "source_host": "example.com",
  "cover_url": null,
  "note": null,
  "progress": {
    "progress_url": "https://example.com/series/example/chapter-100",
    "progress_label": "Chapter 100",
    "version": 3,
    "updated_at": "2026-11-12T10:15:00Z"
  },
  "created_at": "2026-11-10T08:00:00Z",
  "updated_at": "2026-11-10T08:00:00Z"
}
```

**Error:**

``` json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "progress_url must use http or https",
    "details": {}
  }
}
```

Các mã lỗi: `VALIDATION_ERROR` (422), `UNAUTHORIZED` (401), `NOT_FOUND`
(404), `PROGRESS_CONFLICT` (409), `REGISTRATION_DISABLED` (403),
`RATE_LIMITED` (429).

### 8.2. Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Đăng ký (403 nếu `ALLOW_REGISTRATION=false`) |
| POST | `/auth/login` | Trả `access_token`, `refresh_token` |
| POST | `/auth/refresh` | Đổi refresh token lấy cặp token mới (rotate) |
| POST | `/auth/logout` | Thu hồi refresh token |
| GET | `/auth/config` | Không cần auth; trả `{ "allow_registration": bool }` |

### 8.3. Reading Items

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/reading-items` | Danh sách (lọc `?content_type=`) |
| POST | `/reading-items` | Tạo item và progress ban đầu |
| GET | `/reading-items/{id}` | Chi tiết |
| PATCH | `/reading-items/{id}` | Sửa metadata |
| DELETE | `/reading-items/{id}` | Xóa |
| PUT | `/reading-items/{id}/progress` | Cập nhật progress (version check) |

### 8.4. Create Reading Item

``` http
POST /api/v1/reading-items
```

``` json
{
  "id": "6f1c...",
  "title": "Example Manga",
  "content_type": "manga",
  "progress_url": "https://example.com/series/example/chapter-100",
  "progress_label": "Chapter 100",
  "cover_url": null,
  "note": null
}
```

-   `201 Created` kèm Reading Item, `progress.version = 1`.
-   Nếu `id` đã tồn tại và thuộc người dùng này: `200 OK` kèm item hiện
    có (retry idempotent).
-   Nếu `id` đã tồn tại nhưng thuộc người dùng khác: `409`.
-   Không gửi `id` thì server tự sinh.

### 8.5. Update Reading Progress

``` http
PUT /api/v1/reading-items/{id}/progress
```

``` json
{
  "progress_url": "https://example.com/series/example/chapter-101",
  "progress_label": "Chapter 101",
  "expected_version": 3
}
```

**Thành công:**

``` json
{
  "updated": true,
  "item": { "...": "Reading Item với progress.version = 4" }
}
```

**Retry của request đã áp dụng** (dữ liệu trùng khớp bản hiện tại): `200`
với `"updated": false` kèm item.

**Xung đột:**

``` http
409 Conflict
```

``` json
{
  "error": {
    "code": "PROGRESS_CONFLICT",
    "message": "Progress was updated from another device",
    "details": {
      "current": {
        "progress_url": "https://example.com/series/example/chapter-105",
        "progress_label": "Chapter 105",
        "version": 5,
        "updated_at": "2026-11-12T11:00:00Z"
      }
    }
  }
}
```

### 8.6. Update Metadata

``` http
PATCH /api/v1/reading-items/{id}
```

``` json
{ "title": "New title", "content_type": "manhwa", "note": "..." }
```

Chỉ những trường có mặt trong body mới được cập nhật; gửi `null` để xóa
giá trị. Trả về Reading Item.

### 8.7. Get Reading Library

``` http
GET /api/v1/reading-items?content_type=manga
```

``` json
{
  "items": [ { "...": "Reading Item" } ],
  "total": 1
}
```

Sắp xếp theo `GREATEST(progress.updated_at, item.updated_at) DESC`.
Không phân trang trong MVP vì số lượng item của mỗi người dùng nhỏ.

### 8.8. Delete Reading Item

``` http
DELETE /api/v1/reading-items/{id}
```

`204 No Content`. Item không tồn tại thì trả `404`; client coi là thành
công.

------------------------------------------------------------------------

## 9. Main User Flows

### 9.1. Add from Chrome (current tab)

``` mermaid
sequenceDiagram
    actor User
    participant Ext as Chrome Extension
    participant API
    participant DB

    User->>Ext: Open popup, click "Dùng URL tab hiện tại"
    Ext->>Ext: Read tab.url + tab.title (activeTab)
    User->>Ext: Edit title, choose content type, label
    Ext->>Ext: Validate URL + duplicate check
    alt Possible duplicate
        Ext-->>User: "Có vẻ bạn đã lưu..." [Update item đó / Vẫn tạo mới]
    end
    User->>Ext: Save
    Ext->>API: POST /reading-items (client id)
    API->>DB: Insert item + progress(version=1)
    API-->>Ext: 201 item
    Ext-->>User: Saved
```

### 9.2. Update Progress with Conflict

``` mermaid
sequenceDiagram
    actor User
    participant Phone as Web App (iPhone)
    participant PC as Chrome Extension
    participant API

    Note over Phone: Offline
    User->>Phone: Update -> Ch 50 (expected_version=3)
    Phone->>Phone: Queue request
    User->>PC: Update -> Ch 55 (expected_version=3)
    PC->>API: PUT progress
    API-->>PC: 200 (version=4)
    Note over Phone: Back online
    Phone->>API: PUT progress (expected_version=3)
    API-->>Phone: 409 current=Ch 55 v4
    Phone-->>User: Conflict dialog
    alt Keep server
        Phone->>Phone: Drop local change
    else Use mine
        Phone->>API: PUT progress (expected_version=4)
        API-->>Phone: 200 (version=5)
    end
```

### 9.3. Continue Reading on iPhone

``` mermaid
sequenceDiagram
    actor User
    participant PWA as Web App
    participant API
    participant Safari
    participant Website

    User->>PWA: Open from Home Screen
    PWA-->>User: Show cached Library
    PWA->>API: GET /reading-items
    API-->>PWA: Items
    PWA-->>User: Refreshed Library
    User->>PWA: Continue Reading
    PWA->>Safari: Open progress_url
    Safari->>Website: Navigate
```

------------------------------------------------------------------------

## 10. Error Handling

| Tình huống | Hành vi |
|------------|---------|
| URL rỗng, không parse được, sai scheme, quá dài | Không gửi request; hiển thị lỗi; giữ nguyên dữ liệu đã nhập |
| Mất mạng / Backend không phản hồi / 5xx | Đưa vào Sync Queue, hiển thị Pending, retry với backoff; không báo thành công trước khi Backend xác nhận |
| 401 | Tự refresh token rồi gửi lại một lần; refresh thất bại thì về màn hình đăng nhập, giữ nguyên queue |
| 409 `PROGRESS_CONFLICT` | Hộp thoại xung đột (FR-03) |
| 404 khi update hoặc edit | Item đã bị xóa ở thiết bị khác; thông báo và xóa khỏi cache |
| 404 khi delete | Coi là thành công |
| URL website không còn hoạt động | Giữ URL; người dùng tự cập nhật; hệ thống không đoán URL thay thế |
| Nghi trùng khi Add | Cảnh báo và gợi ý Update (FR-01) |
| 429 | Hiển thị thông báo thử lại sau |

------------------------------------------------------------------------

## 11. Deployment

### 11.1. Production (VPS)

``` text
deployment/
├── docker-compose.yml   # caddy, api, db
├── Caddyfile
└── .env                 # không commit
```

| Service | Technology | Responsibility |
|---------|-----------|----------------|
| caddy | Caddy 2 | HTTPS tự động (Let's Encrypt), `/api/*` tới api, `/*` phục vụ Web App |
| api | FastAPI + Uvicorn | REST API; chạy `alembic upgrade head` khi khởi động |
| db | PostgreSQL 16 | Lưu trữ, volume bền vững |

**Caddyfile (conceptual):**

``` text
reading.example.com {
    handle /api/* {
        reverse_proxy api:8000
    }
    handle {
        root * /srv/web
        try_files {path} /index.html
        file_server
    }
}
```

Backup: cron trên VPS chạy `pg_dump` hằng ngày, giữ 7 bản, nên sao chép
ra ngoài VPS.

Yêu cầu: VPS khoảng 1 vCPU / 1 GB RAM, một tên miền trỏ tới VPS.

### 11.2. Configuration

``` env
DATABASE_URL=postgresql+asyncpg://...
JWT_SECRET=
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=60
ALLOW_REGISTRATION=true
CORS_ORIGINS=chrome-extension://<extension-id>
DOMAIN=reading.example.com
```

Không commit secret. Extension không chứa secret.

### 11.3. Chrome Extension

-   Development: chạy build script, rồi Load unpacked thư mục
    `extension/`.
-   Chia sẻ cho người quen: gửi file zip để Load unpacked, hoặc đăng lên
    Chrome Web Store ở chế độ Unlisted (tùy chọn, phí đăng ký một lần).
-   Host permission cho domain backend production khai báo trong
    `manifest.json`. Nếu người dùng đổi Backend URL thì xin
    `optional_host_permissions` lúc runtime.

### 11.4. Web App on iPhone

1.  Mở `https://reading.example.com` bằng Safari.
2.  Chọn Share → "Thêm vào Màn hình chính".
3.  Đăng nhập một lần. Từ đó mở app như ứng dụng thường.

### 11.5. Local Development

`docker compose -f deployment/docker-compose.dev.yml up` chạy db + api
(hot reload). Web App phục vụ tĩnh ở localhost. Extension trỏ Backend
URL về `http://localhost:8000`.

------------------------------------------------------------------------

## 12. Testing Strategy

### 12.1. Unit Tests

**Backend (pytest):** register/login/refresh/logout, register khi tắt
đăng ký, tạo item, tạo lại cùng `id` (idempotent), validate URL và
content type, PATCH metadata, PUT progress thành công, **PUT với version
cũ trả 409**, **PUT retry trùng dữ liệu trả `updated:false`**, hai PUT
đồng thời chỉ một bên thắng, delete cascade, cách ly dữ liệu giữa hai
user.

**Shared Core (Vitest hoặc node:test):** validate URL, chuẩn hóa để so
sánh, trích Source Host, Duplicate Checker, gộp queue, backoff, xử lý
401/409/404.

### 12.2. Integration

-   Extension ↔ Backend, Web App ↔ Backend.
-   Backend ↔ PostgreSQL (dùng database test thật, không mock).
-   Refresh token khi access token hết hạn.
-   Retry sau khi mất kết nối.

### 12.3. End-to-end

| Test Case | Expected |
|-----------|----------|
| Add bằng nút tab hiện tại trên Chrome | URL và title được điền sẵn, item được lưu |
| Paste URL sách trên Chrome | Book được lưu |
| Add item không có title | Lưu được, hiển thị hostname |
| Add URL đã có | Hiện cảnh báo trùng, chọn Update thì cập nhật đúng item |
| Update progress trên Chrome | Web App iPhone thấy tiến độ mới sau khi refresh |
| Update offline trên iPhone, rồi update trên PC, rồi iPhone online | Hiện hộp thoại xung đột, không ghi đè âm thầm |
| Continue Reading trên iPhone | Safari mở đúng URL |
| Lọc Content Type | Chỉ hiển thị loại tương ứng |
| Edit và Remove | Phản ánh trên thiết bị còn lại |
| URL không hợp lệ | Hiện lỗi, giữ dữ liệu đã nhập |

### 12.4. Acceptance Criteria (MVP)

1.  Thêm Reading Item bằng nút tab hiện tại (Chrome) hoặc paste URL.
2.  Chọn được Content Type (7 loại mặc định).
3.  Cập nhật progress bằng URL mới.
4.  Web App trên iPhone (cài từ Màn hình chính) tải được Library.
5.  Continue Reading mở đúng Progress URL trên cả hai client.
6.  Thay đổi trên một thiết bị hiển thị trên thiết bị kia.
7.  Cập nhật cũ không bao giờ ghi đè âm thầm cập nhật mới (409 kèm hộp
    thoại).
8.  Có cảnh báo khi thêm item nghi trùng.
9.  Edit và Remove hoạt động.
10. Không có adapter riêng cho từng website; extension không có content
    script.
11. Dữ liệu giữa các tài khoản được cách ly.
12. Backend chạy HTTPS trên VPS và có backup hằng ngày.

------------------------------------------------------------------------

## 13. Development Roadmap

### Phase 1 — Backend Foundation

-   Khởi tạo repo (git), cấu trúc thư mục theo mục 5.4.
-   FastAPI, SQLAlchemy, Alembic; schema mục 7.
-   Auth (JWT, refresh token rotation, `ALLOW_REGISTRATION`).
-   Reading Item APIs và Progress API có version check.
-   URL validation.
-   Docker Compose cho dev; pytest.

### Phase 2 — Shared Core + Chrome Extension MVP

-   Shared Core: validator, duplicate checker, API client, manager.
-   Extension MV3: login, Library, Add (tab hiện tại hoặc paste), Update,
    Edit, Remove, Continue Reading, lọc.
-   Build script copy core.

### Phase 3 — Sync Reliability

-   Sync Queue, retry/backoff, gộp thay đổi.
-   Hộp thoại xung đột 409.
-   Trạng thái sync, cache Library local.
-   Test đồng thời.

### Phase 4 — Web App (PWA)

-   UI mobile-first dùng lại Shared Core.
-   Web manifest, service worker (cache app shell).
-   Nút Dán, IndexedDB queue.
-   Kiểm thử trên iPhone thật (Safari, cài từ Màn hình chính).

### Phase 5 — Deploy & Release

-   VPS, tên miền, Docker Compose production, Caddy HTTPS.
-   Backup `pg_dump` hằng ngày.
-   Rate limiting, rà soát bảo mật.
-   End-to-end test, đóng gói extension.

------------------------------------------------------------------------

## 14. Future Improvements

-   Safari Web Extension (cần Mac, Xcode, Apple Developer account).
-   iOS Shortcut cho phép gửi URL từ Share Sheet vào Web App.
-   Context menu "Lưu/Update trang này" trên Chrome.
-   Custom Content Type, tag, collection.
-   Tìm kiếm trong Library.
-   Lịch sử progress.
-   Import/export Library (JSON).
-   Xác minh email, quên mật khẩu, phát hành công khai.
-   Đồng bộ real-time (SSE/WebSocket) thay cho refresh khi mở app.

------------------------------------------------------------------------

## 15. Open Design Considerations

1.  **Lưu token trên Web App:** MVP lưu access token và refresh token
    trong localStorage/IndexedDB, giảm rủi ro XSS bằng CSP chặt và không
    dùng script bên thứ ba. Có thể chuyển refresh token sang cookie
    `HttpOnly; Secure; SameSite=Strict` (cùng origin) nếu muốn tăng bảo
    mật.
2.  **Progress granularity:** một số reader lưu vị trí trong local
    storage thay vì URL. MVP chỉ lưu URL kèm label do người dùng nhập.
3.  **Làm mới dữ liệu:** MVP làm mới Library khi mở client và khi bấm
    Sync now. Đồng bộ real-time để sau.
4.  **Clipboard trên iOS:** `navigator.clipboard.readText()` có thể hiện
    popup xác nhận "Dán". Luôn giữ ô paste thủ công làm phương án dự
    phòng.

------------------------------------------------------------------------

## 16. Final Design Summary

-   Người dùng chủ động cung cấp URL (nút tab hiện tại trên Chrome, nút
    Dán hoặc paste).
-   Không phân tích DOM, không có website adapter, không có content
    script.
-   Reading Item định danh bằng UUID; URL chỉ là Progress URL.
-   Cập nhật tiến độ qua item có sẵn; khi thêm item nghi trùng thì có
    cảnh báo.
-   Optimistic concurrency bằng `version`: không bao giờ ghi đè âm thầm.
-   Hai client: Chrome Extension (PC) và Web App PWA (iPhone), dùng chung
    Shared Core.
-   Backend FastAPI + PostgreSQL trên VPS, Caddy HTTPS.
-   Chỉ lưu tiến độ mới nhất.

**MVP Definition:** Người dùng lưu nhiều loại nội dung đọc từ Chrome
trên PC bằng một cú bấm hoặc paste URL, phân loại trong Library, cập
nhật vị trí đọc bằng URL mới, rồi mở Web App trên iPhone để tiếp tục đọc
đúng URL đã đồng bộ, mà không cần máy Mac và không phụ thuộc vào website
cụ thể.
