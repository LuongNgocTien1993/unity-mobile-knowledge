# Kinh nghiệm Unity mobile: fix bug, lỗi build, lỗi lên store

Tài liệu chung cho mọi dự án Unity (Android và iOS). Không chứa thông tin riêng của dự án nào. Đúc kết từ dự án IdleGuildMaster, nên một số ví dụ có nguồn gốc từ dự án đó.

Khi áp dụng vào một dự án, đọc thêm file riêng của dự án đó (nếu có) để biết cấu hình cụ thể: profile build, nơi lưu secret, môi trường, v.v.

---

## 0. Strip code: lỗi nguy hiểm nhất, đọc trước

Strip (managed stripping / IL2CPP linker) xóa code mà Unity thấy không được gọi trực tiếp. Lỗi do strip **chỉ xuất hiện trong bản Release/IL2CPP**, Editor và Development build vẫn chạy bình thường. Nên rất dễ lọt qua test trong Editor và chỉ lộ khi phát hành.

### Triệu chứng
- Tính năng hoạt động trong Editor, hỏng trên bản release. Ví dụ: hộp thoại ATT không hiện, mua hàng không có sản phẩm, quảng cáo không load.
- Log kiểu `No adapter for ...`, `Type not found`, `Method not found`, `NullReferenceException` ở chỗ gọi qua reflection.
- Dữ liệu deserialize bị trống hoặc thiếu field, dù JSON vẫn đúng.
- Lỗi chỉ xuất hiện sau khi đổi Stripping Level, bật/tắt một package, hoặc thêm một SDK.

### Code thường bị strip nhầm
- Class/assembly chỉ được đăng ký qua `RuntimeInitializeOnLoadMethod`, không có ai tham chiếu trực tiếp (ví dụ adapter quảng cáo, plugin).
- Code gọi từ native: callback JNI (Android), callback Objective-C/Swift (iOS), delegate của SDK.
- Gọi qua reflection: `Type.GetType("...")`, `Activator.CreateInstance`, tên type/method dạng chuỗi.
- Type chỉ xuất hiện trong prefab, ScriptableObject hoặc Addressables, không có reference trong code.
- Thư viện chỉ được gọi bằng tên, ví dụ khóa license key (`GooglePlayTangle`).

### Cách giữ lại code
1. **`link.xml`** (ưu tiên, dễ kiểm soát): đặt trong `Assets/` hoặc trong thư mục của package/assembly. Ví dụ:
   ```xml
   <linker>
     <assembly fullname="MyAdapterAssembly" preserve="all" />
     <assembly fullname="Assembly-CSharp">
       <type fullname="Namespace.SomeClass" preserve="all" />
     </assembly>
   </linker>
   ```
   Dùng `preserve="all"` để giữ toàn bộ assembly/type. Có thể giới hạn xuống `preserve="fields"` hoặc `methods` nếu cần giảm kích thước.
2. **`[Preserve]`** (`UnityEngine.Scripting`) trên class, method hoặc constructor được gọi từ native hoặc reflection.
3. **`[assembly: AlwaysLinkAssembly]`** khi cả assembly không có reference nào trỏ tới, nhưng vẫn cần nạp lúc chạy (đăng ký qua `RuntimeInitializeOnLoadMethod`).
4. **Tham chiếu thật từ code**: gọi một hàm của assembly đó từ chỗ khởi động. Đây là cách không phụ thuộc cấu hình, nhưng phải là code chạy thật, không phải biến không dùng.
5. **Hạ Managed Stripping Level** (Low hoặc Minimal) để thử nghiệm và xác định nguyên nhân. Không nên giữ mức cao nếu chưa kiểm tra kỹ.

