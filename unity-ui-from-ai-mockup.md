# Làm UI Unity giống y mockup AI: hợp đồng từ code → mockup → cắt từ mockup → layout.json → chặn tự động

Tài liệu chung cho dự án Unity mobile dựng UI bằng code (uGUI + TextMeshPro) từ mockup do AI vẽ. Đúc kết từ dự án Jelly Cut (2 đợt, 13 màn/popup, ~25 mockup). Không chứa thông tin riêng của dự án.

## Vì sao cách cũ lệch mockup
Cách hay làm: vẽ mockup để duyệt → vẽ thêm "sheet asset" riêng → cắt → code tự đặt vị trí bằng hằng số. Kết quả game khác mockup ở ba chỗ:
1. **Hình dạng**: asset vẽ lại là một ảnh khác (panel có gân thành panel trơn, kéo 9-slice làm mất chi tiết).
2. **Nội dung**: mockup do AI bịa (trộn hai popup, thêm nút code không có, thiếu nhánh điều kiện).
3. **Kích thước/vị trí**: hằng số ước bằng mắt, lệch 10–30%.
Quy trình dưới đây bỏ cả ba chỗ đoán: nội dung lấy từ code, hình lấy từ chính mockup, vị trí đo tự động, và có bước chặn bằng máy.

## 1. Hợp đồng màn (trước khi vẽ mockup)
Đọc code màn + flow, lập một file mỗi màn:
- **Trạng thái cần mockup**: chỉ những tổ hợp điều kiện làm ĐỔI BỐ CỤC (có/không tim, có/không nút ✕, 1/2/3 khách, mô tả 1 dòng/3 dòng). Trạng thái chỉ đổi chữ/sprite thì code tự đổi, không vẽ. Giảm số mockup ~2 lần.
- **Phần tử**: id (= tên field code, = id trong layout), loại, nội dung, điều kiện hiện, nguồn `file:dòng`.
- **Chữ**: các ngôn ngữ, biến, bản DÀI NHẤT (một trạng thái phải vẽ chữ dài nhất).
- **Hành động** của từng nút.
- **KHÔNG có trong code (cấm vẽ)**: liệt kê bằng grep (nút quảng cáo, giá, nhân vật, nút phụ...).
- `visibility.json`: id nào hiện/ẩn ở trạng thái nào, dùng cho bước chặn.
Muốn thêm tính năng thì sửa code + hợp đồng trước, mockup sau. Đọc code để lập hợp đồng là việc tìm kiếm: giao agent rẻ (model nhỏ) chạy song song.

## 2. Mockup theo hợp đồng
- Prompt liệt kê đúng phần tử, chữ thật, và câu: "STRICT CONTENT RULE: draw ONLY the elements listed, with EXACTLY the text given. Do not add any other button, icon, badge, character, counter, price..."
- Trạng thái thứ hai của cùng màn = SỬA mockup thứ nhất ("same popup... changes only: ..."), để panel/cỡ không trôi.
- Truyền mockup đã duyệt làm chuẩn style cho mọi đợt sau.
- Kiểm từng mockup theo bảng hợp đồng (đủ/thừa/thiếu) trước khi gửi duyệt. Một lượt review bởi agent "nghĩ" bắt được lỗi **đồng nhất giữa màn** (thanh tab 3 kiểu, icon xu/booster khác nhau, header khác màu): chọn MỘT mockup chuẩn cho mỗi phần dùng chung.
- Công cụ CLI vẽ ảnh (vd cursor-agent trên Windows qua .cmd) có thể **cắt prompt nhiều dòng**: truyền prompt một dòng "Read X.md and do exactly what it says".

## 3. Cắt sprite từ chính mockup
Không cắt thẳng được vì phần tử chồng nhau (nhân vật che góc panel, chữ trên nút). Cách làm:
1. Nhờ AI **sửa mockup thành bản sạch (plate)** trên nền magenta phẳng: `-panel` (chỉ panel, vẽ trọn mép), `-parts` (banner/nút/icon KHÔNG chữ), `-small` (badge chồng lên icon). Giữ nguyên vị trí.
2. AI KHÔNG giữ vị trí: lệch ~8 px, cỡ lệch vài %, có khi phóng to 2–3× một mảnh lẻ, màu trôi (kem → vàng), sót bóng chữ, vẽ thừa. Nên mọi mảnh phải được **khớp lại vào mockup**:
   - phóng to plate ×4 (Real-ESRGAN anime, miễn phí, chạy GPU) rồi mới tách nền;
   - tách nền theo khoảng cách màu tới nền + khử viền; nền hồng đậm làm mảnh đỏ mất nửa → khóa theo **hue**;
   - **khớp**: dò thang (0.94–1.06 mỗi trục) và dịch (±12 px), chi phí = sai khác RGB trên **dải 5 px trong mép** mảnh (nét viền). Tương quan xám hay gradient KHÔNG phân biệt được lệch vị trí khi mockup có chữ đè; thất bại thì dò thô đa thang (0.35–2.8×) ở 1/4 độ phân giải;
   - hộp tìm được = **số đo vị trí trên mockup** (khỏi đo tay);
   - khớp màu tuyến tính theo mockup CHỈ ở pixel sáng (fit trên vùng kem kéo viền tối thành nâu);
   - xóa bóng chữ: mỗi hàng trong vùng chữ trộn về trung vị của hàng (giữ gradient dọc).
