# AI Sales Analytics

Website phân tích dữ liệu kinh doanh bất động sản bằng câu hỏi tiếng Việt. Dự án dùng Next.js App Router, TypeScript, Tailwind CSS, Supabase PostgreSQL, Gemini, Recharts và Vercel.

## Kiến trúc mục tiêu

```text
Browser (Next.js UI)
  ├── Dashboard / AI hỏi đáp / lịch sử / xuất CSV
  └── Next.js Route Handlers (server only)
        ├── Input validation + query orchestration
        ├── Gemini: câu hỏi → JSON intent + SELECT SQL + chart hint
        ├── SQL validator: chỉ SELECT/CTE đọc dữ liệu, allowlist bảng/cột, giới hạn kết quả
        ├── PostgreSQL pool (1 client / instance) → Supabase Transaction Pooler
        └── Result formatter → bảng/KPI/biểu đồ
```

### Luồng AI

1. API nhận câu hỏi tiếng Việt và kiểm tra đầu vào.
2. Server đọc schema thật của `public.sales` từ `information_schema.columns` (không suy đoán kiểu cột); schema được cache 60 giây và có thể làm mới.
3. Prompt gửi Gemini kèm schema, từ điển nghiệp vụ và ví dụ SQL.
4. Gemini trả JSON có intent, SQL, lời giải thích và gợi ý biểu đồ.
5. Backend kiểm tra chỉ có một truy vấn đọc, chỉ dùng bảng/cột cho phép; gắn giới hạn dòng và timeout.
6. PostgreSQL thực thi câu SQL đã kiểm tra; backend định dạng dữ liệu, frontend xác nhận kiểu dữ liệu trước khi vẽ biểu đồ.

**Quy tắc doanh thu:** mọi chỉ số doanh thu dùng `SUM(gia_tri_giao_dich_vnd)` với `ngay_ky_hd IS NOT NULL`; kỳ tháng/năm dựa trên `ngay_ky_hd`. Không dùng giá chào, ngân sách, tiền cọc hoặc hoa hồng dự kiến thay cho doanh thu. Thống kê khách hàng, pipeline, lead và lịch hẹn không tự động thêm điều kiện ngày ký.

## Cấu trúc thư mục dự kiến

```text
src/
  app/                 # App Router pages, layouts, API routes
  components/          # Sidebar, KPI cards, charts, query result UI
  config/              # Runtime configuration and business rules
  hooks/               # UI hooks and local query history
  lib/
    db/                # Pool and live schema reader
    ai/                # Gemini client, prompt, SQL generation
    sql/               # Validation and execution policy
    format/            # Result/chart/CSV formatting
  services/            # Dashboard and analytics use cases
  types/               # Shared request/response types
public/                # Static assets
docs/                  # Architecture and evaluation notes
```

## Thư viện dự kiến

| Mục đích | Gói |
|---|---|
| Web / API | `next`, `react`, `react-dom` |
| UI / CSS | `tailwindcss`, `@tailwindcss/postcss`, `lucide-react` |
| Biểu đồ | `recharts` |
| PostgreSQL | `pg` |
| AI | `@google/genai` (Google GenAI SDK) |
| Input/schema validation | `zod` |
| Giữ module backend khỏi client bundle | `server-only` |

`package.json` dùng npm `latest` tags ở giai đoạn khởi tạo để npm lấy phiên bản ổn định tương thích hiện tại. Sau lần cài đặt đầu, `package-lock.json` sẽ khóa phiên bản đã phân giải và cần commit lên GitHub.

## Biến môi trường

Sao chép `.env.example` thành `.env.local` rồi điền:

| Biến | Dùng cho |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string của Supabase Pooler; chỉ dùng ở server |
| `GEMINI_API_KEY` | Khóa Google AI Studio; không đưa lên frontend |
| `GEMINI_MODEL` | Tên model Gemini cấu hình được |
| `DB_SCHEMA` | Schema PostgreSQL; mặc định `public` |
| `QUERY_MAX_ROWS` | Giới hạn hàng trả về; mặc định 500 |

Không commit `.env.local`. Chỉ lưu secrets trong Vercel Environment Variables khi triển khai.

## Giai đoạn triển khai

1. Khởi tạo dự án và khung giao diện.
2. Kết nối Supabase, đọc schema sống từ `information_schema`.
3. Xây Gemini Text-to-SQL và prompt nghiệp vụ.
4. Xây API xác thực, kiểm tra, giới hạn và thực thi SQL.
5. Hoàn thiện UI AI hỏi đáp và kết quả.
6. Xây dashboard bằng dữ liệu thật.
7. Lịch sử localStorage và xuất CSV.
8. Bộ tối thiểu 30 câu hỏi, đánh giá SQL/kết quả/thời gian và sửa lỗi.
9. Hướng dẫn đưa lên GitHub, cấu hình Vercel và kiểm tra production.

## Giai đoạn 1 — chạy khung dự án

Yêu cầu Node.js **20.9 trở lên** và npm. Mở thư mục dự án trong VS Code, sau đó chạy:

```bash
cp .env.example .env.local
npm install
npm run dev
```

Mở `http://localhost:3000`. Kết quả mong đợi là trang khởi tạo AI Sales Analytics có lộ trình 9 giai đoạn và nguyên tắc doanh thu. Ở giai đoạn này trang chưa truy vấn database hay Gemini.

Nếu `npm install` lỗi, kiểm tra `node -v` (phải từ 20.9), `npm -v`, kết nối mạng và chạy lại `npm install`. Nếu cổng 3000 đang dùng, dừng tiến trình cũ hoặc chạy `npm run dev -- --port 3001`.

