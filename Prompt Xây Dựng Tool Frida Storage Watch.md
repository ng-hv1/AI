# MASTER PROMPT: XÂY DỰNG CLI TOOL `frida-storage-watch` QUÉT STORAGE LEAK ANDROID THEO TESTCASE

## 1. VAI TRÒ & MỤC TIÊU DỰ ÁN
Bạn là một Senior Android Runtime Instrumentation Engineer và Python CLI Architect.
Nhiệm vụ của bạn là xây dựng hoàn chỉnh một công cụ dòng lệnh (CLI) gọn nhẹ, chạy ổn định ở mức production có tên là **`frida-storage-watch`**.

**Bài toán cần giải quyết:**
Khi kiểm thử ứng dụng Android (thuần Kotlin/Java) theo từng Testcase (bấm tay thủ công hoặc chạy Automation Test như Maestro/Appium/ADB), kỹ sư cần:
1. **Trong khi chạy Testcase (Real-time Callsite Tracking):** Theo dõi trực tiếp theo thời gian thực mọi file được ứng dụng tạo ra, sao chép, đổi tên hoặc xóa đi, đồng thời chỉ ra **chính xác tên file và dòng code (`Class.kt:Line` hoặc `Class.java:Line`)** trong dự án đã kích hoạt hành vi đó (mà không cần sửa mã nguồn hay build lại APK).
2. **Sau khi kết thúc Testcase (Post-Testcase Leak Audit):** Đối soát trạng thái bộ nhớ với ổ cứng thực tế trên thiết bị để phát hiện và cảnh báo chính xác các lỗi rò rỉ lưu trữ (**Storage Leak**) và rò rỉ luồng I/O (**Unclosed Stream / FD Leak**), xuất báo cáo trực quan trên Terminal và file JSON chuẩn hóa.

---

## 2. RÀNG BUỘC KỸ THUẬT & TƯƠNG THÍCH HỆ THỐNG
1. **Ngôn ngữ & Môi trường Host:**
   - Python 3.10+ chạy trên macOS / Linux.
   - Chỉ sử dụng thư viện chuẩn của Python (`argparse`, `subprocess`, `json`, `pathlib`, `time`, `sys`, `dataclasses`) kết hợp với `frida` và `frida-tools`. Không cài thêm các dependency nặng không cần thiết.
2. **Tương thích kép Frida 17+ và Frida 16 (BẮT BUỘC):**
   - Từ **Frida 17+**, `Java` bridge đã tách khỏi core runtime thành package riêng. Hệ thống build agent phải kiểm tra phiên bản `frida.__version__`:
     - Nếu `major >= 17`: Tự động kiểm tra thư mục `node_modules/frida-java-bridge`; nếu chưa có thì tự gọi `frida-pm install frida-java-bridge`, sau đó dùng `frida.Compiler().build(...)` để bundle file agent có dòng `import Java from "frida-java-bridge";`.
     - Nếu `major <= 16`: Tự động loại bỏ dòng `import Java ...` để dùng biến toàn cục `Java` sẵn có của Frida 16.
3. **Đối tượng mục tiêu (Target App):**
   - Ứng dụng Android thuần Kotlin/Java (Android 8.0 / API 26 đến Android 15 / API 35+).
   - Giao tiếp qua `frida-server` (máy ảo Emulator / máy đã Root) hoặc `libfrida-gadget.so` (máy thật không Root).
4. **Nguyên tắc Fail-Open (An toàn tuyệt đối cho App đang test):**
   - Mọi logic theo dõi bên trong các hàm hook của Frida Agent phải được bọc `try-catch` cẩn thận để đảm bảo nếu việc đọc Stack Trace hay chuẩn hóa đường dẫn gặp lỗi biên, hàm gốc của hệ điều hành vẫn được gọi và trả về bình thường, **tuyệt đối không làm crash ứng dụng mục tiêu** (trừ khi người dùng chủ động bật cờ `--inject-fault`).

---

## 3. CẤU TRÚC THƯ MỤC DỰ ÁN CẦN TẠO
Hãy tạo đầy đủ toàn bộ các file theo cấu trúc sạch sẽ, rõ ràng sau (không để lại TODO hay code giả lập):