3. Xuất 3 px/css (≥ cỡ hiển thị trên máy 1080p), kèm ảnh kiểm (mockup | mảnh đặt lên | hiệu).
4. Mảnh hỏng thì cắt lại với vùng tìm, hoặc lấy mảnh tương đương từ màn khác; trạng thái khác (toggle tắt) sinh từ mảnh đã cắt (lật + đổi hue) thay vì vẽ lại.
5. **Nhân vật "nấp sau panel"**: đừng ghép ảnh bán thân + bàn tay rời (lộ hết, mép áo cắt ngang, tay lơ lửng). Tách nhân vật nguyên con từ mockup, giữ đúng phần NHÌN THẤY (ngoài panel + chỗ mockup trùng nhân vật = bàn tay), đặt trước panel.

## 4. Đo chữ
- Cỡ chữ = chiều cao chữ hoa ÷ `capHeight` thật của font (đọc OS/2 bằng fontTools), VÀ cỡ theo bề rộng (đo chữ thật bằng font game) — lấy số nhỏ hơn. Font của mockup AI thường hẹp hơn font game: đo theo chữ hoa làm chữ game rộng hơn 13–19%. Chữ nhiều dòng chỉ dùng bề rộng.
- TMP căn giữa dọc đặt tâm khung dòng ở baseline − (ascent − descent)/2·size; tính với font để biết tâm hộp ứng với gì (với Lilita One trùng tâm chữ hoa).
- Hộp chữ con (`tx, ty, tw, th` tương đối hộp cha) cho tiêu đề banner/nút: công thức lề cố định đặt chữ lệch.
- Nhãn cùng loại dùng trung vị; chữ động có `minScale` + autosize; chụp thêm bản ngôn ngữ dài nhất.

## 5. layout.json và code
- `Resources/UiLayout/<màn>.json`: `states[]` (mỗi mockup một state) → `panel` + `items[] {id, x, y, w, h, font, color, align, outline, autosize, minScale, sprite, icon, tx..th, anchor, stretch, parent}` + `repeats[] {id, cell, cols, count, dx, dy, parts[]}` cho phần tử lặp (lưới ô màn, thẻ, cột nút; part có variant `id:on`, sprite có `{i}`).
- Sinh JSON bằng SCRIPT từ hộp đo + chữ đo, không gõ tay.
- Lấy đơn vị css từ CanvasScaler thật (canvas không phải lúc nào cũng 390/375).
- Code: tên GameObject = id, `Bind(id, rt)`; Configure chọn state rồi áp hộp; code chỉ gắn dữ liệu và bật/tắt. Thiếu JSON thì giữ đường cũ.
- Popup: panel theo màn, item tương đối panel; panel nguyên khối (Simple), mỗi state một ảnh; co toàn popup khi màn hẹp (9:19.5 → canvas ~362 css) / thấp (4:3).
- Màn toàn màn hình: khung "Fit" bám safe area, co đều k = min(1, W/400, H/711); item có anchor 9 điểm để màn dài/thấp không vỡ. HUD ở màn tab cần state riêng (sát mép trên) thay vì vị trí của Home.
- Phần tử có animation: đặt hộp nhưng giữ pivot (nút nảy, sao xoay); vị trí gốc cho animation lấy từ layout.
- Không gọi Load/parse JSON trong Update. Sprite trong Resources luôn vào build: panel to nén thấp hơn (ASTC 6x6), cân nhắc SpriteAtlas cho icon nhỏ.
- Khi `Destroy` con lúc dựng lại màn (trễ tới cuối frame), đổi tên/tắt con cũ trước, để `Find` không bắt nhầm.

## 6. Chặn tự động
- Smoke trong Editor batchmode dựng ĐÚNG từng trạng thái và DỮ LIỆU của mockup (đơn vị, số màn, số mạng), chụp 720×1280 và xuất hộp thật của mọi id (css).
- Script so: hiện/ẩn từng id theo hợp đồng; hộp lệch ≤ 3 css; độ giống ảnh từng vùng (bỏ phần tử mảnh < 8 css và phần tử phụ thuộc dữ liệu). Mục lệch có lý do (giới hạn gameplay, sao xoay đo bbox) ghi vào `_why` để không tính là lỗi.
- Chụp thêm 590/860/960 px rộng và kiểm không phần tử nào ra ngoài canvas; phần dùng chung (thanh tab) phải trùng hộp giữa các màn.
- Lặp sửa tới khi chỉ còn mục có lý do. Người duyệt cuối xem ảnh ghép mockup | game của mọi trạng thái, rồi Play mode.

## 7. Tổ chức việc với nhiều agent
- Hợp đồng: agent đọc code (rẻ) song song theo nhóm màn.
- Code: chia track KHÔNG chung file, viết song song KHÔNG compile, một lần compile chung (Unity khóa project). Mỗi track ghi "điều kiện JSON" ra file để agent dữ liệu sinh JSON mà không đọc code.
- Một agent dữ liệu: gán mảnh cắt vào id, đo chữ, sinh JSON, kiểm đủ id/sprite.
- Review code bởi agent riêng trước khi gộp; lỗi hay gặp: phần tử dùng chung nằm ngoài khung co, popup tràn màn hẹp, fallback khi thiếu sprite giữ ảnh state trước, chữ dài tràn hộp đo từ chuỗi ngắn.

## Số liệu tham khảo
- Mỗi ảnh AI ~2–4 phút; khớp một mảnh 10–20 s (dò thô thêm vài giây); smoke 24 state ~2 phút.
- Đợt 2: 11 màn, 20 mockup, 6 track code viết song song compile 0 lỗi lần đầu; gate 24 state, mọi hộp lệch ≤ 3 css trừ mục có lý do.
