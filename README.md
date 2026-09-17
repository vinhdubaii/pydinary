# Pydinary App (desktop, Tauri)

Bản desktop Windows của [Pydinary – Music Player](https://github.com/vinhdubaii/pydinary-web),
đóng gói bằng Tauri v2.

Repo này **không chứa và không clone** source của pydinary-web. Cửa sổ app
load thẳng `https://pydinary.pages.dev/` (bản đang chạy thật trên Cloudflare
Pages) — y hệt việc mở link đó trong trình duyệt, chỉ là không có khung
trình duyệt. Đổi gì bên pydinary-web (deploy lên Cloudflare Pages) là **tất
cả user desktop thấy ngay lập tức**, không cần build/release lại app desktop.
App desktop từ giờ chỉ cần release khi đổi phần khung native (titlebar,
icon, updater, kích thước cửa sổ...).

Titlebar tự vẽ (3 nút minimize/maximize/close + F11/Esc fullscreen) được
nhúng thẳng vào binary lúc compile (`include_str!`) và tiêm vào trang bằng
`initialization_script` của Tauri — chạy trước khi trang được parse, xem
`src-tauri/src/main.rs`.

## Cấu trúc

```
titlebar/                 Titlebar tự vẽ (html/css/js) — nhúng vào binary,
                            tiêm vào trang lúc runtime (xem main.rs)
src-tauri/
  src/main.rs             Tự tạo cửa sổ (để tiêm titlebar script) +
                            tự kiểm tra/cài update im lặng
  tauri.conf.json          url trỏ thẳng pydinary.pages.dev, decorations:false,
                            min-size 1280x800, cấu hình updater
  capabilities/default.json  Quyền minimize/maximize/close/kéo cửa sổ/fullscreen
                              — cấp cho origin pydinary.pages.dev (remote)
  icons/                   Sinh ra lúc build từ img/logo.png, KHÔNG commit
img/logo.png              Logo nguồn (PNG vuông) để sinh bộ icon
scripts/gen-icons.mjs     Gọi `tauri icon` để sinh src-tauri/icons/
```

## Cài đặt lần đầu

Cần: Node 20+, Rust stable, Git. Trên Windows, WebView2 runtime
(thường có sẵn từ Win10/11, Tauri tự cài kèm nếu thiếu lúc chạy installer).

```bash
npm install
```

### Icon

Bộ icon **không commit vào git** — nó được sinh tự động từ `img/logo.png`.

Việc bạn cần làm: đặt logo vào `img/logo.png` (PNG vuông, nên 1024×1024, có
alpha). Ví dụ dùng chung branding với pydinary-web:

```bash
curl -o img/logo.png https://raw.githubusercontent.com/vinhdubaii/pydinary-web/main/icon-512.png
```

Sau đó `scripts/gen-icons.mjs` sẽ gọi `tauri icon` để sinh toàn bộ
`src-tauri/icons/` (bao gồm 4 file khai trong `tauri.conf.json`). Script này
được khai thẳng trong `beforeDevCommand`/`beforeBuildCommand` của
`tauri.conf.json` — nên `dev`, `build` và cả GitHub Actions đều tự lo, không
cần chạy tay.

Script có cache: nó lưu hash của `logo.png` vào `src-tauri/icons/.source-hash`
và chỉ sinh lại khi logo thay đổi. Muốn chạy riêng thì `npm run gen-icons`.

## Chạy thử / build

```bash
npm run tauri dev      # chạy thử, tự load thẳng pydinary.pages.dev
npm run tauri build    # build .msi/.exe, ra trong src-tauri/target/release/bundle/
```

Cần internet cả lúc dev lẫn lúc chạy app thật — cửa sổ load thẳng URL sống,
không có gì được đóng gói sẵn để chạy offline. Mất mạng thì trắng trang
(WebView2 tự hiện lỗi mặc định của nó), không có màn hình fallback riêng.

## Cửa sổ: kích thước & fullscreen

Cửa sổ khoá kích thước tối thiểu 1280×800 (`minWidth`/`minHeight` trong
`tauri.conf.json`), vẫn resize được lớn hơn thoải mái (`resizable: true`).

`F11` bật/tắt fullscreen thật của hệ điều hành (khác với "chế độ fullscreen
player" riêng của web, nút `.fp-collapse`). `Esc` chỉ thoát fullscreen (không
bao giờ bật). Lúc fullscreen, 3 nút minimize/maximize/close tự vẽ tự ẩn đi.
Toàn bộ logic nằm trong `titlebar/titlebar.js` + `titlebar/titlebar.css`.

## Auto-update

App tự kiểm tra bản mới **1 lần duy nhất mỗi lúc vừa mở** (không lặp lại định
kỳ trong lúc đang chạy, để không bao giờ cắt ngang giữa lúc đang nghe nhạc).
Nếu có bản mới: tự tải và cài hoàn toàn im lặng (`installMode: "quiet"`,
không hiện gì cả). Bản cập nhật có hiệu lực ở **lần mở app kế tiếp**
(`restart_after_install(false)` trong `main.rs`) — không tự bật lại app ngay
sau khi cài.

Nguồn kiểm tra là `releases/latest/download/latest.json` trên GitHub — API
này **chỉ thấy release đã bấm Publish**, bỏ qua release còn ở dạng Draft. Nên
quy trình "release.yml tạo draft, tự tay viết note rồi bấm Publish" vẫn giữ
nguyên, và tự nhiên trở thành bước kiểm duyệt trước khi người dùng nhận được
bản cập nhật.

**Bắt buộc phải thiết lập trước khi push bất kỳ commit nào** (kể cả build
debug thường ngày cũng cần, vì `bundle.createUpdaterArtifacts: true` đòi hỏi
key này để build) — vào repo trên GitHub → Settings → Secrets and variables →
Actions → New repository secret, tạo 2 secret:

- `TAURI_SIGNING_PRIVATE_KEY` — private key dùng để ký update (giữ bí mật
  tuyệt đối, mất key này là không thể phát hành update mới cho người dùng cũ
  được nữa).
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` — mật khẩu của private key ở trên.

Public key tương ứng đã nhét sẵn vào `plugins.updater.pubkey` trong
`tauri.conf.json` (an toàn khi public, không cần giữ bí mật).

Nếu muốn tạo lại cặp key khác (vd làm mất key cũ):

```bash
npx tauri signer generate -w ./pydinary-updater.key
```

rồi copy nội dung file `.key` (private) + mật khẩu vào 2 secret trên, và nội
dung file `.key.pub` (public) vào `plugins.updater.pubkey`. **Tuyệt đối không
commit file `.key` vào git.**

## Web: load trực tiếp, không đóng gói

Cửa sổ "main" trong `tauri.conf.json` có `"url": "https://pydinary.pages.dev/"`
và `"create": false` — Tauri không tự tạo cửa sổ này, `main.rs` tự tạo lại
(y hệt config, không đổi gì) chỉ để gắn thêm `initialization_script()` (chèn
titlebar), việc mà JSON config không làm được.

**Đánh đổi bảo mật cần biết**: Tauri v2 mặc định không cấp quyền IPC nào cho
nội dung tải từ domain ngoài. Để 3 nút titlebar + F11 hoạt động,
`capabilities/default.json` phải khai rõ `"remote": { "urls":
["https://pydinary.pages.dev/*"] }` — nghĩa là bất kỳ script nào chạy được
trên trang đó (kể cả do lỗ hổng XSS nào đó, giả sử có) đều có thể gọi các
lệnh minimize/maximize/close/fullscreen của cửa sổ app. Rủi ro thấp (chỉ là
điều khiển cửa sổ, không phải đọc file hay chạy lệnh hệ thống) nhưng là thật,
không phải lý thuyết — cần nhớ nếu sau này cấp thêm quyền khác cho domain
này.

## Vấn đề đã biết: titlebar chồng lên nút có sẵn của web

Ở màn hình fullscreen player, pydinary-web tự vẽ 2 nút riêng ở góc phải
trên: nút thu nhỏ (`.fp-collapse`, `top:1.5rem;right:1.5rem`, 40×40px) và
nút lyrics (`.fp-lyrics-toggle`, `top:1.5rem;right:4.75rem`). Titlebar của
app (cao 36px, 3 nút ở góc phải) và 2 nút đó **có vùng chồng nhau khoảng
12px** ở mép trên (titlebar chiếm y:0–36px, `.fp-collapse` chiếm y:24–64px,
cùng nằm trong dải x từ mép phải mà titlebar phủ tới).

Ảnh hưởng thực tế: click vào đúng 12px mép trên cùng của nút `.fp-collapse`
có thể bị nút titlebar "cướp" thay vì bấm được nút gốc; phần thân nút vẫn
bấm bình thường. Đã build thử và patch thật (không phải đoán suông) để xác
nhận đúng là có chồng, nhưng chưa chạy được trên máy Windows thật để xem
mức độ khó chịu ra sao.

2 hướng xử lý, tuỳ bạn chọn khi build xong nhìn thấy có khó chịu không:
- Không đụng gì — 12px là rất nhỏ, có thể chấp nhận được.
- Chỉnh `top` của `.fp-collapse`/`.fp-lyrics-toggle` trong pydinary-web
  (vd `1.5rem` → `3rem`) để chúng nằm hẳn dưới titlebar — nhưng như vậy là
  sửa source pydinary-web, phá nguyên tắc "không liên quan tới web" đã
  thống nhất.
