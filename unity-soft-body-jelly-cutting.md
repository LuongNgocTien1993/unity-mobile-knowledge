# Game cắt vật thể mềm 3D trên Unity mobile: XPBD, cắt bằng SDF, skin GPU, Burst

Tài liệu chung cho game Unity mobile có vật thể mềm (thạch) cắt được bằng nét vuốt, chia miếng theo khối lượng. Đúc kết từ dự án Jelly Cut (Unity 6000.3, URP, port từ game web; ~420 model, 300 màn). Không chứa thông tin riêng của dự án. Số liệu hiệu năng đo trong Editor, chưa đo trên máy thật.

## 0. Kiến trúc
- **Core tách khỏi Runtime bằng asmdef**: Core chỉ dùng Unity.Mathematics/Collections/Burst, không MonoBehaviour → toàn bộ logic (vật lý, cắt, luật chơi, meta) chạy trong EditMode test và batchmode.
- **Một hệ tọa độ cho logic**; đổi sang Unity chỉ ở biên (vd `unity = (x, y, -z)`). Đảo một trục thì phải đảo cả winding tam giác, normal mặt phẳng và ma trận cofactor.
- **Scene và UI dựng bằng code** (menu "Build Game Scene") → diff được, sinh lại được, không lệ thuộc prefab.
- **Dữ liệu sinh tự động**: model mô tả bằng DSL → transpiler ra C# (SDF) + HLSL (màu) + shader. Có bước kiểm model (SDF không NaN, bbox bao trọn, thể tích 5–70% bbox, Lipschitz ≤ 1.05, classify tổng = 1) và đối chiếu SDF với bản tham chiếu (< 1e-3).

## 1. Vật thể mềm: XPBD trên lưới tứ diện
- Mỗi bước 60 Hz chia ~10 substep: dự đoán vị trí → giải ràng buộc tet → lưỡi cắt → va chạm → sàn/tường → guide; cuối substep tính vận tốc từ chênh vị trí, kẹp tốc độ tối đa, damping.
- Ràng buộc tet **co-rotational**: trích rotation bằng vài vòng lặp trên quaternion lưu từng tet (khởi đầu từ kết quả bước trước nên hội tụ nhanh) + ràng buộc thể tích. Độ cứng = compliance nội suy theo log (4e-4 → 3e-6). Cache hệ số theo (độ cứng, h).
- **Tiết kiệm CPU**: va chạm giữa miếng chạy cách substep; giữ khoảng cách giữa miếng chạy 30 Hz với lực gấp đôi; substep tự giảm 10 → 4 khi chậm; **sleep** khi mọi node gần đứng yên đủ lâu (khỏi ghi lại dữ liệu vẽ) và "calm" hãm dần theo tuổi để lắng nhanh.
- **Kéo miếng bằng ngón tay**: kéo các node bị bắt về đích (trọng số f², giới hạn tốc độ để ngón tay xa không "bắn" miếng), node còn lại theo ~0.6 lần dịch trung bình, tắt ma sát sàn khi đang kéo. Nếu chỉ kéo node bị bắt thì miếng bị giãn dài.
- **Lỗi hay gặp sau khi cắt**: các miếng chồng lên nhau → lấy mẫu "hull" trên da từng miếng và đẩy trượt ra tới khoảng cách tối thiểu; cắt gần ngang làm nửa trên rơi đè nửa dưới → guide trượt miếng trên bàn về một phía.

## 2. Thread vật lý
- Chạy vật lý ở thread nền với đồng hồ 60 Hz riêng; có chế độ đồng bộ `Tick()` cho test (tất định).
- Hai khóa, luôn lấy theo cùng thứ tự (body rồi state) → không deadlock. Main thread chỉ giữ khóa khi cài world hoặc đọc, chờ tối đa vài bước.
- Tác động từ main thread (lắc do xoay camera, đích kéo của ngón tay) gom lại rồi áp ở bước kế, không chặn main thread. Hai bộ trạng thái để nội suy khi vẽ.
- Mỗi lần cài world tăng "generation"; kết quả thuộc world cũ bị bỏ → tránh race giữa cắt xong và đổi lưới.

