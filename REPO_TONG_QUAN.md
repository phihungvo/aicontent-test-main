# Tổng Quan Repo AI Content

Tài liệu này tóm tắt repo `aicontent-test`: mục tiêu sản phẩm, kiến trúc, dữ
liệu, luồng chạy, các ràng buộc quan trọng và danh sách task cần làm để hoàn
thành bài test. Repo dùng nhiều tên file, biến, cột và hàm bằng tiếng Việt
không dấu; khi viết code cần giữ nguyên các định danh đó để khớp hệ thống và bộ
test.

## 1. Mục Tiêu Sản Phẩm

Đây là ứng dụng web giúp chủ shop nhỏ sản xuất nội dung mạng xã hội đều tay.
Mục tiêu trong đề bài là giúp người dùng đăng được 10 bài Facebook mỗi ngày mà
không cần thuê thêm nhân sự.

Sản phẩm không phải landing page. Nó là một dashboard nội bộ gồm:

- Hồ sơ thương hiệu: sản phẩm, chân dung khách hàng, insight, trụ cột nội dung,
  giọng điệu và điều cấm kỵ.
- Kho bài đã đăng của chính kênh người dùng.
- Kho bài của kênh bên ngoài mà người dùng theo dõi.
- Lớp AI sinh ý tưởng, bài viết, kịch bản video và về sau là ảnh.
- Hàng đợi worker để gọi mô hình, thay vì gọi mô hình trực tiếp trong request
  web.

Đề bài chính nằm trong `TEST-BRIEF.md`. Năm route trong sidebar hiện là phần cần
làm:

| Route | Trạng thái hiện tại | Việc cần làm |
|---|---:|---|
| `/studio/de-xuat` | 404 | Đề xuất ý tưởng hôm nay |
| `/studio/bien-soan` | 404 | Từ một ý tưởng sinh bài viết, có thể sửa |
| `/studio/chuoi-bai` | 404 | Sinh chuỗi bài nối mạch, không lặp ý |
| `/studio/hang-loat` | 404 | Sinh nhiều bài một lượt, phục vụ mục tiêu 10 bài/ngày |
| `/studio/so-giong` | 404 | So sánh 4 giọng trên 4 bề mặt |

Mốc bắt buộc là `/studio/de-xuat`. Không có mốc này thì các mốc sau không có
nguồn ý tưởng để chạy.

## 2. Stack Kỹ Thuật

| Thành phần | Công nghệ |
|---|---|
| Framework web | Next.js App Router |
| UI | React 19, CSS thủ công theo từng màn |
| Auth | Auth.js / NextAuth v5, session lưu database |
| Database | PostgreSQL |
| ORM | Drizzle ORM + Drizzle Kit |
| Queue | Bảng `jobs` trong PostgreSQL, worker Node.js |
| AI runner | API runner Gemini/OpenAI hoặc runner CLI Claude/Codex trong sandbox |
| Test | `node --test`, TypeScript check, Next build |
| Package manager | npm |

Lệnh quan trọng:

```bash
npm ci
docker compose -f docker/postgres/compose.yml up -d
cp .env.example .env
npm run db:migrate
docker exec -i aicontent-test-postgres psql -U postgres -d aicontent_test < db/seed.sql
docker exec -i aicontent-test-postgres psql -U postgres -d aicontent_test < db/seed-test.sql
npm run dev
node -r dotenv/config workers/model/index.js
```

Kiểm trước khi nộp:

```bash
npx tsc --noEmit
node --test tests/
npm run build
```

## 3. Cấu Hình Môi Trường

File mẫu là `.env.example`.

Biến bắt buộc:

- `PORT`: mặc định 6980.
- `DATABASE_URL`: chuỗi kết nối của tiến trình web, dùng role `aicontent_app`.
- `DATABASE_URL_WORKER`: chuỗi kết nối của worker, dùng role `aicontent_worker`.
- `AUTH_SECRET`: khóa session Auth.js, sinh bằng `openssl rand -hex 32`.
- `AUTH_URL`: nên là `http://localhost:6980`, không dùng `127.0.0.1`.
- `AUTH_DEV_LOGIN=1`: bật cửa đăng nhập nhanh riêng cho bài test.
- `AI_PROVIDER=gemini` hoặc `openai`.
- `GEMINI_API_KEY` hoặc `OPENAI_API_KEY`.

Biến tùy chọn:

- `AI_MODEL`: ghi đè tên model API.
- `APIFY_TOKEN`: dùng khi kéo bài thật từ nền tảng ngoài.
- `ELEVENLABS_API_KEY`: dùng khi bóc phụ đề video.
- `KHO_MEDIA`: kho ảnh/video tải về, mặc định nằm ngoài source code.

Lưu ý bảo mật:

- Web app phải dùng `DATABASE_URL`, không dùng `DATABASE_URL_WORKER`.
- `.env` không được commit.
- Test `khong-lo-secret-ra-trinh-duyet` quét `.next/static` để chặn secret và
  prompt giọng lọt xuống bundle client.

