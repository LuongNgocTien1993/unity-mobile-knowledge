# Crash và ANR trên Unity mobile: lỗi đã gặp, vì sao, cách fix, cách tránh

Tài liệu chung cho mọi dự án Unity mobile (Android là chính, iOS áp dụng phần lớn). Đúc kết từ các bản 1.0.2 → 1.0.7 của một game idle RPG (IdleGuildMaster, Unity 6.3, IL2CPP, Firebase Crashlytics, Unity IAP 5, AdMob waterfall, PlayFab, Google Play Games). Lỗi build và store nằm ở [unity-mobile-build-store.md](unity-mobile-build-store.md); file này chỉ nói về crash, ANR và các kiểu "đơ game" mà người chơi gặp trên máy thật.

---

## 0. Đọc Crashlytics trước khi sửa

Không phải lỗi nào trong Crashlytics cũng là lỗi của game. Phân loại trước, rồi mới sửa:

| Loại | Dấu hiệu | Làm gì |
|---|---|---|
| Lỗi thật của game | Stack có class của game hoặc package của mình, lặp lại trên nhiều người dùng, nhiều dòng máy | Sửa theo mục 2 |
| Robot kiểm tra trước khi phát hành (Play pre-launch report) | Mỗi lỗi 1 lần, xuất hiện ngay sau khi upload, máy Google lab, lỗi trong activity của SDK (`ProxyBillingActivity`, `GamesResolutionActivity`, `HsdpShimActivity`, `libgpg.so not found`) | Vẫn nên chặn bằng guard (mục 2.5), vì người dùng thật có máy không có Play Services cũng sẽ gặp |
| Lỗi của hệ điều hành | `CannotDeliverBroadcastException`, `DeadSystemException`, crash trong `system_server` | Không sửa được trong game, bỏ qua |
| APK bị mod | ANR/crash trong thư viện không phải của Unity (ví dụ `libgame.so`), tập trung ở một bản, kèm doanh thu IAP ảo | Không sửa; xem skill/tài liệu về purchase giả |
| Hết bộ nhớ lúc nạp tài nguyên | Crash native trong `libunity.so` (`DebugStringToFile` SIGTRAP, SIGBUS), RAM trống của máy rất thấp (< 250 MB), xảy ra trong vài giây đầu | Mục 2.12 |

Số liệu để biết bao nhiêu là "ổn": khi người dùng còn ít (vài chục DAU), mỗi người crash làm tỉ lệ crash-free user tụt vài điểm. Xem tỉ lệ **phiên** không crash (crash-free sessions) và số người dùng bị ảnh hưởng của từng lỗi, đừng chỉ nhìn tỉ lệ user.

Để đọc được stack:
- Upload native symbols của IL2CPP (`libil2cpp.sym.so`) cho Crashlytics, nếu không frame native chỉ là địa chỉ.
- Upload `mapping.txt` của R8 (Crashlytics Gradle plugin + Play Console), nếu không frame Java bị đổi tên.
- Bỏ `-renamesourcefileattribute` trong proguard để stack giữ tên file thật thay vì `SourceFile`.
- Gắn custom key ngữ cảnh (màn hình đang mở, bước tutorial, đang hiện ad hay không) để biết người chơi đang làm gì lúc crash.

---

## 1. Bảng tổng hợp

| # | Triệu chứng | Loại | Nguyên nhân gốc |
|---|---|---|---|
| 2.1 | ANR `NativeCode::~NativeCode` khi app bị đóng từ nền | ANR | Lưu game lần 2 (ghi đĩa đồng bộ, JNI, upload cloud) trong `OnApplicationQuit`/`OnDestroy` sau khi đã lưu lúc pause |
| 2.2 | NRE sau khi mua hàng, có khi SIGSEGV trong `libil2cpp` | Crash | Store IAP cũ vẫn đăng ký event của singleton `StoreController` sau khi reload scene |
| 2.3 | `NullReferenceException` trong `Popup.Hide()` | Crash | Dùng `?.` trên `UnityEngine.Object` đã bị destroy |
| 2.4 | `KeyNotFoundException` trong `AdMobAdapter.Show` | Crash | Cờ "banner đã load" không bị xoá khi reload thất bại |
| 2.5 | NPE trong `ProxyBillingActivity` / `GamesResolutionActivity` | Crash | Gọi mua hàng hoặc đăng nhập Play Games trên máy không có Play Services dùng được |
| 2.6 | Crash ngay khi mở bản release | Crash | R8 full mode xoá constructor `WorkDatabase_Impl` (Room) |
| 2.7 | Crashlytics không ghi gì cả phiên, log lỗi `PlatformAccessor` cctor | Mất báo cáo | Gọi API Crashlytics trước khi Firebase kiểm tra dependency xong |
| 2.8 | Game đứng hình sau khi xem quảng cáo | Đơ | Game bị pause 2 lần, và SDK không báo đóng ad nên không bao giờ resume |
| 2.9 | Intro đứng ở trang đầu, chạm không ăn | Đơ | Callback analytics ném exception sau khi đã nhận tap; panel debug `OnGUI` nuốt touch |
| 2.10 | Lỗi coroutine trên object inactive | Exception | Bắt đầu coroutine trên popup đã đóng |
| 2.11 | Offline catch-up kẹt, combat không chạy | Đơ | Exception trong routine lồng nhau làm cờ "đang catch-up" không được trả về |
| 2.12 | SIGTRAP `DebugStringToFile` trong `libunity.so` lúc mở game | Crash native | Máy ít RAM, hết bộ nhớ lúc nạp texture lên GPU |