~~~text
frida-storage-watch/
├── pyproject.toml                 # Khai báo package và CLI entrypoint `frida-storage-watch`
├── requirements.txt               # frida>=16.0.0, frida-tools>=12.0.0
├── README.md                      # Hướng dẫn cài đặt frida-server/gadget và các ví dụ chạy CLI
└── frida_storage_watch/
    ├── __init__.py
    ├── __main__.py                # Cho phép chạy qua `python3 -m frida_storage_watch`
    ├── cli.py                     # Xử lý tham số CLI, điều phối vòng đời Testcase & Exit codes
    ├── loader.py                  # Kết nối thiết bị USB/Serial, xử lý Frida 16/17 compiler & attach/spawn
    ├── reporter.py                # Định dạng màu ANSI log thời gian thực, bảng tổng kết & xuất JSON
    └── agent/
        └── agent.js               # Frida JavaScript Agent (Shadow Ledger + 8 AOSP Choke-Point Hooks)
~~~

---

## 4. ĐẶC TẢ CHI TIẾT PHẦN 1: FRIDA AGENT (`agent/agent.js`)

### 4.1. Cơ chế Sổ cái đối soát (`Shadow Ledger`) trong RAM
Duy trì trạng thái theo từng Testcase gồm 4 cấu trúc dữ liệu:
1. `files` (`Map<canonicalPath, FileEntry>`):
   - `path`: Đường dẫn đã chuẩn hóa.
   - `isEphemeral`: Boolean xác định file có thuộc diện tạm thời bắt buộc phải xóa sau testcase hay không.
   - `state`: `"ACTIVE"` | `"SILENT_DELETE_FAILURE"` | `"DEFERRED_DELETE_ON_EXIT"`.
   - `createdAt`: Timestamp (ms).
   - `sourceApi`: Tên cổ phễu phát hiện lệnh tạo file.
   - `primaryCallsite`: Dòng code đầu tiên thuộc package của app (`com.example.MyClass.myMethod(MyClass.kt:42)`).
   - `appStack`: Mảng tối đa 8 stack frames thuộc app (hoặc thư viện 3rd-party nếu không có frame của app).
   - `openFds`: `Map<fdInt, openCallsite>` lưu các File Descriptor hiện chưa đóng của file này.
   - `copiedTo`: Mảng các đường dẫn đích nếu dữ liệu của file này đã được sao chép sang file khác.
   - `notes`: Mảng ghi chú lịch sử biến đổi (rename, copy, lỗi xóa).
2. `activeFds` (`Map<fdInt, { path, isReadOnly, threadId, callsite }>`):
   - Ánh xạ số nguyên File Descriptor sang đường dẫn file để nối mạch giữa `open` -> `write` -> `close`.
   - Cách trích xuất `fdInt` từ `java.io.FileDescriptor`: Gọi `fdObj.getInt$()`, có fallback đọc trường `descriptor` nếu cần.
3. `pendingUris` (`Map<uriString, { uri, createdAt, primaryCallsite, appStack }>`):
   - Lưu các URI trên `MediaStore` được `insert` với cờ `is_pending = 1`.
4. `faultConfig` (`{ enabled, matchSubstring, failAfterWrites, writeCount, triggered }`):
   - Cấu hình tiêm lỗi cưỡng bức `java.io.IOException` để test nhánh `catch` / `finally`.

### 4.2. Chuẩn hóa Đường dẫn, Lọc Nhiễu & Bóc tách Callsite
1. **Chuẩn hóa Symlink Android (`normalizePath`):**
   - Trên Android, `/data/data/<pkg>` là symlink của `/data/user/0/<pkg>`, và `/sdcard` là symlink của `/storage/emulated/0`.
   - Phải luôn chuẩn hóa bằng `File.$new(rawPath).getCanonicalPath()` (fallback về `getAbsolutePath()`) trước khi làm Key trong `Shadow Ledger`. Nếu không chuẩn hóa, lúc tạo ở `/data/data/...` và lúc xóa ở `/data/user/0/...` sẽ lệch Key và gây False Positive!