## 4. Luồng Vào App

`app/layout.tsx` cài đặt metadata, font và script khởi tạo theme trước khi
hydrate.

`middleware.ts` chặn tất cả route nếu chưa đăng nhập, trừ:

- `/api/auth`
- `/api/dev-login`
- `/healthz`
- `/dang-nhap`
- asset Next/static

`next.config.js` có redirect:

- `/` -> `/templates`
- `/app/templates` -> `/templates`

Vào nhanh khi dev:

```text
http://localhost:6980/api/dev-login
```

Route này chỉ hoạt động khi `NODE_ENV !== production` và `AUTH_DEV_LOGIN=1`.
Nó tạo session database giống Auth.js, không dùng Credentials provider vì repo
có chủ ý giữ session strategy là database.

## 5. Cấu Trúc Thư Mục

| Đường dẫn | Vai trò |
|---|---|
| `app/` | UI, layouts, route handlers, server actions |
| `app/(auth)/dang-nhap` | Trang đăng nhập |
| `app/(dash)` | Dashboard sau đăng nhập: sidebar, topbar, các màn chính |
| `app/api/auth/[...nextauth]` | Route Auth.js |
| `app/api/dev-login` | Đăng nhập nhanh cho bài test |
| `app/api/media/[...duongDan]` | Phục vụ media trong kho |
| `app/healthz` | Health check công khai |
| `db/schema` | Schema Drizzle chia theo nhóm nghiệp vụ |
| `db/migrations` | Migration SQL đã generate |
| `db/rollback` | Rollback tương ứng migration |
| `db/seed.sql` | Dữ liệu tối thiểu: user, workspace, 5 trụ cột |
| `db/seed-test.sql` | Dữ liệu mẫu đầy đủ cho bài test |
| `lib/auth` | Lấy user/workspace từ session |
| `lib/data-access` | Tầng truy cập database duy nhất cho nghiệp vụ |
| `lib/brand` | Độ đầy đủ hồ sơ, đặc tả nhóm, bóc tách hồ sơ |
| `lib/model-runner` | Chọn model, prompt, API/CLI runner, parse JSON kết quả |
| `lib/queue` | Đẩy việc, claim việc, retry, thả khóa việc treo |
| `lib/keo-bai` | Kéo bài từ Apify, bóc dữ liệu, lưu asset, phụ đề |
| `lib/studio` | Hiện mới có `boc-cong-thuc.ts`; cần thêm logic bài test |
| `workers/model` | Worker gọi mô hình cho các job AI |
| `workers/keo` | Worker kéo bài nền tảng ngoài |
| `tests` | Test đơn vị, test guard, test integration nhỏ |

## 6. Các Màn UI Đã Có

### Shell Dashboard

`app/(dash)/app-shell.tsx` dùng client component để:

- Hiện sidebar theo các nhóm: Hồ sơ kênh, Nội dung, Khám phá, Hệ thống.
- Toggle drawer mobile.
- Toggle theme light/dark bằng `localStorage`.
- Hiện topbar, notice gói dùng thử và thông tin user seed đang hard-code.

Năm link `/studio/*` đã có trong sidebar nhưng chưa có page.

### `/templates`

Màn kho mẫu nội dung. Hiện là UI tĩnh với sample trong
`content-template-samples.ts`. Trang này là default destination của `/`.

### `/brand`

Tổng quan hồ sơ thương hiệu:

- Đọc toàn bộ hồ sơ qua `docToanBoHoSo()`.
- Tính độ đầy đủ bằng `tinhDoDayDu()`.
- Nếu dưới 60% thì hiện trạng thái chặn đề xuất.
- Link sang các nhóm: sản phẩm, chân dung, insight, trụ cột, giọng điệu.

### `/brand/[nhom]`

Màn CRUD dùng chung cho 4 nhóm danh sách:

- `san-pham`
- `chan-dung`
- `insight`
- `tru-cot`

Đặc tả field nằm ở `lib/brand/dac-ta-nhom.ts`.

### `/brand/giong-dieu`

Màn riêng cho `brand_profiles`: mô tả, giọng điệu, điều cấm kỵ, phông chữ.

### `/brand/dan-van-ban`

Màn dán văn bản để AI bóc tách hồ sơ. Quan trọng: AI chỉ tạo bản nháp xem
trước, không ghi database trực tiếp. Người dùng phải duyệt rồi mới lưu.

### `/bai-da-dang`

Màn danh sách bài đã đăng của chính kênh:

- Đọc `contents` với `trangThai='da_dang'`.
- Đọc asset kèm theo từ `assets`.
- Đọc số liệu bóc từ `bai_keo_tho`.
- Bảng client có lọc theo cột, mở rộng dòng để xem ảnh/video/phụ đề.

### `/kenh-ngoai`