### Quy tắc khi thêm SDK, plugin hoặc tính năng reflection
- Mỗi khi thêm SDK/package/adapter có callback native, reflection hoặc đăng ký tự động: thêm `link.xml` hoặc `[Preserve]` ngay khi viết, không đợi lỗi.
- Đừng xóa `link.xml` khi dọn project. Kiểm tra `git status` để chắc file không bị mất.
- Sau mỗi thay đổi liên quan đến strip, build **Release IL2CPP** và chạy thử trên máy thật. Không chỉ test Development build.
- Ghi chú trong code vì sao phải giữ (ví dụ "nothing references this assembly, IL2CPP strips it"), để người sau không xóa nhầm.

### Ví dụ đã gặp (IdleGuildMaster)
- Hộp thoại ATT không hiện trên iOS vì assembly quảng cáo bị strip. Sửa bằng `AlwaysLinkAssembly` và `[Preserve]`.
- Adapter quảng cáo AdMob/UnityAds/AppLovin bị strip, waterfall báo `No adapter`. Sửa bằng `link.xml` với `preserve="all"`.
- Khóa `GooglePlayTangle` bị strip, receipt validation hỏng. Sửa bằng `link.xml` trong `Assets/_Game/`.

Chi tiết commit và file: xem file riêng của từng dự án.

---

## 1. Quy tắc fix bug

1. **Tìm nguyên nhân gốc, không vá triệu chứng.** Ví dụ: code tạo UI chỉ tồn tại trong `#if UNITY_EDITOR` làm player build lỗi. Cách sửa đúng là đưa phần runtime ra ngoài `#if`, không thêm `if` ở từng chỗ gọi.
2. **Grep mọi caller của hàm vừa sửa.** Sửa một chỗ trong hàm dùng chung thay vì sửa từng nơi.
3. **Log thật từ thiết bị trước khi kết luận.** Dùng `adb logcat` (Android) hoặc Xcode console (iOS) để tìm nguyên nhân, đừng đoán từ Editor.
4. **Sau mỗi lần merge nhánh, kiểm tra lại edit có thật sự được áp dụng không.** Nhánh cũ bị cắt có thể làm mất edit mà không báo conflict, và code tham chiếu tới hằng số chưa được định nghĩa.
5. **Không xóa `.meta`, không xóa prefab.** Khôi phục lại rất tốn công. Dời tài sản bằng Unity hoặc `git mv` kèm `.meta`.
6. **Sau khi sửa C#, refresh Unity và đọc console trước khi báo xong.**

---

## 2. Lỗi hay gặp khi build

### 2.1 Code chỉ chạy trong Editor
- **Triệu chứng:** Editor chạy đúng, player build NRE hoặc thiếu UI/icon.
- **Nguyên nhân:** logic runtime nằm trong `#if UNITY_EDITOR` (thường là phần bake UI).
- **Cách fix:** đưa phần cần lúc runtime ra ngoài `#if`. Chỉ giữ bake/menu trong Editor.
- **Lưu ý:** sau khi đổi code bake, phải chạy build player thật, không chỉ test trong Editor.

### 2.2 Thiếu assembly reference trong asmdef
- **Triệu chứng:** Editor compile được, player build báo `type not found`.
- **Cách fix:** thêm vào `references` của asmdef các assembly của nền tảng (ví dụ `Unity.Notifications.Android`, `Unity.Notifications.iOS`).

### 2.3 Strip code
- Xem **mục 0** ở đầu file. Đây là nhóm lỗi nguy hiểm nhất, và cách xử lý (link.xml, `[Preserve]`, `AlwaysLinkAssembly`) đều nằm ở đó.

### 2.4 Hook editor theo nền tảng chạy sai target
- **Triệu chứng:** build Android chạy hook của iOS (hoặc ngược lại), ví dụ sai manifest của AdMob.
- **Nguyên nhân:** hook `#if UNITY_ANDROID` / `#if UNITY_IOS` chỉ có đúng sau domain reload cho target đang chọn.
- **Cách fix:** trong script build, kiểm tra target hiện tại của assembly editor so với platform sắp build. Nếu lệch, gọi `CompilationPipeline.RequestScriptCompilation()` và dừng với thông báo bảo người dùng bấm lại.