2. **Bộ lọc Nhiễu hệ thống (`isSystemNoise`):**
   - Bỏ qua hoàn toàn các đường dẫn chứa: `"/proc/"`, `"/sys/"`, `"/dev/"`, `"/system/"`, `"/apex/"`, `"/vendor/"`, `"/code_cache/"`, `"/app_webview/"`, `"/shared_prefs/"`.
   - Bỏ qua các hậu tố nội bộ của SQLite/ART: `"-journal"`, `"-wal"`, `"-shm"`, `".odex"`, `".vdex"`, `".art"`, `".prof"`.
   - Chỉ theo dõi các file nằm trong thư mục dữ liệu của `targetPackage`, các thư mục `extraWatchDirs` do user truyền vào, hoặc bộ nhớ ngoài (`/sdcard/`, `/storage/`).
3. **Phân loại File Tạm (`isEphemeralPath`):**
   - Trả về `true` nếu được tạo từ API `createTempFile`, HOẶC đường dẫn chứa `"/cache/"`, `"/external_cache/"`, `"/tmp"`, `"/temp"`, HOẶC kết thúc bằng `".tmp"`, `".temp"`, `".part"`, `".download"`.
4. **Bóc tách Callsite Thông minh (`captureCallsite`):**
   - Chụp Stack Trace qua `android.util.Log.getStackTraceString(Java.use("java.lang.Exception").$new())`.
   - Lọc bỏ các frame hệ thống (`libcore.*`, `java.*`, `javax.*`, `sun.*`, `android.*`, `com.android.*`, `dalvik.*`, `kotlin.io.*`, `kotlin.coroutines.*`).
   - Tìm frame đầu tiên bắt đầu bằng `targetPackage` làm `primaryCallsite`. Nếu toàn bộ luồng do thư viện bên thứ ba chạy ngầm (như Glide/OkHttp/WorkManager), gán `primaryCallsite = "[3rd-Party] " + firstThirdPartyFrame`.

### 4.3. Danh sách 8 Chốt chặn Cổ phễu AOSP (BẮT BUỘC CÀI ĐẶT ĐỦ)
Không hook dàn trải hàng chục hàm bề mặt; hãy đặt hook chính xác tại 8 cổ phễu hội tụ sau của Android AOSP (lưu ý duyệt `.overloads` an toàn):

1. **Chốt 1 — `libcore.io.IoBridge.open(String, int, int)`:**
   - Bắt toàn bộ `FileOutputStream`, `FileInputStream`, `RandomAccessFile`, `FileWriter`, Kotlin `writeBytes`/`writeText`/`copyTo`, `Okio`, `OkHttp`, `Glide`.
   - Nếu `(flags & OsConstants.O_CREAT.value) !== 0`: Ghi nhận sự kiện `CREATE`.
   - **Phát hiện tự động Copy Staging Trap:** Kiểm tra `Thread.currentThread().getId()`. Nếu cùng một Thread đang mở file mới ở chế độ GHI (`(flags & O_ACCMODE) !== O_RDONLY`) trong khi `activeFds` vẫn đang giữ một FD mở ở chế độ ĐỌC (`O_RDONLY`) trỏ tới một file trong `files` Map, lập tức ghi nhận file nguồn đã được copy sang file đích (`copiedTo.push(dstPath)`) và bắn sự kiện `COPY`.
2. **Chốt 2 — `java.io.UnixFileSystem.createFileExclusively` (duyệt mọi overloads):**
   - Bắt 100% lệnh gọi `File.createTempFile()`, `File.createNewFile()` và Kotlin `createTempFile()` khi ứng dụng tạo vỏ file rỗng mà chưa mở Stream.
3. **Chốt 3 — `sun.nio.fs.UnixChannelFactory.open` (bọc `try-catch`, duyệt mọi overloads):**
   - Bắt 100% nhánh Java NIO.2 (`java.nio.file.Files.createTempFile`, `Files.createFile`, `Files.write`, `Files.newByteChannel`).
4. **Chốt 4 — Đóng File Descriptor:**
   - Hook `libcore.io.IoBridge.closeAndSignalBlockedThreads` và (trong khối `try-catch`) `sun.nio.ch.FileChannelImpl.implCloseChannel`.
   - Khi FD đóng, xóa `fdInt` khỏi `activeFds` và khỏi `fileEntry.openFds`.