Màn danh sách bài của kênh người dùng đang theo dõi:

- Đọc `trend_signals`.
- Lọc theo người dùng qua `theo_doi_cua_toi`.
- Không lọc `da_dung` vì đây là màn tra cứu, bài đã dùng vẫn phải xem lại được.

### `/cai-dat/kenh`

Màn cài đặt kênh của mình và kênh theo dõi:

- Lưu kênh Facebook cá nhân vào `kenh_dang`.
- Kéo bài đã đăng về qua Apify nếu có token.
- Thêm/tắt/bật theo dõi kênh ngoài qua `kenh_theo_doi` và `theo_doi_cua_toi`.
- Kéo ngay kênh theo dõi về `trend_signals`.

## 7. Miền Dữ Liệu Nghiệp Vụ

### Workspace Là Biên Bảo Mật Chính

Mỗi bảng nghiệp vụ đều có `workspace_id`. Tầng truy cập dữ liệu bắt buộc nhận
`workspaceId` và mọi query phải lọc theo workspace.

Nguồn hợp lệ của `workspaceId` trong app là:

```ts
createRepo(await workspaceHienTai())
```

Không được lấy `workspaceId` từ body, form, query string hoặc prop.

### Trục Cô Lập Thứ Hai: User

Danh sách kênh theo dõi là riêng từng người trong cùng workspace. Vì vậy các
hàm liên quan `theo_doi_cua_toi` không nhận `userId: string` thường, mà nhận
`NguoiDungTuPhien` tạo bởi `nguoiDungHienTai()`.

### Auth

Bảng Auth.js:

- `users`
- `oauth_accounts`
- `sessions`
- `verification_tokens`

Bảng workspace:

- `workspaces`
- `workspace_members`

`auth.ts` là ngoại lệ được import `db` trực tiếp vì DrizzleAdapter bắt buộc
nhận Drizzle instance. Nghiệp vụ còn lại phải qua `lib/data-access`.

### Hồ Sơ Thương Hiệu

Bảng:

- `brand_profiles`: mô tả, giọng điệu, điều cấm kỵ, màu sắc, phông chữ, độ đầy đủ.
- `products`: sản phẩm/dịch vụ.
- `personas`: chân dung khách hàng.
- `insights`: insight có bằng chứng.
- `content_pillars`: trụ cột nội dung, tỉ lệ mục tiêu, khóa không tự giảm.

Độ đầy đủ được tính lại lúc đọc, không tin vào cột `brand_profiles.do_day_du`.
Ngưỡng để đề xuất là 60%.

### Nội Dung

Bảng chính:

- `kenh_dang`: kênh của chính workspace.
- `kenh_theo_doi`: kênh của người khác mà workspace có thể theo dõi.
- `theo_doi_cua_toi`: user nào đang theo dõi kênh nào.
- `trend_signals`: bài/tín hiệu từ kênh ngoài.
- `ideas`: ý tưởng do máy đề xuất, xu hướng hoặc người tự nhập.
- `contents`: bài viết/kịch bản/bản nháp/đã đăng.
- `assets`: ảnh/video/file gắn với content.
- `bai_keo_tho`: bản thô của bài đã kéo về từ kênh mình.
- `quality_scores`: điểm chất lượng nội dung.
- `che_do_xem`: view tùy biến cho bảng nội dung.

Enum đáng chú ý:

- `be_mat`: `fanpage`, `ho_so_ca_nhan`, `tiktok`, `zalo`.
- `trang_thai_noi_dung`: `y_tuong`, `ban_nhap`, `da_cham`, `san_sang`,
  `da_dang`, `dang_theo_doi`, `da_chot_ket_qua`, `da_bo`.
- `dang_bai`: `chu`, `anh_chu`, `kich_ban_quay`.
- `nguon_y_tuong`: `may-de-xuat`, `xu-huong`, `nguoi-tu-nhap`.

### Đo Lường

Bảng:

- `metric_snapshots`: bản chụp số liệu thô bất biến.
- `comments_raw`: bình luận thô, đã ẩn danh tác giả.
- `comment_classifications`: phân loại ý định bình luận.
- `effectiveness_scores`: điểm hiệu quả theo formula version.

Nguyên tắc lớn: số liệu thô bất biến, không update để tránh phá lịch sử.

### Vận Hành

Bảng:

- `jobs`: hàng đợi việc nền.
- `model_runs`: nhật ký gọi mô hình.
- `cost_log`: sổ chi phí.
- `credit_ledger`: số dư tín dụng.

`jobs.idempotency_key` chặn việc trùng ở tầng database.

## 8. Tầng Truy Cập Dữ Liệu

`lib/data-access/index.ts` là cổng duy nhất giữa web app và database.

Hàm chính:

- `createRepo(workspaceId)`: tạo repo cho request hiện tại.
- `trongGiaoDich(workspaceId, viec)`: chạy nhiều lệnh ghi trong một transaction.
- `taoRepo(ketNoi, workspaceId)`: ghép repo trên một connection bất kỳ, dùng nội bộ.