---

## 2. Chi tiết từng lỗi

### 2.1 ANR khi Android đóng app đang ở nền
- **Triệu chứng:** Crashlytics báo ANR, stack chính ở `NativeCode::~NativeCode` / `GameActivity.onDestroy`. Hay gặp trên máy yếu.
- **Vì sao:** khi app ở nền bị hệ thống đóng, `GameActivity.onDestroy` chờ Unity thoát xong. Game đã lưu đầy đủ lúc `OnApplicationPause(true)`, nhưng `OnApplicationQuit` và `OnDestroy` lại lưu thêm một lần nữa: chụp trạng thái, ghi file đồng bộ, gọi JNI đặt lịch notification, upload cloud. Tổng thời gian vượt giới hạn ANR (5 giây).
- **Cách fix:** đặt cờ `_savedOnPause = true` khi lưu lúc pause, xoá cờ khi resume. Trong quit/destroy, nếu cờ đang bật thì chỉ flush hàng ghi đang dở (rẻ khi hàng rỗng), không lưu lại từ đầu.
  ```csharp
  void OnApplicationQuit()
  {
      if (_savedOnPause) { Save.FlushPending(); return; } // đã lưu lúc pause
      PersistImmediate();
  }
  ```
- **Cách tránh ở game khác:**
  - Coi `OnApplicationPause(true)` là điểm lưu chính trên mobile. `OnApplicationQuit` không đáng tin trên Android (thường không được gọi) và nếu được gọi thì đang trong đồng hồ ANR.
  - Không làm việc nặng hoặc gọi mạng trong `OnApplicationQuit`, `OnDestroy`, `OnDisable` của object sống cả phiên.
  - Ghi file ở thread nền; trên luồng chính chỉ chụp dữ liệu.

### 2.2 Crash sau khi mua hàng do store IAP cũ còn sống
- **Triệu chứng:** NRE trong popup của màn đã bị huỷ ngay sau khi mua, một số máy có SIGSEGV trong `libil2cpp.so`. Chỉ xảy ra sau khi người chơi đổi ngôn ngữ (game reload scene).
- **Vì sao:** `StoreController` của Unity IAP 5 là singleton sống cả process. Đối tượng bọc store của session cũ vẫn đăng ký `OnPurchasePending`, `OnPurchaseConfirmed`... sau khi scene reload. Khi có đơn hàng, cả session cũ lẫn mới đều nhận: session cũ trao vật phẩm vào dữ liệu đã chết, đụng UI đã destroy, và **confirm đơn lần thứ hai**.
- **Cách fix:**
  - Lớp bọc store implement `IDisposable`, huỷ đăng ký mọi event và xoá callback trong `Dispose()`. Chủ sở hữu (`GameSession`) gọi `Dispose()` trong `OnDestroy`.
  - Unity IAP chỉ bắn `OnPurchasePending` **một lần cho mỗi giao dịch trong một process**. Đơn đã gửi tới store bị dispose sẽ không đến lại. Vì vậy store mới phải đọc `PendingOrders` trong kết quả `FetchPurchases` và tự xử lý các đơn đó, nếu không đơn phải đợi lần mở game sau.
- **Cách tránh ở game khác:**
  - Mọi đăng ký vào singleton, static event hoặc SDK sống lâu hơn scene đều phải có huỷ đăng ký tương ứng. Kiểm bằng cách reload scene giữa lúc đang mua.
  - Nếu game có reload scene (đổi ngôn ngữ, đăng xuất, xoá tài khoản), thử toàn bộ luồng IAP, ads, cloud save sau khi reload.
  - Grant và confirm phải idempotent theo transaction id, để lỡ gọi 2 lần cũng không cộng 2 lần.