## 3. Cắt
- **Mặt phẳng cắt** chứa mắt camera và hai tia qua điểm đầu/cuối nét vuốt → cắt đúng theo hướng nhìn. Chỉ cắt miếng mà nét đi "ngoài → ngoài"; nét dừng giữa chừng là "cắn" (cắt một phần).
- **Làm việc trong không gian rest**: đổi mặt phẳng sang rest bằng `c' = c − n·cw + nr·cr`, `nr = Rᵀn` (R trực giao) → chia miếng không tích lũy sai số do rotation.
- **Miếng là mô tả, không phải mesh**: danh sách mặt phẳng (`n·p < c` là trong) + sector (cắt quanh lỗ) + vùng trừ (cắt đoạn nhô ra). Mesh dựng lại từ SDF của model giao các mặt phẳng; chia lưới con lấy giá trị từ lưới cha, không gọi lại SDF.
- **Surface Nets** trên lưới SDF, mỗi đỉnh chiếu lên mặt bằng Newton. Lưới mô phỏng là cage tứ diện thô (Kuhn, 6 tet mỗi cube); mỗi đỉnh mesh gắn 4 node cage + 4 trọng số barycentric → mesh mịn, mô phỏng rẻ.
- **Khối lượng**: thể tích mesh kín bằng định lý phân kỳ (double); khối lượng miếng = khối lượng còn lại × thể tích miếng / tổng thể tích.
- **Cắt chính xác (trợ giúp)**: chặt nhị phân độ dịch song song của nét tới khi một bên nặng đúng mục tiêu; thể tích một phía bằng cắt tam giác theo mặt phẳng (Sutherland–Hodgman) + fan; trừ phần hao do mép cắt bo tròn `RC²(1 − π/4)` mỗi đơn vị chu vi.

## 4. Burst cho SDF (cải thiện lớn nhất)
- Profiling: lấy mẫu lưới SDF chiếm 68% thời gian load màn; chiếu đỉnh trong Surface Nets chiếm 76% mỗi lần cắt.
- Transpiler sinh mỗi model một lớp `[BurstCompile]` có hàm **batch** n điểm; giữ `FunctionPointer` cache; mọi chỗ gọi đổi sang batch; giữ đường managed (có cờ tắt Burst) để so sánh/debug; phần toán dùng chung một nguồn (double).
- Kết quả (Editor, 53 model): load 452 → 56 ms; cắt lần 1 102 → 42 ms, lần 2 124 → 50 ms.
- Kiểm: test parity Burst/managed chênh ≤ 3.6e-7 trên mọi model, topology mesh giống hệt trên các model nặng.
- Mẹo: Newton từng đỉnh độc lập nên dễ batch; dùng "danh sách đỉnh còn hoạt động" thay `break` để giữ đúng chuỗi phép tính như vòng tuần tự.

## 5. Render vật thể mềm
- **Skin trên GPU**: một mesh tĩnh, vertex 56 byte (rest position, rest normal, 4 trọng số, 4 chỉ số node); vertex shader skin theo dữ liệu node. Bounds đặt rất lớn để không bị cull khi biến dạng.
- Dữ liệu node 16 float: vị trí + 3 cột **cofactor** của F trung bình; normal biến đổi bằng `cof(F)·n` (đúng cho biến dạng không đều).
- **Texture2D RGBA32F thay StructuredBuffer**: nhiều GPU Android GLES3 không có SSBO ở vertex stage.
- Shader mỗi model = template + HLSL sinh từ DSL (màu theo pixel: noise, chấm). Giới hạn độ phức tạp (≤ 16 material, ≤ 3 noise, ≤ 2 dots) vì chạy mỗi pixel trên mobile. Màu hấp thụ `sigma = −ln(max(1e-4, tint)) / 0.3`.
- CI ép biên dịch toàn bộ shader cho D3D/Vulkan/GLES3 (hàng trăm shader, hơn chục phút) để bắt lỗi trước khi build.