5. **Chốt 5 — Đổi tên, Di chuyển & Copy NIO (Chống báo lỗi giả - False Positive):**
   - Hook `java.io.File.renameTo(File)` và `java.nio.file.Files.move(...)`:
     - Khi thành công và `srcPath` nằm trong `files` Map: Xóa `srcPath` khỏi Map. Cập nhật lại thuộc tính `isEphemeral` theo `dstPath`, lưu tiếp dưới khóa `dstPath` và bắn sự kiện `RENAME` (nhờ vậy khi app tải vào `.tmp` rồi đổi tên thành `.mp4`/`.jpg` trong thư mục bền vững, nó được đánh dấu đã promote hợp lệ chứ không báo rò rỉ file `.tmp`).
   - Hook thêm `java.nio.file.Files.copy` (nếu đối số đầu là `Path` nằm trong `files` Map) để ghi nhận vào `copiedTo`.
6. **Chốt 6 — Xóa File & Bẫy `deleteOnExit`:**
   - Hook `java.io.File.delete()`, `java.nio.file.Files.delete(Path)`, `java.nio.file.Files.deleteIfExists(Path)`:
     - Nếu xóa thành công: Xóa khỏi `files` Map và bắn sự kiện `DELETE` (nếu lúc xóa mà `openFds.size > 0` thì đính kèm cảnh báo xóa khi chưa đóng Stream).
     - Nếu `File.delete()` trả về `false` nhưng `this.exists()` vẫn là `true`: Chuyển `state = "SILENT_DELETE_FAILURE"` và bắn sự kiện cảnh báo đỏ `DELETE_FAILED` ngay lập tức.
   - Hook `java.io.File.deleteOnExit()`:
     - Nếu file đang nằm trong `files` Map, đánh dấu `state = "DEFERRED_DELETE_ON_EXIT"` và bắn cảnh báo `DELETE_ON_EXIT`.
7. **Chốt 7 — Scoped Storage `MediaStore` (`android.content.ContentResolver`):**
   - Hook `insert(Uri, ContentValues)`: Nếu `values.containsKey("is_pending")` và giá trị bằng `1`, lưu URI vào `pendingUris` và bắn sự kiện `CREATE_URI`.
   - Hook `update(...)`: Nếu hạ `is_pending` về `0`, xóa URI khỏi `pendingUris` và bắn sự kiện `COMMIT_URI`.
   - Hook `delete(...)`: Xóa URI khỏi `pendingUris` và bắn sự kiện `DELETE_URI`.
8. **Chốt 8 — Tiêm lỗi cưỡng bức tại `libcore.io.IoBridge.write(FileDescriptor, byte[], int, int)`:**
   - Nếu `faultConfig.enabled == true` và đường dẫn của `fdObj` chứa `faultConfig.matchSubstring`: Tăng bộ đếm ghi; khi đạt ngưỡng `failAfterWrites`, bắn sự kiện `FAULT_INJECTED` và ném `throw IOException.$new("Frida Simulated Disk Full Error")` để ép luồng chạy của app nhảy vào nhánh `catch`.

### 4.4. Bộ phân loại 6 Nhãn Lỗi khi Kiểm toán (`stopAndAudit` RPC)
Khi kết thúc Testcase, dùng `Java.perform` kiểm tra thực tế từng mục còn lại trên ổ cứng (`File.exists()` và `File.length()`), phân loại chính xác thành các mã lỗi sau:
1. `SILENT_DELETE_FAILURE`: Đã gọi `delete()` nhưng trả về `false`, file thực tế vẫn còn nằm trên ổ cứng.
2. `DEFERRED_DELETE_ON_EXIT`: Dùng `deleteOnExit()` khiến file rác bị ngâm trên ổ cứng cho đến khi tắt hẳn tiến trình.
3. `COPY_STAGING_LEAK`: File tạm đã được chép nội dung sang file đích (`copiedTo.length > 0`) nhưng file tạm gốc bị bỏ quên trên đĩa.
4. `ORPHANED_TEMP_FILE`: File tạm (`isEphemeral == true`) vẫn tồn tại trên ổ cứng sau khi kết thúc testcase.
5. `UNCLOSED_STREAM`: Còn ít nhất 1 File Descriptor (`openFds.size > 0`) chưa gọi `.close()`.
6. `ORPHANED_SCOPED_URI`: Bản ghi `MediaStore` vẫn bị treo ở trạng thái `is_pending = 1`.
7. *(Tùy chọn)* `UNEXPECTED_PERSISTENT_FILE`: Khi người dùng bật cờ `--strict-cancel` (dành cho các testcase kiểm thử nút "Hủy / Cancel"), mọi file bền vững mới sinh ra trong `/files/` hoặc `/sdcard/` chưa được xóa cũng bị đánh dấu là Leak.