### 2.3 `?.` trên `UnityEngine.Object` đã destroy
- **Triệu chứng:** `NullReferenceException` trong `ArenaLobbyPopupView.Hide`, gọi từ `Refresh` của màn chính sau khi màn đó đã bị huỷ.
- **Vì sao:** `popup?.Hide()` chỉ kiểm tra null của C#, bỏ qua kiểm tra "đã destroy" mà Unity cài trong `operator ==`. Object đã destroy vẫn khác null với `?.`, nên lời gọi đi tiếp và chết trong thân hàm.
- **Cách fix:** `if (popup != null) popup.Hide();`
- **Cách tránh ở game khác:** không dùng `?.`, `??`, `is null` với mọi kiểu kế thừa `UnityEngine.Object` (MonoBehaviour, GameObject, ScriptableObject, Component). Bật analyzer `UNT0008`/`UNT0007` của Microsoft.Unity.Analyzers để bắt tự động.

### 2.4 Banner AdMob `KeyNotFoundException` sau khi reload thất bại
- **Triệu chứng:** `KeyNotFoundException` trong `AdMobAdapter.Show` khi hiện banner.
- **Vì sao:** adapter lưu `_bannerLoaded` khi load thành công, nhưng khi lần refresh sau thất bại thì không xoá cờ đó. `IsReady` vẫn trả true, `Show` tra `_banners[unitId]` thì không còn view. Banner cũng không bao giờ được load lại.
- **Cách fix:** xoá cờ khi load thất bại; trong `Show` dùng `TryGetValue`, không có thì báo `Failed` và xoá cờ.
- **Cách tránh ở game khác:**
  - Trạng thái "ready" của ad phải bị xoá ở mọi nhánh: load lỗi, hết hạn, đã show, bị SDK huỷ.
  - Trong code callback của SDK, không dùng indexer `dict[key]`; dùng `TryGetValue` và trả kết quả lỗi thay vì ném exception.

### 2.5 Máy không có Play Services dùng được
- **Triệu chứng:** NPE trong `ProxyBillingActivity.onCreate` (mua hàng) và `GamesResolutionActivity.onCreate` (đăng nhập Play Games). Thường do robot pre-launch report, nhưng người dùng thật trên máy Huawei, máy Trung Quốc, máy chưa cập nhật Play Services cũng gặp.
- **Vì sao:** SDK mở activity của Google mà không kiểm tra Play Services, activity đó crash và kéo cả app.
- **Cách fix:** gọi `GoogleApiAvailability.isGooglePlayServicesAvailable` qua JNI trước khi đăng nhập hoặc gọi `PurchaseProduct`. Không có thì báo lỗi rõ cho người chơi (`store_not_ready`), không gọi SDK.
  ```csharp
  using var api = new AndroidJavaClass("com.google.android.gms.common.GoogleApiAvailability");
  using var inst = api.CallStatic<AndroidJavaObject>("getInstance");
  using var player = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
  using var activity = player.GetStatic<AndroidJavaObject>("currentActivity");
  bool ok = inst.Call<int>("isGooglePlayServicesAvailable", activity) == 0; // ConnectionResult.SUCCESS
  ```
  Nếu chính lời kiểm tra ném exception thì cho luồng cũ chạy tiếp, đừng chặn người chơi chỉ vì guard hỏng.
- **Cách tránh ở game khác:** mọi chỗ mở activity của Google (billing, Games, review, update) đều qua một helper kiểm Play Services dùng chung.

### 2.6 Crash ngay khi mở bản release vì R8
- Chi tiết ở mục 2.13 của [unity-mobile-build-store.md](unity-mobile-build-store.md). Tóm tắt: R8 full mode (AGP 9) xoá constructor không tham số của `WorkDatabase_Impl`, WorkManager (được kéo vào bởi notification/Firebase) crash trong `androidx.startup`. Thêm `-keep class * extends androidx.room.RoomDatabase { <init>(); }`.
- **Cách tránh:** sau mỗi lần nâng AGP, Gradle hoặc bật minify, chạy **bản release ký thật** trên máy thật ít nhất tới màn chính. Bản debug và Editor không lộ lỗi này.