### 2.5 `google-services.json` không được copy khi dùng custom Gradle template
- **Triệu chứng:** Firebase/Crashlytics không hoạt động trên bản build, không có lỗi rõ ràng.
- **Nguyên nhân:** khi có `launcherTemplate.gradle` tự định nghĩa, Unity không tự copy file này.
- **Cách fix:** copy thủ công trong Gradle. Đường dẫn tương đối tới `Assets/` phụ thuộc cấu trúc output của Unity, nên ghi chú lại và kiểm tra khi đổi phiên bản Unity.

### 2.6 Crashlytics và R8 mapping
- **Triệu chứng:** stack trace crash bị obfuscate, không đọc được.
- **Cách fix:** bật R8 cho release với file proguard riêng. Thêm plugin `com.google.firebase.crashlytics` vào Gradle để upload mapping. Giữ tên `SourceFile` để stack trace dễ đọc hơn.
- **Lưu ý:** file `baseProjectTemplate.gradle` và `launcherTemplate.gradle` do dự án tự quản lý. Khi nâng AGP hoặc Gradle, kiểm tra tương thích theo bảng của Google.

### 2.7 BOM làm hỏng file JSON
- **Triệu chứng:** Unity Package Manager không đọc được `Packages/manifest.json`.
- **Nguyên nhân:** file được lưu với UTF-8 BOM.
- **Cách fix:** lưu lại UTF-8 không BOM. Khi ghi JSON bằng PowerShell 5.1, dùng `-Encoding utf8` hoặc ghi bằng công cụ không thêm BOM.

### 2.8 Addressables
- **Catalog hoặc namespace lỗi trong Editor:** kiểm tra lại namespace và đường dẫn catalog.
- **Build sai môi trường:** luôn kiểm tra profile (dev/staging/production) trước khi build remote content.
- **Art load bằng Addressables nhưng chưa tải:** nếu không tải art cần thiết trước khi vào gameplay, UI hiện ô trống. Tải art khi restore hoặc khi vào màn hình tương ứng.
- **Catalog init lỗi:** thêm retry và refresh catalog khi khởi động.
- **Rollback:** lưu `addressables_content_state.bin` theo từng phiên bản để có thể `Content Update` về bản cũ. Không xóa file này.
- **Đổi nguồn remote (ví dụ từ CCD sang CDN khác):** bản đã phát hành vẫn giữ URL catalog cũ. Phải phát hành một bản build mới dùng nguồn mới.
- **Bỏ kiểm tra CRC bundle** giúp load nhanh hơn, nhưng chỉ nên làm khi đã chắc bundle không hỏng.

### 2.9 Kích thước build
- Dùng **ASTC 6x6** cho texture runtime trên cả Android và iOS nếu chất lượng chấp nhận được.
- Không để bản sao Addressables trùng trong AAB. Đưa nhạc và asset lớn sang remote.
- Không đưa scene demo của asset store vào build.
- Dùng Build Report Tool để phân tích kích thước trước khi phát hành.

### 2.10 Version code
- Tự tăng `bundleVersionCode` khi build AAB. Áp dụng cho App Bundle, không áp dụng cho APK.

### 2.11 Keystore và mật khẩu
- Unity không lưu mật khẩu keystore giữa các lần mở Editor. Có thể lưu tạm trong EditorPrefs (chỉ làm nhẹ, không phải mã hóa), và nhập lại nếu build báo sai mật khẩu.
- **Không commit keystore vào git.** Đây là khóa ký release. Nếu bị lộ, người khác có thể ký app giả.

---

## 3. Lỗi hay gặp khi chạy trên thiết bị