---

## 5. ĐẶC TẢ CHI TIẾT PHẦN 2: PYTHON CLI & LUỒNG CHẠY TESTCASE (`cli.py`)

### 5.1. Giao diện Tham số Dòng lệnh (CLI Flags)
Thiết kế CLI hỗ trợ đầy đủ các tham số sau:
- `-p`, `--package <PACKAGE_NAME>` *(Bắt buộc)*: Package name của ứng dụng (ví dụ: `com.example.app`).
- `-d`, `--device <SERIAL>`: Chỉ định thiết bị ADB cụ thể theo serial ID; nếu bỏ trống thì kết nối thiết bị USB mặc định (`frida.get_usb_device`).
- `--spawn`: Khởi động lại ứng dụng từ đầu (Cold Start) và gắn hook trước khi dòng code đầu tiên chạy. Nếu không truyền `--spawn`, tự động tìm PID đang chạy của package để attach.
- `--cmd "<SHELL_COMMAND>"`: Chạy tự động một lệnh kiểm thử bên ngoài (ví dụ: `"maestro test flow.yaml"` hoặc `"adb shell am instrument ..."`). Tự động bắt đầu ghi nhận trước khi chạy lệnh và tự động quét Leak sau khi lệnh kết thúc.
- `--grace <SECONDS>`: Thời gian chờ (mặc định `2.0` giây) sau khi kết thúc thao tác để các luồng dọn rác ngầm trên `Dispatchers.IO` / `Executor` hoàn tất trước khi chốt sổ kiểm toán.
- `--inject-fault <PATH_SUBSTRING>`: Bật chế độ tiêm lỗi `IOException` vào các file có đường dẫn chứa chuỗi này nhằm kiểm chứng khối `try-catch-finally`.
- `--fail-after <INT>`: Số lần gọi `write()` trước khi kích hoạt tiêm lỗi (mặc định `1`).
- `--strict-cancel`: Đánh dấu cả các file lưu trữ bền vững mới tạo ra nhưng không xóa là Leak (dùng cho Testcase hủy thao tác).
- `-o`, `--output <FILE.json>`: Đường dẫn lưu báo cáo JSON chi tiết.

### 5.2. Hai Chế độ Vận hành mượt mà
1. **Chế độ Tương tác Nhiều Testcase Liên tiếp (Interactive QA Loop — khi không truyền `--cmd`):**
   - Kết nối vào ứng dụng **1 lần duy nhất**.
   - Nếu attach vào app đang chạy: Chờ người dùng điều hướng tới trước màn hình cần test, nhấn `[Enter]` để bắt đầu **Testcase #1**.
   - Trong lúc người dùng thao tác trên điện thoại: In log thời gian thực có màu sắc phân biệt rõ `[+ CREATE TEMP]`, `[+ CREATE FILE]`, `[~ COPY]`, `[~ RENAME]`, `[- CLEANED]`, `[! DELETE_FAILED]` kèm dòng code `↳ Tạo tại: Class.kt:Line`.
   - Khi người dùng nhấn `[Enter]` để kết thúc Testcase: Đợi `--grace` giây, chạy `stopAndAudit()`, in bảng báo cáo chi tiết của Testcase đó.
   - Sau khi in báo cáo xong, hỏi người dùng: **`Nhấn [Enter] để bắt đầu Testcase tiếp theo (reset Sổ cái), hoặc gõ 'q' để thoát:`**. Tính năng này giúp QA test liên tục 10–20 testcase mà không phải attach lại Frida nhiều lần.
