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

## Nguyên tắc lập trình

### Kiến trúc & tổ chức code

1. **Monolith NestJS, một module mỗi cụm nghiệp vụ** (danh sách ở "Module dự kiến"). KHÔNG microservice, không CQRS/event-sourcing, không thêm layer trừu tượng khi chưa đau — hệ này một người vận hành, đơn giản là sống còn.
2. **Controller MỎNG**: chỉ nhận request → validate DTO → gọi service → trả kết quả. Toàn bộ nghiệp vụ nằm trong service. Truy cập CSDL qua lớp ORM/repository; service không rải SQL khắp nơi.
3. Module cần module khác → import và gọi qua **service public** của module đó, không với thẳng vào repository của người khác.
4. Không cài thư viện mới khi Nest/stdlib làm được; cài gì ghi lý do vào Current State.

### Cấu trúc thư mục & đặt tên file

```
src/
├── main.ts · app.module.ts
├── common/                     # dùng chung toàn app: guards/ · filters/ · pipes/ · decorators/ · utils/
├── config/                     # nạp .env (ConfigModule), cấu hình hạ tầng
└── <module>/                   # mỗi cụm nghiệp vụ một thư mục: auth/ · catalog/ · orders/ ·
    │                           #   library/ · ai/ · ai-review/ · content/ · config-nghiep-vu/
    ├── orders.module.ts
    ├── orders.controller.ts    # @Controller('don-hang') — route vẫn tiếng Việt
    ├── orders.service.ts
    ├── orders.service.spec.ts  # test đặt CẠNH file được test
    ├── dto/                    # create-order.dto.ts · update-order-status.dto.ts · filter-orders.dto.ts
    └── entities/               # order.entity.ts · order-item.entity.ts (ánh xạ bảng don_hang, chi_tiet_don_hang)
```

**Quy tắc đặt tên:**

a. **File**: kebab-case + hậu tố chuẩn Nest (`.module.ts`, `.controller.ts`, `.service.ts`, `.guard.ts`, `.filter.ts`, `.dto.ts`, `.entity.ts`, `.spec.ts`). Mỗi file một class chính, tên file khớp tên class.
b. **Ranh giới ngôn ngữ (quan trọng — áp toàn dự án):** tiếng Việt không dấu dành cho những gì "nhìn thấy từ bên ngoài" — route API (`don-hang`), tên bảng/cột CSDL, **tên trường JSON/DTO** (`ma_san_pham`, `so_luong` — khớp ERD và `mock.ts`/`kieu.ts` phía frontend); tiếng Anh dành cho định danh nội bộ code — class/interface PascalCase (`OrdersService`, `AdminGuard`), method/biến camelCase. Không trộn hai kiểu trong cùng một loại tên.
c. **Entity**: file/class tiếng Anh (`order.entity.ts` → class `Order`), ánh xạ tên bảng tiếng Việt qua decorator (`@Entity('don_hang')` hoặc `@@map` của Prisma); **property đặt đúng tên cột snake_case của ERD** (`ma_don_hang`, `trang_thai`) — nhìn entity là thấy nguyên bảng, không có lớp "dịch tên" ở giữa.
d. **DTO là code nội bộ → tiếng Anh trọn vẹn**: file `create-order.dto.ts`, class `CreateOrderDto` (khớp quy tắc a và b). Chỉ có **các trường bên trong DTO** là snake_case tiếng Việt (`ma_san_pham`, `so_luong`) vì chúng chính là JSON mà client gửi/nhận.
e. Bảng nối/khái niệm ERD chưa có module riêng (vd `nguon_cau_duyet`) → entity nằm trong module chủ quản gần nhất (`ai-review/entities/answer-source.entity.ts` ánh xạ bảng `nguon_cau_duyet`), không đẻ module mới chỉ vì có bảng.

### API

5. Prefix chung `/api`. Route kebab-case tiếng Việt không dấu, khớp tên frontend đang chờ: `/api/don-hang`, `/api/san-pham`, `/api/quyen-doc`, `/api/cau-hoi-ai`, `/api/bai-viet`, `/api/cau-hinh`…
6. REST chuẩn: GET danh sách (phân trang `?trang=&moi_trang=`, lọc qua query), GET `/:id`, POST, PATCH. **Hầu như không có DELETE thật** — dữ liệu lịch sử đổi trạng thái bằng PATCH (đúng nguyên tắc ERD không xoá cứng).
7. Mã HTTP đúng nghĩa: 400 validate, 401 chưa đăng nhập, 403 không đủ quyền, 404 không có, **409 chuyển trạng thái không hợp lệ**. Body lỗi thống nhất `{ thong_bao: "câu tiếng Việt tử tế cho người dùng" }` — không bao giờ lộ stack trace/SQL/chi tiết kỹ thuật ra ngoài.