| Triệu chứng | Nguyên nhân thường gặp | Cách fix |
|---|---|---|
| Màn hình intro "đứng", không bấm được | Debug OnGUI che và nuốt touch | Chỉ bật debug UI trong Editor hoặc Development build |
| Crash khi khởi động, log native | Gọi SDK (Crashlytics, analytics) trước khi SDK sẵn sàng | Trì hoãn đến sau khi SDK khởi tạo xong |
| Đóng băng cinematic | Callback analytics/persist chặn luồng chính | Callback không được chặn luồng gameplay hoặc intro |
| Đăng nhập Google Play Games lỗi | Play Services không có trên máy | Kiểm tra `GoogleApiAvailability` trước khi đăng nhập |
| Mua hàng crash | Play Services không khởi động được flow mua | Kiểm tra Play Services trước, báo lỗi rõ cho người dùng |
| Đăng nhập lỗi trên device | Dùng CustomID thay vì device ID | Dùng login theo device ID của nền tảng |
| Thông báo không có icon | Chưa set icon cho notification | Set small/large icon cho Android local notification |
| Hộp thoại quyền thông báo không hiện | Hỏi quyền khi app đang nền | Chỉ hỏi khi app ở foreground, vào thời điểm hợp lý trong flow |
| Màn hình tự tắt giữa chừng | Màn hình tự khóa | Bật `Screen.sleepTimeout = SleepTimeout.NeverSleep` khi đang chơi |
| Ký tự hiện thành ô vuông | Font không có glyph (ví dụ `−` Unicode) | Dùng ký tự ASCII hoặc thêm glyph vào font |
| Kéo để scroll lại kích hoạt tap | Không chặn tap khi đang drag | Chặn tap khi đã vượt ngưỡng drag |
| Save cũ không load được | Migration bị bỏ hoặc sai | Backup save cũ, chỉ load schema hiện tại, ghi rõ trạng thái cho người dùng |
| Mất progress, cloud conflict sai | So sánh timestamp sau khi đã cập nhật | So sánh timestamp trước khi áp dụng thay đổi |

---

## 4. Lỗi import art và sprite

- **Sprite không hiện:** kiểm tra `Texture Type` là `Sprite (2D and UI)`. Hay gặp khi thêm PNG mới hoặc regen art hàng loạt.
- **Thiếu `.meta`** cho PNG mới: mở Unity để tạo meta rồi commit cả PNG và meta.
- **Art AI trộn nhân vật hoặc lỗi:** kiểm tra mắt bằng QA trước khi commit. Giữ art cũ cho đến khi art mới được duyệt.
- **Nền đen trên icon:** xóa nền trước khi import.
- **Crop sai khi resize:** khôi phục kích thước trước khi crop, không crop chồng lên bản đã crop.

---

## 5. IAP và thanh toán

- **Store trả về một phần sản phẩm:** Unity có thể gọi `OnProductsFetched` và báo lỗi nhưng vẫn có sản phẩm mua được. Không đặt trạng thái "chưa sẵn sàng" chỉ vì thiếu một vài SKU.
- **Receipt validation (Android):** kiểm tra chữ ký receipt bằng `CrossPlatformValidator` và license key. Cần `link.xml` giữ `GooglePlayTangle` để key không bị strip.
- **Log "license key missing" trong Editor là bình thường:** security của IAP là stub trong Editor. Chỉ coi là lỗi trên device.
- **Refund phía client:** lưu ledger mỗi đơn hàng (idempotent). So sánh danh sách sở hữu của store với ledger để thu hồi quyền lợi. Consumable thường cần server để kiểm tra chính xác.
- **Apple receipt:** cần kiểm tra hoặc có server xác thực, không nên tin hoàn toàn vào client.
- **Thông báo server sau khi grant IAP:** làm ở server (CloudScript, backend), không làm trong Editor.
- **Reviewer của App Store cần screenshot cho từng IAP.** Thiếu screenshot làm sản phẩm ở trạng thái `MISSING_METADATA`.