Repo hiện có:

- `workspaces`
- `workspaceMembers`
- `jobs`
- `modelRuns`
- `costLog`
- `contents`
- `hoSo`
- `sanPham`
- `chanDung`
- `insight`
- `truCot`
- `yTuong`
- `kenh`
- `baiKeoTho`
- `asset`
- `kenhTheoDoi`
- `theoDoiCuaToi`
- `tinHieuXuHuong`

Test `data-access-guard.test.cjs` quét code để chặn:

- Import `db` ngoài các file được phép.
- Tự tạo `Pool`, `Client`, `drizzle`.
- Gọi `createRepo()` với đối số từ body/query/prop.
- Gọi `createSystemRepo()` sai chỗ.
- Tạo `NguoiDungTuPhien` sai chỗ.

Khi thêm code mới trong `app/`, pattern đúng là:

```ts
const repo = createRepo(await workspaceHienTai());
```

Khi cần user hiện tại cho kênh theo dõi:

```ts
const nguoi = await nguoiDungHienTai();
```

## 9. Model Runner Và Worker

Web app không gọi Gemini/OpenAI/Claude/Codex trực tiếp. Web app gọi:

```js
chayNhiemVu({
  nhiemVu,
  duLieuVao,
  moHinh: 'auto',
  khongGianLamViec: workspaceId,
})
```

`chayNhiemVu()`:

- Chọn model bằng `chonMoHinh()`.
- Ghép `nhiemVu` và `moHinh` vào payload job.
- Sinh `idempotency_key`.
- Insert vào `jobs`.
- Mặc định chờ kết quả tối đa 360 giây, polling mỗi giây.

Worker `workers/model/index.js`:

- Claim job bằng `FOR UPDATE SKIP LOCKED`.
- Chạy `thucThiNhiemVu()`.
- Ghi `model_runs` và `cost_log`.
- Update job `xong` hoặc `loi`.
- Retry lỗi hạ tầng theo lịch 1 phút, 5 phút, 30 phút.
- Thả khóa job treo sau 15 phút.

Runner hiện có:

- `runner-api.js`: dùng Gemini/OpenAI, là đường phù hợp cho bài test.
- `runner-claude.js`: gọi Claude CLI trong sandbox.
- `runner-codex.js`: gọi Codex CLI trong sandbox.

Prompt nằm ở `lib/model-runner/loi-nhac-theo-nhiem-vu.js`.
Ba nhiệm vụ bài test còn TODO:

- `viet-bai`
- `viet-kich-ban`
- `de-xuat-y-tuong`

`lib/model-runner/loi-nhac-xu-huong.js` còn `KHOI_Y_TUONG_TU_XU_HUONG = ''`,
cần viết để ép mô hình chỉ thấy chủ đề/công thức kể, không thấy nguyên văn bài
người khác.

## 10. Dữ Liệu Seed

`db/seed.sql` tạo:

- User `seed@aicontent.local`.
- Workspace `SANG 5M STUDIO`.
- Membership chủ sở hữu.
- 5 trụ cột nội dung:
  - `Xay long tin`
  - `Chi meo huu ich`
  - `Bang chung ket qua`
  - `Chao ban`
  - `Bat xu huong`

`db/seed-test.sql` tạo thêm 4 nguồn đầu vào cho bài test:

- Hồ sơ thương hiệu và 3 chân dung khách hàng.
- 3 insight có bằng chứng.
- 6 bài đã đăng của kênh mình trong `contents`.
- 2 kênh theo dõi và 6 bài kênh ngoài trong `trend_signals`.

Sau khi seed đúng thứ tự, `/bai-da-dang` có 6 bài và `/kenh-ngoai` có 6 bài.

## 11. Luồng Kéo Bài Từ Nền Tảng Ngoài

Module `lib/keo-bai` dùng cho phần ngoài bài test, nhưng có sẵn để hiểu repo:

- Chuẩn hóa URL theo bề mặt.
- Gọi Apify actor.
- Bóc bài fanpage/TikTok/hồ sơ cá nhân.
- Lưu bài kéo về.
- Lưu media/asset.
- Trần chi phí Apify.
- Bóc phụ đề video bằng ElevenLabs.

Lượt kéo kênh theo dõi có nguyên tắc: actor chạy xong là tiền đã mất, nên lỗi ở
một kênh phải thành cảnh báo, không ném lỗi làm queue retry cả lượt và tốn tiền
lần nữa.

## 12. Các Ràng Buộc Bắt Buộc Của Bài Test

### 12.1 Mọi Query Đi Qua `lib/data-access`

Không import `db` trong page/action/module nghiệp vụ. Nếu cần query mới, thêm
hàm repo vào `lib/data-access/*`, rồi gọi qua `createRepo(await workspaceHienTai())`.

