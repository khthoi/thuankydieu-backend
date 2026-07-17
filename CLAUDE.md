# thuankydieu-backend — API & nghiệp vụ thuankydieu.vn

Backend **NestJS 11** (TypeScript, npm) cho website công khai thuankydieu.vn. Đọc `../CLAUDE.md` (gốc dự án) trước — nơi đó có 5 quyết định đã chốt, danh mục tài liệu thiết kế và những điều **chưa chốt**.

## Current State

**Mới scaffold (2026-07-17)** — chỉ có AppModule/AppController mặc định của Nest CLI. **Chưa có:** module nghiệp vụ nào, kết nối CSDL (chưa chọn hệ quản trị lẫn ORM), cầu nối sang lõi AI SAGEBOOK. Mọi thứ dưới đây là **kế hoạch** — đừng giả định đã tồn tại.

## Vai trò & phạm vi

- API duy nhất cho `../thuankydieu-frontend` (cả phần khách lẫn phần quản trị).
- Chủ nghiệp vụ: đơn hàng, thanh toán, quyền đọc, lượt hỏi AI, bộ nhớ câu trả lời đã duyệt, tham số cấu hình.
- **Cầu nối lõi AI SAGEBOOK:** phần tra sách + sinh câu trả lời + bộ nhớ thích nghi đã có sẵn bằng Python ở thư mục ông bà (`D:\SAGEBook`: `query/`, `adaptive/`, `api/main.py` FastAPI). Cách gọi (HTTP nội bộ sang FastAPI, hay tách service riêng) **chưa quyết** — hỏi trước, đừng port sang TypeScript khi chưa được yêu cầu.

## Cơ sở dữ liệu — nguồn sự thật

Thiết kế bảng theo đúng **`../../Documentation/Business Analysis/Mo ta so do ERD chi tiet.docx`** (đọc bằng MCP `word-document-server`): **15 bảng** — `khach_hang`, `nhan_vien`, `sach`, `noi_dung_sach`, `san_pham`, `don_hang`, `chi_tiet_don_hang`, `thanh_toan`, `quyen_doc`, `cau_hoi_ai`, `cau_tra_loi_da_duyet`, `nguon_cau_hoi`, `nguon_cau_duyet`, `bai_viet`, `tham_so_cau_hinh`.

Quy tắc bất di bất dịch từ tài liệu đó:

- Tên bảng/cột snake_case không dấu, khoá chính `ma_...`; tiền `DECIMAL` (không FLOAT); mật khẩu mã hoá một chiều.
- **Không xoá cứng** bảng có tính lịch sử (`don_hang`, `thanh_toan`, `cau_hoi_ai`, `cau_tra_loi_da_duyet`) — dùng cột trạng thái (`DA_HUY`, `THU_HOI`, `TAM_TAT`…).
- `quyen_doc` UNIQUE trên (ma_khach_hang, ma_sach). `cau_tra_loi_da_duyet.ma_cau_hoi` UNIQUE + cho phép NULL.
- Các khoá ngoại cho phép NULL có chủ đích: đơn của khách vãng lai, câu hỏi miễn phí theo máy (`ma_thiet_bi`), sản phẩm gói lượt không gắn sách, quyền đọc cấp tay.
- Nếu buộc phải lệch thiết kế: cập nhật file Word đó + Current State ở `../CLAUDE.md`.

Hệ quản trị CSDL (đề xuất PostgreSQL) và ORM (đề xuất Prisma/TypeORM) **chưa chọn**.

## Module dự kiến (theo cụm nghiệp vụ ERD — chưa build)

- `auth` — đăng ký/đăng nhập khách + nhân viên, phân quyền quản trị.
- `catalog` — sach, noi_dung_sach (kèm cờ đọc thử), san_pham.
- `orders` — don_hang, chi_tiet_don_hang, thanh_toan (COD / chuyển khoản QR đối chiếu tay; vòng đời MOI → DA_XAC_NHAN → DANG_GIAO → HOAN_TAT / DA_HUY; đơn PDF & gói lượt bỏ qua DANG_GIAO), mã vận đơn GHN.
- `library` — quyen_doc (sinh khi đơn PDF thanh toán xong), phục vụ đọc online.
- `ai` — cau_hoi_ai + nguon_cau_hoi (gọi lõi SAGEBOOK), đếm câu miễn phí theo máy, trừ lượt, hạn mức chi phí AI/ngày (`HAN_MUC_AI_NGAY`).
- `ai-review` — cau_tra_loi_da_duyet + nguon_cau_duyet (duyệt vào bộ nhớ).
- `content` — bai_viet.
- `config` — tham_so_cau_hinh (SO_CAU_MIEN_PHI, GIA_LUOT_HOI, HAN_MUC_AI_NGAY, TY_LE_DOC_THU_MAC_DINH).

## Quy tắc nghiệp vụ AI (kế thừa SAGEBOOK — bắt buộc)

- **No-hallucination hai tầng**: tầng "Theo sách" grounding tuyệt đối; tầng suy luận phải gắn nhãn. DỪNG CỨNG với bệnh nặng/cấp cứu/ngoài chủ đề/đòi số liệu lâm sàng.
- **Chốt chặn bộ nhớ**: khi câu hỏi mới trúng `cau_tra_loi_da_duyet`, phải tra sách lại và đòi kết quả **giao với `nguon_cau_duyet`** — không giao thì bỏ qua bộ nhớ, chạy pipeline. Ngưỡng giống-nhau 0.70 (ưu tiên precision; xem "Đã đo" ở `D:\SAGEBook\CLAUDE.md`).
- **KHÔNG lưu câu trả lời có tầng suy luận vào bộ nhớ đã duyệt**; câu hỏi kèm ảnh (nếu sau này có) cũng không đi qua bộ nhớ.
- Chi phí/token mỗi lượt ghi vào `cau_hoi_ai`; cộng theo ngày so với `HAN_MUC_AI_NGAY`, vượt thì tạm ngưng nhận câu hỏi mới.

## Quy ước

- Dev chạy port **3001**: đặt env `PORT=3001` (main.ts của Nest đã đọc `process.env.PORT`), tránh đụng Next ở 3000.
- Cấu hình/bí mật để trong `.env` (gitignore) — không hardcode key, không log key.
- Validate DTO ở biên (class-validator) khi bắt đầu viết API thật.
- Tài liệu/comment hướng người đọc bằng tiếng Việt.

## Lệnh

- `npm run start:dev` — chạy dev có watch.
- `npm run build` · `npm run lint` · `npm run test` (unit) · `npm run test:e2e`.