---

## 6. Lỗi hay gặp khi lên store

### 6.1 Google Play
- **Listing nhiều ngôn ngữ:** đẩy bằng Play Developer API qua service account. Giữ key ngoài repo.
- **Privacy policy phải có email liên hệ và mục xóa tài khoản/dữ liệu** nếu game có tài khoản. Play yêu cầu mục này.
- **Feature graphic và logo Play Games on PC** nên có sẵn khi cập nhật listing.
- **Screenshot kích thước Android:** chụp từ Game View đúng tỷ lệ yêu cầu.
- **In-app review:** không ép người chơi đánh giá. Chỉ hỏi vào thời điểm tự nhiên (ví dụ sau khi thắng), có giới hạn số lần và khoảng cách giữa các lần hỏi. Kiểm tra lại chính sách Play hiện hành.
- **Ads phải ở chế độ Live khi phát hành:** dùng test ID khi test và chuyển sang live ID cho bản production. Kiểm tra lại trước khi build release.
- **Đánh giá người chơi:** đọc review 1 sao để tìm rủi ro về save, mất progress sau update, và monetization gate content. Đây là rủi ro lớn về danh tiếng, không chỉ là lỗi kỹ thuật.

### 6.2 App Store
- **App Privacy:** làm trên App Store Connect (web), không có API đầy đủ. Khai báo đúng việc có tracking hay không.
- **Age rating:** khai báo trước khi submit.
- **ATT:** nếu prompt không hiện, kiểm tra IL2CPP strip (mục 2.3).
- **Screenshot cho IAP:** bắt buộc cho mọi sản phẩm, kể cả subscription.
- **Subscription intro offer:** cấu hình trong App Store Connect nếu có.
- **Đẩy metadata bằng App Store Connect API:** dùng key ngoài repo, script idempotent (kiểm tra trạng thái trước khi ghi).

### 6.3 Cả hai store
- **Không commit secret:** API key, credentials, keystore, key JSON. Dùng biến môi trường hoặc file ngoài repo, chỉ commit phần loader.
- **Cheat panel và debug UI chỉ bật trong dev build.** Kiểm tra lại trước mỗi release.
- **Ẩn các nút cheat khi chụp screenshot store** bằng một tùy chọn có sẵn trong build tool.
- **Không đưa scene demo của asset store vào build** nếu không dùng.

---

## 7. Checklist trước khi build release

1. Build Editor OK, console không có lỗi compile.
2. Đúng môi trường (production), đúng profile Addressables, đúng Team ID (iOS).
3. Ads ở chế độ Live.
4. Cheat panel, debug UI, test SDK không bật trong release.
5. Build AAB (tự tăng bundleVersionCode), kiểm tra manifest, `google-services.json`, R8 mapping upload được. Đã kiểm tra `link.xml` còn đủ, và các SDK/adapter có callback native hoặc reflection vẫn có `[Preserve]` hoặc `link.xml` (mục 0).
6. Chạy bản release trên máy thật: intro, tutorial, mua IAP (sandbox), đăng nhập, thông báo, ATT.
7. Sau khi upload Addressables, kiểm tra URL catalog trỏ đúng nguồn mới.
8. Kiểm tra `git status` trước khi commit: không có keystore, secret, hoặc file build output.
9. Lưu lại `addressables_content_state.bin` của bản vừa build để rollback được.

---

## 8. Quy tắc làm việc

- Không in ra giá trị secret, không chép key vào chat hoặc file trong repo.
- Thao tác production (deploy script, upload store) cần người có quyền xác nhận. AI viết script, người dùng chạy, AI chỉ xác minh read-only.
- Text trong UI mobile không in đậm, cỡ tối thiểu theo chuẩn của dự án, nút đủ cao để bấm trên điện thoại.
- Sau khi sửa C#, refresh Unity qua CLI hoặc Editor và đọc console trước khi báo xong.