## 6. Input, camera, kéo thả
- 1 ngón bắt đầu trên miếng = kéo miếng; ngoài miếng = vẽ nét cắt; 2 ngón = xoay/zoom camera và hủy thao tác dở.
- Camera orbit quanh tâm, zoom giới hạn, tự khung theo hộp các miếng, quán tính khi thả (không quán tính nếu ngón dừng > 80 ms). Tốc độ góc của camera truyền cho vật lý để thạch lắc.
- Kéo miếng 3D thật lên đĩa của khách (không thay bằng ảnh thu nhỏ): thả → chọn thẻ bị hộp màn hình của miếng che nhiều nhất, không thì thẻ dưới ngón tay; thả ra ngoài thì miếng rơi lại và chờ vào trong tường; miếng kéo lại gần camera thì thu nhỏ trên màn hình.
- Giới hạn chiều cao vùng thẻ UI phụ thuộc khung camera: đổi một bên phải đổi bên kia.

## 7. Luật màn
- Một lớp session giữ toàn bộ trạng thái logic, không dính render/vật lý → test được.
- Cơ chế: kiên nhẫn của khách (bồn chồn ở 1/2 và 1/4, dừng khi trợ giúp "calm"), đổi đơn giữa hai khách, "mù" (chỉ biết khối lượng khi còn nguyên), bàn xoay, rung, màn nhiều giai đoạn, vòng đơn, hàng chờ, xích chặn cắt (contour lồi lấy mẫu đều, chặn nét cắt qua mắt xích tới khi đủ lần cắt).
- `canFinish`: kiểm còn chia được các miếng hiện có cho các đơn không → phát hiện thua khi hết lượt cắt.
- Chấm: màn chia đều so sai số với dung sai; màn đơn hàng chấm theo phần đã giao cho từng khách. 2 sao khi `maxErr/tol ≤ 0.65`, 3 sao khi `≤ 0.3`.
- Undo giữ 5 bước: miếng, world, node, rotation, đĩa, số cắt + mặt phẳng cắt.

## 8. Lưu trữ và chuỗi
- `JsonUtility` không đọc Dictionary: đổi thành list cặp, hoặc tự viết parser JSON phẳng cho bảng chuỗi.
- Ghi file nguyên tử: ghi file tạm rồi Replace/Move (.NET Standard 2.1 không có `File.Move(overwrite)`).
- i18n: khóa là câu gốc; thiếu bản dịch thì trả khóa và ghi vào danh sách "missing" để rà. Định dạng số và cân nặng (g/oz, lb) qua MỘT bộ định dạng; mọi chữ hiển thị cân nặng phải đi qua nó (chuỗi ghép cứng " g" sẽ sai khi người chơi chọn oz).
- Dịch vụ store/ads/analytics có bản giả để chạy offline và trong test.

## 9. Test và smoke
- EditMode test (~1000): parity Burst, hình học, vật lý, luật chơi, meta, hợp đồng layout UI, model hợp lệ. Chạy qua script batchmode.
- Smoke cơ chế bằng `-executeMethod`: mỗi cơ chế một bài, log `[CI] mech <tên> ...`, `Must(cond, msg)` ném lỗi khi sai, render ảnh ra `Logs/`. Không dùng `-nographics` vì cần render.
- Chế độ đồng bộ của thread vật lý làm smoke tất định.

## 10. Hạn chế còn lại (để biết trước khi làm)
- Hiệu năng Burst chưa đo trên máy thật; build IL2CPP ARM64 ~8–14 phút.
- Hash noise khác bản web nên hoa văn hơi khác; chưa có khúc xạ/back-depth (1 pass); chưa tái dùng đỉnh lưới cha khi cắt.