### 12.2 Không Bịa Trụ Cột Và Chân Dung

Model có thể trả về tên sai. Code phải đối chiếu với danh sách có thật trong
workspace. Nếu không khớp thì đặt `null`, không tự thêm tên mới vào hệ thống.

### 12.3 Học Cách Kể, Không Chép Bài

Khi dùng bài kênh ngoài:

- Được dùng chủ đề, kiểu hook, độ dài, có CTA hay không.
- Không được đưa nguyên văn `trend_signals.noiDung` vào prompt đề xuất ý tưởng.
- Ràng buộc này phải ép bằng code, không chỉ viết trong prompt.
- Mỗi ý tưởng sinh từ tham khảo phải trỏ về đúng `trendSignalId`.

### 12.4 Web Không Gọi Model Provider Trực Tiếp

Mọi tác vụ AI đi qua `chayNhiemVu()`, sau đó worker mới gọi runner.

## 13. Hợp Đồng Code Cần Thêm Cho Mốc 1

Theo `TEST-BRIEF.md`, cần tạo `lib/studio/kieu.ts`:

```ts
export type YTuongDeXuat = {
  tieuDe: string;
  truCot: string | null;
  chanDung: string | null;
  gocTiepCan: string | null;
  cauMoDau: string | null;
  lyDoDeXuat: string | null;
  beMat: BeMat;
  khamPha: boolean;
};
```

Cần tạo `lib/studio/de-xuat.ts` export đúng các tên:

```ts
export const TI_LE_KHAM_PHA = 0.2;

export type ThamSoDeXuat = {
  workspaceId: string;
  beMat: BeMat;
  soLuong: number;
};

export type TruCotMucTieu = { ten: string; tiLeMucTieu: number | null };

export function donKetQuaDeXuat(tho: unknown, /* ... */): YTuongDeXuat[];

export function raiTheoTruCot(
  yTuongTho: YTuongDeXuat[],
  truCotMucTieu: TruCotMucTieu[],
  soLuong: number,
): YTuongDeXuat[];

export async function deXuatYTuong(
  thamSo: ThamSoDeXuat,
): Promise<KetQuaStudio<YTuongDeXuat[]>>;
```

Nên tạo thêm type chung `KetQuaStudio<T>` trong `lib/studio/kieu.ts`, vì nhiều
mốc sau cũng cần trả `{ ok, duLieu }` hoặc `{ ok: false, loi }`.

## 14. Roadmap Task Cần Làm

### Task 1: Nền Type Và Kết Quả Studio

Độ ưu tiên: bắt buộc.

Việc cần làm:

- Tạo `lib/studio/kieu.ts`.
- Định nghĩa `BeMat` nếu chưa có type export dùng được từ schema.
- Định nghĩa `YTuongDeXuat`.
- Định nghĩa `KetQuaStudio<T>`.
- Nếu cần, thêm type cho bài viết, kịch bản, cảnh video.

Tiêu chí xong:

- TypeScript import được từ các page/action.
- Tên field khớp đúng hợp đồng đề bài.

### Task 2: Viết Prompt Cho `de-xuat-y-tuong`

Độ ưu tiên: bắt buộc.

Sửa:

- `lib/model-runner/loi-nhac-theo-nhiem-vu.js`
- `lib/model-runner/loi-nhac-xu-huong.js`

Việc cần làm:

- Viết chỉ dẫn cho `de-xuat-y-tuong`.
- Giữ nguyên `truongBatBuoc: ['yTuong']`.
- Giữ nguyên dòng `Cau truc`.
- Viết `KHOI_Y_TUONG_TU_XU_HUONG` để nói rõ chỉ học chủ đề/công thức kể.
- Không đưa nội dung gốc bài kênh ngoài vào prompt.

Tiêu chí xong:

- Model trả JSON có `yTuong`.
- Prompt không phụ thuộc vào văn bản gốc của kênh ngoài.

### Task 3: Viết Hàm `donKetQuaDeXuat`

Độ ưu tiên: bắt buộc.

Việc cần làm:

- Nhận kết quả thô của model.
- Chỉ lấy mảng `yTuong`.
- Ép field string rỗng thành `null`.
- Ép `beMat` về một trong 4 giá trị hợp lệ; sai thì dùng bề mặt yêu cầu.
- Đối chiếu `truCot` với danh sách trụ cột có thật.
- Đối chiếu `chanDung` với danh sách chân dung có thật.
- Trụ cột/chân dung sai thì đặt `null`.
- Cắt bớt số lượng và bỏ dòng quá rỗng.
- Bảo toàn `khamPha` boolean.

Tiêu chí xong:

- Không bao giờ trả trụ cột/chân dung bịa.
- Hàm thuần, test được bằng dữ liệu tay.

### Task 4: Viết Hàm `raiTheoTruCot`

Độ ưu tiên: bắt buộc.

Việc cần làm:

- Nhận danh sách ý tưởng thô, danh sách trụ cột mục tiêu và `soLuong`.
- Tính số slot theo `tiLeMucTieu`.
- Rãnh trụ cột có `null` tỉ lệ thì chia phần còn lại hợp lý.
- Ưu tiên đủ số lượng tổng.
- Không bỏ hết ý tưởng khám phá; giữ khoảng `TI_LE_KHAM_PHA = 0.2` nếu có đủ.
- Không lặp ý tưởng trùng tiêu đề/góc tiếp cận.

Tiêu chí xong:

- Trả đúng `soLuong` nếu đầu vào đủ.
- Phân bổ gần tỉ lệ mục tiêu.
- Test cần có case làm tròn, thiếu ý tưởng một trụ cột, trụ cột null tỉ lệ.

### Task 5: Viết `deXuatYTuong`

Độ ưu tiên: bắt buộc.

Luồng đề xuất nên như sau:

1. Tạo repo bằng `createRepo(workspaceId)`.
2. Đọc `hoSo`, `sanPham`, `chanDung`, `insight`, `truCot`.
3. Kiểm tra độ đầy đủ bằng `kiemTraDeXuat()`.
4. Đọc lịch sử bài đã đăng gần đây từ `repo.contents`.
5. Đọc bài kênh ngoài của người dùng nếu có user context; nếu signature chỉ có
   `workspaceId` thì cần thêm tham số tùy chọn hoặc xử lý ở action.
6. Bóc công thức bài kênh ngoài nếu cần bằng `bocCongThucChoBaiMoi()`.
7. Dùng payload sạch cho model:
   - Hồ sơ thương hiệu.
   - Danh sách trụ cột và tỉ lệ.
   - Danh sách chân dung.
   - Insight.
   - Lịch sử bài đã đăng của mình có thể gồm câu mở đầu/góc tiếp cận, không cần
     gom quá nhiều.
   - Bài kênh ngoài chỉ gồm `id`, `congThuc`, `diemLienQuan`, chủ đề; không gom
     `noiDung` gốc.
8. Gọi `chayNhiemVu({ nhiemVu: 'de-xuat-y-tuong', ... })`.
9. Dọn kết quả.
10. Rải theo trụ cột.
11. Trả `KetQuaStudio<YTuongDeXuat[]>`.

Tiêu chí xong:

- Nếu hồ sơ dưới 60%, trả lỗi rõ cho UI.
- Kết quả bám trụ cột/chân dung có thật.
- Có ít nhất vài ý tưởng `khamPha`.
- Ý tưởng từ xu hướng có thể trỏ về nguồn `trendSignalId` khi lưu.

### Task 6: Lưu Ý Tưởng Vào Database

Độ ưu tiên: bắt buộc cho UI dùng thật.

Việc cần làm:

- Thêm server action cho `/studio/de-xuat`.
- Khi người dùng bấm lưu, map `truCot` tên -> `pillarId`.
- Map `chanDung` tên -> `personaId`.
- Nếu có `trendSignalId`, kiểm `repo.tinHieuXuHuong.coThuocWorkspace()`.
- Insert vào `ideas` qua `repo.yTuong.tao()`.
- Nếu dùng trend signal, gọi `repo.tinHieuXuHuong.danhDauDaDung([id])`.

Tiêu chí xong:

- Không tin ID/tên từ client nếu chưa đối chiếu lại server.
- Reload trang vẫn thấy ý tưởng đã lưu nếu có danh sách.

### Task 7: UI `/studio/de-xuat`

Độ ưu tiên: bắt buộc.

Việc cần làm:

- Tạo `app/(dash)/studio/de-xuat/page.tsx`.
- Tạo client component nếu cần form tương tác.
- Cho chọn bề mặt: fanpage, hồ sơ cá nhân, TikTok, Zalo.
- Cho chọn số lượng.
- Nút "Sinh đề xuất".
- Hiện loading và lỗi.
- Hiện card/table ý tưởng:
  - tiêu đề
  - trụ cột
  - chân dung
  - góc tiếp cận
  - câu mở đầu
  - lý do đề xuất
  - khám phá hay không
  - nguồn tham khảo nếu có
- Nút lưu ý tưởng.
- Link sang `/studio/bien-soan` cho ý tưởng đã lưu.

Tiêu chí xong:

- Chạy được với seed.
- Không 404.
- UX đủ dùng trong video demo.

### Task 8: Viết Prompt `viet-bai`

Độ ưu tiên: mốc 2.

Việc cần làm:

- Sửa TODO trong `loi-nhac-theo-nhiem-vu.js`.
- Giữ `truongBatBuoc: ['tieuDe', 'noiDung']`.
- Giữ cấu trúc JSON có `tieuDe`, `noiDung`, `hashtag`.
- Giải thích cách dùng `epDoDai` và `mach`.
- Ghép giọng bề mặt sẽ được hàm hiện có làm qua `bienThe`.