### 2.7 Gọi Crashlytics trước khi Firebase sẵn sàng
- **Triệu chứng:** logcat có lỗi trong static constructor của `PlatformAccessor`; sau đó Crashlytics im lặng cả phiên, không thấy crash nào được báo.
- **Vì sao:** code đặt `SetUserId`/`SetCustomKey`/`Log` ngay lúc khởi động, trước khi `FirebaseApp.CheckAndFixDependenciesAsync` xong. Static constructor ném exception một lần thì type đó hỏng tới khi tắt app.
- **Cách fix:** lớp bọc Crashlytics có cờ `_initialized`. Trước khi sẵn sàng, user id và custom key được giữ trong dictionary; `Initialize()` áp lại tất cả rồi mới bật cờ. Log và exception trước đó bỏ qua.
- **Cách tránh ở game khác:** mọi SDK có bước khởi tạo bất đồng bộ (Firebase, Ads, IAP, PlayFab) đều đi qua một lớp bọc có hàng chờ, không gọi thẳng từ `Awake` của các hệ thống khác.

### 2.8 Game đứng sau khi xem quảng cáo
- **Triệu chứng:** sau khi bấm xem ad, game dừng hẳn (combat, timer không chạy) dù ad đã tắt hoặc không hiện.
- **Vì sao:**
  1. Game bị pause hai chỗ: hàm `RewardedAds.Play` tự pause, và cầu nối pause theo event open/close của waterfall cũng pause. Khi waterfall còn đang tìm mạng có ad, game đã bị dừng; nếu không tìm được ad thì một bên không resume.
  2. Có SDK mở ad nhưng không bao giờ báo đóng (callback mất, activity bị hệ thống huỷ). Cờ `_busy` của waterfall và trạng thái pause kẹt mãi.
- **Cách fix:**
  - Chỉ một chỗ được pause game: theo event "ad đã hiện" và "ad đã đóng" của waterfall.
  - Watchdog: sau khi ad mở, đếm `showMaxOpenSeconds` (300 giây). Hết giờ mà SDK chưa báo đóng thì tự coi là `Closed` (không thưởng), mở khoá `_busy` và resume game.
- **Cách tránh ở game khác:** trạng thái "đang chờ callback của SDK" nào cũng phải có timeout. Pause/resume phải ghép cặp ở cùng một chỗ, đừng rải ở nhiều lớp.

### 2.9 Intro hoặc màn hình đứng, chạm không ăn
- **Triệu chứng:** intro kẹt ở trang 1 chỉ trên máy thật, Editor chạy bình thường.
- **Vì sao:**
  1. Tap đã được nhận (cờ đã đổi), rồi callback báo analytics/lưu tiến độ ném exception. Phần còn lại của `AdvanceFromTap` không chạy, nên lần chạm sau cũng không làm gì. Trong Editor Firebase không chạy thật nên không lộ.
  2. Panel debug vẽ bằng `OnGUI` của PlayFab bật trong cấu hình, che mép dưới màn hình và nuốt chạm vào nút "Tap to continue".
- **Cách fix:**
  - Analytics là best-effort: `Send` bọc `try/catch`, kể cả khi flush hàng chờ.
  - Controller gọi callback bên ngoài qua hàm bọc `try { cb?.Invoke(); } catch (Exception e) { Debug.LogException(e); }`.
  - Panel debug chỉ bật khi `Application.isEditor || Debug.isDebugBuild`, bất kể asset cấu hình ghi gì.
- **Cách tránh ở game khác:**
  - Luồng bắt buộc (intro, tutorial, nút tiếp tục) không được phụ thuộc vào kết quả của analytics, cloud hay ads.
  - Đổi trạng thái sau cùng, sau khi các lời gọi phụ đã chạy, hoặc bọc lời gọi phụ.
  - Mọi công cụ debug, cheat compile trong `#if UNITY_EDITOR || DEVELOPMENT_BUILD`.

### 2.10 Coroutine trên object đã inactive
- **Triệu chứng:** `Coroutine couldn't be started because the game object is inactive`.
- **Vì sao:** popup shop đóng xong vẫn bị gọi rebind banner (callback ad trả về muộn), và rebind gọi `StartCoroutine`.
- **Cách fix:** kiểm `isActiveAndEnabled` trước khi bắt đầu coroutine; bỏ qua nếu popup đã đóng.
- **Cách tránh:** mọi callback bất đồng bộ (ad, mạng, Addressables) trả về UI phải kiểm UI còn sống và còn hiện.