## Giai đoạn 2 — kết nối PostgreSQL và đọc schema

1. Trong Supabase, chọn **Connect → Transaction pooler** và sao chép connection URI có cổng `6543`.
2. Điền URI vào `DATABASE_URL` trong `.env.local`; giữ `sslmode=require`. Không gửi URI này qua chat hoặc commit lên GitHub.
3. Khởi động lại dev server bằng `npm run dev`.
4. Mở `http://localhost:3000/api/system/database`. Thành công trả JSON `ok: true`, tên bảng `sales`, số cột và kiểu dữ liệu đọc từ database thật.
5. Dùng `http://localhost:3000/api/system/database?refresh=1` để đọc lại schema ngay sau khi cấu trúc bảng thay đổi.

Endpoint kiểm tra chỉ trả metadata của bảng, không trả dữ liệu khách hàng. Schema được cache 60 giây; lỗi trả mã và hướng dẫn an toàn, không để lộ URI.

### Xử lý lỗi kết nối

- `DATABASE_URL_MISSING`: điền URI vào `.env.local` và khởi động lại server.
- `DATABASE_SSL_REQUIRED`: URI cần có `?sslmode=require` (hoặc `&sslmode=require` nếu đã có query parameters).
- `SALES_TABLE_NOT_FOUND`: kiểm tra bảng tên chính xác `sales` và giá trị `DB_SCHEMA`.
- `DATABASE_AUTH_FAILED`: mật khẩu trong URI không đúng.
- `DATABASE_HOST_NOT_FOUND`: hostname trong URI sai hoặc DNS chưa phân giải được.
- `DATABASE_UNAVAILABLE`: kiểm tra password, host, port `6543` và URI Transaction Pooler trong Supabase.

Ứng dụng dùng pool cục bộ một kết nối cho môi trường serverless. Supabase hướng dẫn dùng Transaction Pooler cho hàm serverless; cổng mặc định là 6543 và `sslmode=require` bật mã hóa kết nối. https://supabase.com/docs/guides/database/connecting-to-postgres

## Database và triển khai

Vercel cần cùng các biến môi trường trong Project Settings → Environment Variables. GitHub/Vercel deploy sẽ được hướng dẫn ở giai đoạn 9.


## Giai đoạn 3 — Gemini đề xuất SQL

Bản ZIP giai đoạn 3 chứa toàn bộ dự án và các bản sửa giai đoạn 2. Giữ lại `.env.local` từ thư mục đang chạy; không ghi đè cấu hình cá nhân bằng `.env.example`. Nếu đã dùng `DB_SSL_VERIFY=false` cho demo, giữ giá trị đó khi chuyển thư mục.

1. Lấy Gemini API key trong Google AI Studio: https://aistudio.google.com/apikey
2. Điền `GEMINI_API_KEY` vào `.env.local`. Đặt `GEMINI_MODEL` theo model đang được phép dùng trong AI Studio (ví dụ `gemini-3.8-flash`). Quota miễn phí phụ thuộc model và tài khoản; lỗi 429 cần kiểm tra hạn mức.
3. Chạy `npm install`, sau đó `npm run dev`.
4. Mở http://localhost:3000/ai-preview và gửi câu hỏi.
5. Thành công: hiện SQL đề xuất, lời giải thích, loại biểu đồ và thời gian phản hồi. Chưa có kết quả kinh doanh vì giai đoạn này không thực thi SQL.

Các câu để kiểm tra:
- Doanh thu tháng 10 năm 2026 là bao nhiêu? → SUM có CASE kiểm tra numeric, lọc ngày ký và tháng 2026-10.
- Có bao nhiêu giao dịch đã ký hợp đồng? → COUNT với ngày ký không NULL/rỗng.
- Có những trạng thái pipeline nào? → GROUP BY trạng thái, không tự lọc ngày ký.
- Tổng tiền cọc? → dùng tien_coc_vnd, không thay bằng doanh thu.
- Top nhân viên? → hỏi lại chỉ tiêu xếp hạng.

Module đọc schema thực mỗi lần hết cache. Gemini chỉ nhận schema và câu hỏi, không nhận các hàng khách hàng. JSON được kiểm tra bằng Zod; SQL vẫn là nội dung chưa tin cậy, chưa có validator SQL và tuyệt đối không được gửi vào executor ở giai đoạn 3.

Dữ liệu quan sát do người dùng cung cấp: 1.800 dòng, 1.707 ngày ký NULL, 0 ngày ký rỗng, suy ra 93 dòng có ngày ký. Đây là ảnh chụp thời điểm kiểm tra, không được hardcode thành KPI. Mẫu ngày ISO có giờ, tiền VND dạng chữ số nguyên, diện tích có dấu phẩy thập phân. Mẫu không chứng minh toàn bộ dữ liệu hợp lệ; cảnh báo chất lượng dữ liệu đi kèm SQL đề xuất. Chưa ép ngày text sang date vì chưa kiểm tra toàn bộ ngày hợp lệ.

Nguồn SDK: https://ai.google.dev/gemini-api/docs/generate-content/structured-output

### Kiểm tra bản giai đoạn 3

TypeScript (`tsc --noEmit`) và ESLint đã qua. Build production bị chặn bởi runtime hiện tại: `ENOENT: uv_resident_set_memory`; chưa xác nhận build production thành công. Chưa thử Gemini hoặc database live trong môi trường tạo mã vì không có credentials của người dùng. Cần chạy `npm run build` trên máy bạn và thử trang `/ai-preview`.