Tiêu chí xong:

- Bài viết bám ý tưởng, brand voice, bề mặt.
- Không vượt khoảng từ theo bề mặt.

### Task 9: UI Và Logic `/studio/bien-soan`

Độ ưu tiên: mốc 2.

Việc cần làm:

- Tạo page và action.
- Cho chọn ý tưởng đã lưu hoặc nhập nhanh ý tưởng.
- Gọi `chayNhiemVu({ nhiemVu: 'viet-bai', bienThe: beMat })`.
- Lưu kết quả vào `contents` với `trangThai='ban_nhap'`.
- Cho sửa `tieuDe`, `noiDung`, `hashtag`.
- Có nút lưu bản nháp.

Tiêu chí xong:

- Từ một ý tưởng sinh được bài hoàn chỉnh.
- Nội dung được lưu database.

### Task 10: Viết Prompt `viet-kich-ban`

Độ ưu tiên: mốc 3.

Việc cần làm:

- Sửa TODO `viet-kich-ban`.
- Giữ `truongBatBuoc: ['tieuDe', 'phanCanh']`.
- Giữ cấu trúc `phanCanh`.
- Ép mỗi phân cảnh có thời lượng, hình ảnh, lời thoại.
- Không trả một đoạn văn liên tục.

Tiêu chí xong:

- Kịch bản có cấu trúc phân cảnh rõ để quay video.

### Task 11: UI Kịch Bản Quay

Độ ưu tiên: mốc 3.

Có thể làm trong `/studio/bien-soan` hoặc page riêng nếu cần.

Việc cần làm:

- Cho chọn ý tưởng.
- Gọi `viet-kich-ban`.
- Hiện bảng phân cảnh.
- Lưu vào `contents` với `dangBai='kich_ban_quay'`.

### Task 12: `/studio/chuoi-bai`

Độ ưu tiên: mốc 4 phụ.

Việc cần làm:

- Cho chọn chủ đề/ý tưởng gốc.
- Tạo nhiều bài có chung `chuoiId`.
- Mỗi bài có `thuTuTrongChuoi`.
- Truyền `mach` vào `viet-bai` để bài sau không lặp bài trước.
- Lưu vào `contents`.

Ràng buộc DB đã có:

- `chuoiId` và `thuTuTrongChuoi` phải cùng null hoặc cùng có giá trị.
- Unique `(workspaceId, chuoiId, thuTuTrongChuoi)` chặn trùng thứ tự.

### Task 13: `/studio/hang-loat`

Độ ưu tiên: mốc 4.

Việc cần làm:

- Cho chọn số bài cần sinh.
- Có thể dùng ý tưởng đã có hoặc gọi đề xuất trước.
- Đẩy nhiều job viết bài.
- Giới hạn số lượng hợp lý để không treo request.
- Hiện trạng thái job nếu chọn `cho: false`.
- Lưu kết quả thành nhiều `contents`.

Tiêu chí xong:

- Sinh được nhiều bài một lượt, hướng tới 10 bài/ngày.

### Task 14: `/studio/so-giong`

Độ ưu tiên: mốc 5 phụ.

Việc cần làm:

- Cùng một ý tưởng, gọi `viet-bai` với 4 `bienThe`.
- Hiện 4 cột: fanpage, hồ sơ cá nhân, TikTok, Zalo.
- So sánh độ dài, hook, CTA.
- Cho lưu một hoặc nhiều bản.

Tiêu chí xong:

- Nhìn rõ khác biệt giọng giữa 4 bề mặt.

### Task 15: Sinh Ảnh

Độ ưu tiên: mốc cuối.

Schema đã có `assets` và enum `sinh-anh` trong nhiệm vụ/model/job, nhưng runner
hiện chưa có implement ảnh thật.

Việc cần làm:

- Quyết định provider sinh ảnh.
- Tạo worker/runner riêng nếu provider trả file/binary.
- Lưu asset vào kho media, ghi `assets`.
- Gắn asset với `contents`.
- UI hiện preview ảnh.

Tiêu chí xong:

- Bài có asset ảnh mở được qua `/api/media/...`.

## 15. Test Nên Thêm Khi Làm Bài Test

Nên thêm test gắn với từng hàm thuần:

- `tests/de-xuat-y-tuong.test.cjs` hoặc `.mjs`:
  - `donKetQuaDeXuat` bỏ trụ cột/chân dung không có thật.
  - `donKetQuaDeXuat` chuẩn hóa string rỗng về `null`.
  - `raiTheoTruCot` trả đúng số lượng.
  - `raiTheoTruCot` tôn trọng tỉ lệ mục tiêu.
  - `raiTheoTruCot` giữ khoảng 20% khám phá nếu có đủ.
- Test prompt không nhận nội dung gốc kênh ngoài:
  - Hàm dựng payload cho đề xuất không có field `noiDung` của `trend_signals`.