### 2.11 Cờ trạng thái kẹt khi có exception
- **Triệu chứng:** sau khi quay lại game từ offline, combat không chạy tiếp.
- **Vì sao:** offline catch-up chạy theo lát (sliced) trong coroutine lồng nhau (`yield return Areas.CatchUpRoutine(...)`). Khi routine con ném exception, Unity dừng cả chuỗi coroutine **mà không chạy `finally` của routine cha**, nên cờ `_offlineCatchUpRunning` kẹt ở true và các khu vực đứng yên.
- **Cách fix:** không `yield return` thẳng routine con; bọc nó bằng một iterator tự gọi `MoveNext()` trong `try/catch`, rồi mới trả cờ trong `finally` của routine cha.
  ```csharp
  // Tự chạy cả các routine lồng sâu hơn (yield return IEnumerator) bằng stack,
  // để exception ở tầng nào cũng bị bắt ở đây thay vì để Unity dừng cả chuỗi.
  static IEnumerator CatchExceptions(IEnumerator root)
  {
      var stack = new Stack<IEnumerator>();
      stack.Push(root);
      while (stack.Count > 0)
      {
          object current;
          try
          {
              if (!stack.Peek().MoveNext()) { stack.Pop(); continue; }
              current = stack.Peek().Current;
          }
          catch (Exception e) { Debug.LogException(e); yield break; }
          if (current is IEnumerator nested) { stack.Push(nested); continue; }
          yield return current; // WaitForSeconds, null...
      }
  }
  ```
- **Cách tránh:** mọi cờ khoá (`_busy`, `_saving`, `_catchingUp`, `_showingAd`) được trả trong `finally`, hoặc có timeout như mục 2.8. Với coroutine, nhớ `finally` không chạy khi coroutine con ném lỗi; với `async`/UniTask thì `finally` chạy bình thường.

### 2.12 Crash native lúc mở game trên máy ít RAM (chưa xử lý triệt để)
- **Triệu chứng:** SIGTRAP trong `DebugStringToFile` của `libunity.so`, khoảng 1 giây sau khi boot, lúc Unity đang nạp texture lên GPU. Máy bị crash còn 64–211 MB RAM trống (Xiaomi, vivo, Android 16).
- **Vì sao (giả thuyết mạnh):** hết bộ nhớ khi upload texture của màn đầu. Không phải lỗi C#.
- **Hướng giảm:**
  - Bớt texture phải nạp trước màn đầu: chuyển ảnh lớn của màn sau sang Addressables, nạp khi cần.
  - Nén ASTC (6x6 cho UI, 8x8 cho nền lớn nếu chấp nhận được chất lượng), tắt Read/Write, tắt mipmap cho UI.
  - Giới hạn texture upload mỗi frame (`QualitySettings.asyncUploadBufferSize`, `asyncUploadTimeSlice`) khi nạp scene.
  - Theo dõi theo dòng máy; nếu chỉ vài máy rất yếu thì chấp nhận được, không chặn phát hành.

---

## 3. Checklist phòng tránh cho game mới

Khi viết code:
1. Không `?.`, `??` với `UnityEngine.Object`. Bật Microsoft.Unity.Analyzers.
2. Mọi event của singleton/SDK có huỷ đăng ký trong `OnDestroy`/`Dispose`.
3. Mọi cờ khoá được trả trong `finally`; mọi chờ callback SDK có timeout.
4. Analytics, crash report, cloud là best-effort: bọc `try/catch`, không chặn gameplay.
5. Lưu chính ở `OnApplicationPause(true)`; quit/destroy không làm việc nặng.
6. SDK khởi tạo bất đồng bộ đi qua lớp bọc có hàng chờ.
7. Kiểm Play Services trước khi mở activity của Google.
8. Grant IAP idempotent theo transaction id; đọc `PendingOrders` khi store mới khởi tạo.
9. Callback bất đồng bộ trả về UI phải kiểm UI còn sống.
10. Công cụ debug chỉ compile trong Editor/Development build.

Trước khi phát hành:
1. Chạy bản release ký thật (IL2CPP + R8) trên máy thật, tới màn chính và mua thử một gói (license tester).
2. Thử: đổi ngôn ngữ/reload scene rồi mua hàng và xem ad; tắt mạng giữa lúc xem ad; đưa app ra nền rồi để hệ thống đóng (Developer options → Don't keep activities hoặc `adb shell am kill`).
3. Thử trên một máy yếu, RAM 2–3 GB.
4. Đã upload native symbols và `mapping.txt` cho bản này.

Sau khi phát hành:
1. Đọc Crashlytics sau 24–48 giờ và sau 7 ngày; phân loại theo mục 0.
2. Lỗi có từ 2 người dùng trở lên và stack trong code của mình: sửa ở hàm dùng chung, không chỉ ở đường gọi bị báo.
3. Ghi lỗi mới vào file này.