2. **Chế độ Tự động hóa CI/CD (Headless Mode — khi truyền `--cmd`):**
   - Không dừng hỏi `input()`. Tự động `startTestcase()` -> chạy lệnh `--cmd` -> đợi `--grace` giây -> `stopAndAudit()` -> xuất file JSON (nếu có `-o`) -> thoát với **Exit Code chuẩn**:
     - `0`: PASS (Không phát hiện rò rỉ).
     - `1`: FAIL (Phát hiện ít nhất 1 Storage/Stream Leak).
     - `2`: Lỗi môi trường / kết nối (chưa bật `frida-server`, không tìm thấy thiết bị, sai package name).

---

## 6. ĐỊNH DẠNG BÁO CÁO JSON CHUẨN (`--output`)
File JSON xuất ra phải tuân thủ cấu trúc rõ ràng sau để dễ dàng tích hợp vào Dashboard hoặc pipeline phân tích tự động:

~~~json
{
  "metadata": {
    "packageName": "com.example.app",
    "deviceId": "emulator-5554",
    "fridaVersion": "17.x.x",
    "strictCancelMode": false,
    "faultInjection": {
      "enabled": false,
      "matchSubstring": ""
    },
    "gracePeriodSeconds": 2.0,
    "timestamp": "2026-11-20T14:30:00Z"
  },
  "testcases": [
    {
      "testcaseIndex": 1,
      "passed": false,
      "leaksCount": 1,
      "persistentFilesCount": 0,
      "totalLeakedBytes": 524288,
      "leaks": [
        {
          "path": "/data/user/0/com.example.app/cache/export_raw_9182.tmp",
          "existsOnDisk": true,
          "sizeBytes": 524288,
          "sourceApi": "UnixFileSystem.createFileExclusively",
          "createdAtCallsite": "com.example.app.video.VideoExporter.prepareTemp(VideoExporter.kt:84)",
          "appStack": [
            "com.example.app.video.VideoExporter.prepareTemp(VideoExporter.kt:84)",
            "com.example.app.video.VideoViewModel.exportClip(VideoViewModel.kt:129)"
          ],
          "unclosedStreams": [],
          "copiedTo": [
            "/data/user/0/com.example.app/files/exported/final_clip.mp4"
          ],
          "leakReasons": [
            "COPY_STAGING_LEAK"
          ],
          "messages": [
            "Đã chép dữ liệu sang [/data/user/0/com.example.app/files/exported/final_clip.mp4] nhưng bỏ quên file tạm gốc"
          ]
        }
      ],
      "persistentFiles": []
    }
  ]
}
~~~

---

## 7. QUY TRÌNH THỰC HIỆN & TIÊU CHÍ NGHIỆM THU (DEFINITION OF DONE)
Hãy thực hiện tuần tự các bước sau:
1. Khởi tạo cấu trúc thư mục và viết `pyproject.toml` + `requirements.txt`.
2. Viết `frida_storage_watch/agent/agent.js` đảm bảo đầy đủ 8 chốt chặn AOSP, chuẩn hóa `getCanonicalPath()`, lọc nhiễu SQLite/hệ thống, bóc tách `primaryCallsite` và bọc `try-catch` an toàn (fail-open).
3. Viết `frida_storage_watch/loader.py`, `reporter.py`, `cli.py` và `__main__.py`. Xử lý gọn gàng các thông báo lỗi tiếng Việt dễ hiểu khi thiết bị chưa cắm cáp USB, chưa chạy `frida-server`, hoặc ứng dụng chưa mở.
4. Viết `README.md` hướng dẫn ngắn gọn cách cài đặt, cách đẩy `frida-server` lên máy ảo Android Emulator chỉ trong 4 dòng lệnh ADB, và 4 ví dụ sử dụng thực tế (Interactive QA, Cold Start `--spawn`, Fault Injection `--inject-fault`, và CI Automation `--cmd`).
5. **Tự kiểm tra chất lượng trước khi bàn giao:**
   - Chạy `python3 -m py_compile` trên toàn bộ các file `.py` để đảm bảo không có bất kỳ lỗi cú pháp nào.
   - Kiểm tra `python3 -m frida_storage_watch --help` hiển thị đầy đủ hướng dẫn sử dụng.
   - Rà soát lại code JavaScript trong `agent.js` để chắc chắn mọi phương thức RPC bất đồng bộ (`stopAndAudit`) đều trả về giá trị tuần tự hóa JSON hợp lệ (không trả về trực tiếp đối tượng `Map` hay Java Wrapper thô của Frida qua RPC).
~~~