- Test server action:
  - Lưu ý tưởng đối chiếu `trendSignalId` thuộc workspace.

Chạy lại toàn bộ:

```bash
npx tsc --noEmit
node --test tests/
npm run build
```

## 16. Rủi Ro Và Điểm Cần Để Ý

- Các route `/studio/*` chưa tồn tại, nên đây là phần trọng tâm.
- Prompt `viet-bai`, `viet-kich-ban`, `de-xuat-y-tuong` còn TODO.
- `KHOI_Y_TUONG_TU_XU_HUONG` đang rỗng, trong khi ràng buộc không chép bài kênh
  ngoài là điểm chấm nặng.
- `ideasRepo` hiện chỉ là CRUD chung; có thể cần thêm query riêng để list ý
  tưởng kèm tên trụ cột/chân dung.
- `contentsRepo` hiện tối thiểu; nếu làm biên soạn/hàng loạt có thể cần query
  list bản nháp, list theo `ideaId`, list theo `chuoiId`.
- `chayNhiemVu()` mặc định chờ đến 360 giây. Với UI sinh hàng loạt, nên đẩy job
  `cho: false` và polling, tránh request quá lâu.
- `runner-api.js` dùng network. Khi thiếu `AI_PROVIDER` hoặc API key, job sẽ lỗi
  cấu hình chứ app vẫn chạy.
- Không được import prompt model-runner vào client component, vì test sẽ chặn lộ
  prompt giọng.
- Khi làm UI client, server action nên là nơi gọi AI/database; client chỉ gửi
  tham số tối thiểu.
- Nếu thêm migration, dùng `drizzle-kit generate` và `migrate`, không dùng
  `drizzle-kit push`.

## 17. Thứ Tự Làm Để Tối Ưu Điểm

Nếu chỉ có ít thời gian, nên làm theo thứ tự:

1. `lib/studio/kieu.ts`.
2. `lib/studio/de-xuat.ts` với `donKetQuaDeXuat`, `raiTheoTruCot`,
   `deXuatYTuong`.
3. Prompt `de-xuat-y-tuong` và `KHOI_Y_TUONG_TU_XU_HUONG`.
4. Test cho hai hàm thuần.
5. Page `/studio/de-xuat` có form sinh và hiện kết quả.
6. Server action lưu ý tưởng vào `ideas`.
7. Prompt `viet-bai`.
8. Page `/studio/bien-soan`.
9. Prompt/page kịch bản quay.
10. Hàng loạt, chuỗi bài, so giọng, ảnh.

Hoàn thành chắc mốc 1 và 2 tốt hơn chạm cả 5 mốc nhưng không cái nào đúng.

## 18. Bản Đồ File Nên Mở Khi Code Tiếp

Khi làm đề xuất:

- `TEST-BRIEF.md`
- `lib/brand/do-day-du.ts`
- `lib/data-access/index.ts`
- `lib/data-access/contents.ts`
- `lib/data-access/tin-hieu-xu-huong.ts`
- `lib/studio/boc-cong-thuc.ts`
- `lib/model-runner/loi-nhac-theo-nhiem-vu.js`
- `lib/model-runner/loi-nhac-xu-huong.js`

Khi làm UI:

- `app/(dash)/app-shell.tsx`
- `app/(dash)/studio/studio.css`
- `app/(dash)/brand/brand.css`
- `app/(dash)/templates/page.tsx`
- `app/(dash)/bai-da-dang/page.tsx`

Khi làm data write:

- `app/(dash)/brand/actions.ts`
- `app/(dash)/cai-dat/kenh/actions.ts`
- `lib/data-access/crud-theo-workspace.ts`
- `lib/auth/current-workspace.ts`
- `lib/auth/nguoi-dung-tu-phien.ts`

Khi debug worker/model:

- `lib/model-runner/index.js`
- `lib/model-runner/thuc-thi-nhiem-vu.js`
- `lib/model-runner/runner-api.js`
- `workers/model/index.js`
- `workers/model/vong-lap.js`
- `lib/queue/enqueue.js`
- `lib/queue/claim.js`

## 19. Kết Luận Ngắn

Repo đã có nền móng chắc: auth, workspace isolation, schema, seed, dashboard
shell, hồ sơ thương hiệu, bảng bài đã đăng, bảng kênh ngoài, kéo bài, queue và
model runner. Phần còn thiếu đúng trọng tâm đề bài là module Studio: sinh ý
tưởng, sinh bài, sinh kịch bản, sinh hàng loạt, so giọng và ảnh.

Hướng làm đúng là tiếp tục bám vào các lớp đã có:

- UI/server action trong `app/(dash)/studio`.
- Nghiệp vụ thuần trong `lib/studio`.
- Database qua `lib/data-access`.
- AI qua `chayNhiemVu()`.
- Prompt trong `lib/model-runner`.
- Test cho mỗi hàm thuần và mỗi ràng buộc bảo mật/không chép bài.