### Validate & không tin client

8. Global `ValidationPipe` với `whitelist: true, forbidNonWhitelisted: true` — mọi input đi qua DTO có class-validator, field lạ bị chặn từ biên.
9. **KHÔNG TIN BẤT KỲ CON SỐ NÀO CLIENT GỬI LÊN.** Client chỉ gửi `ma_san_pham` + `so_luong`; đơn giá, thành tiền, tổng tiền **luôn tính lại ở server** từ bảng `san_pham` (và chép vào `chi_tiet_don_hang.don_gia` tại thời điểm đặt). Số lượt hỏi cộng/trừ chỉ do server quyết. Quyền đọc chỉ sinh từ thanh toán đã xác nhận.
10. Phân quyền bằng Guard: route `/api/quan-tri/...` chỉ cho `nhan_vien`; route tài khoản kiểm tra **sở hữu** (khách chỉ xem được đơn/quyền đọc/lượt hỏi của chính mình).
11. Mật khẩu hash một chiều (bcrypt hoặc argon2). Không bao giờ để `mat_khau_ma_hoa` lọt ra response (serialization loại trừ mặc định).

### Toàn vẹn nghiệp vụ (tiền bạc — quan trọng nhất)

12. **Trạng thái đơn là máy trạng thái**, chỉ cho phép bước chuyển hợp lệ: `MOI → DA_XAC_NHAN → DANG_GIAO → HOAN_TAT`; huỷ chỉ khi chưa giao; đơn PDF/gói lượt bỏ qua `DANG_GIAO`. Chuyển sai → 409, không ghi gì.
13. **Thao tác chạm nhiều bảng bắt buộc nằm trong transaction**: xác nhận tiền về = cập nhật `thanh_toan` + `don_hang` + sinh `quyen_doc` / cộng `so_luot_hoi_con_lai` — tất cả hoặc không gì cả.
14. **Idempotent**: bấm "xác nhận tiền về" hai lần không được sinh quyền đọc/cộng lượt hai lần — kiểm tra trạng thái hiện tại trước khi ghi + dựa vào ràng buộc UNIQUE của ERD.
15. Mọi hành động quản trị ghi vết `ma_nhan_vien` (ai xác nhận đơn, ai duyệt câu trả lời) — ERD đã chừa sẵn cột.
16. **Tham số nghiệp vụ đọc từ bảng `tham_so_cau_hinh`** (SO_CAU_MIEN_PHI, GIA_LUOT_HOI, HAN_MUC_AI_NGAY…), KHÔNG nhét vào `.env`. `.env` chỉ chứa bí mật hạ tầng: chuỗi kết nối CSDL, key dịch vụ AI, secret phiên đăng nhập.

### AI & chi phí

17. Kiểm tra **trước khi** gọi AI: còn lượt (miễn phí theo `ma_thiet_bi` hoặc lượt trong tài khoản) và tổng chi phí hôm nay < `HAN_MUC_AI_NGAY`. Vượt → trả thông báo tử tế, không gọi.
18. Mỗi lượt hỏi ghi `cau_hoi_ai` (token, chi phí, nguồn trả lời) ngay trong luồng xử lý — chi phí ngày tính gộp từ bảng này, không sổ sách riêng.
19. Rate-limit endpoint hỏi AI theo IP/thiết bị — chống spam đốt tiền AI.

### Kiểm thử & chất lượng

20. Nghiệp vụ đụng tiền/trạng thái **bắt buộc có unit test service**: máy trạng thái đơn, tính tiền server-side, idempotency xác nhận thanh toán. Luồng chính (đặt hàng → xác nhận → mở khoá) có e2e.
21. `npm run lint` + `npm run test` xanh trước khi commit. Commit message tiếng Việt.

### Môi trường & chung

22. Dev chạy port **3001**: đặt env `PORT=3001` (main.ts của Nest đã đọc `process.env.PORT`), tránh đụng Next ở 3000. Bật CORS chỉ cho origin của frontend.
23. `.env` gitignore, có `.env.example` liệt kê tên biến (không giá trị thật); không hardcode key, không log key/mật khẩu/nội dung nhạy cảm.
24. Tài liệu/comment hướng người đọc bằng tiếng Việt; tên định danh trong code theo đúng "ranh giới ngôn ngữ" ở quy tắc (b) mục Cấu trúc thư mục & đặt tên file.

## Lệnh

- `npm run start:dev` — chạy dev có watch.
- `npm run build` · `npm run lint` · `npm run test` (unit) · `npm run test:e2e`.
