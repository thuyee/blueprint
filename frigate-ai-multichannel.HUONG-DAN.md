# Blueprint GỌN: **Frigate AI Task – Thông báo đa kênh**

## 0. TÓM TẮT NHANH (trạng thái 26/09/2026)

| Câu hỏi | Trả lời |
|---|---|
| AI phân tích **ảnh hay video**? | Tuỳ input **`ai_task_media`**: `snapshot` = ảnh (~5s) · **`clip` = video clip của Frigate (~9s, hiểu hành động + hướng di chuyển tốt nhất)** · `both` = cả hai. Automation camera Sân **đang để `snapshot`** (đã test: chạy 28 giây, AI đọc snapshot của Frigate) |
| Media của **Frigate hay HA tự chụp/quay**? | **100% của Frigate.** Đã bỏ hoàn toàn `camera.snapshot` / `camera.record` (grep = 0). HA chỉ **tải** snapshot/clip của Frigate về `/media` vì `ai_task` chỉ nhận media cục bộ |
| Có **tự xoá** chống tràn bộ nhớ? | **Không cần xoá**: tên file đặt theo camera nên mỗi sự kiện **ghi đè** file cũ ⇒ tối đa **2 file/camera (~1.7 MB)**. Kho của Frigate do Frigate tự xoá theo retention (snapshots 10 ngày, recordings 3 ngày) |

Chi tiết:
- Ảnh AI dùng: `https://ngocthuyhome.com/api/frigate/notifications/<id>/snapshot.jpg` → `/media/snapshots/frigate_ai_<cam>_1.jpg`
- Video AI dùng: `https://ngocthuyhome.com/api/frigate/notifications/<id>/<cam>/clip.mp4` → `/media/snapshots/frigate_ai_<cam>_rec.mp4`
- Ảnh trong thông báo (Zalo/điện thoại/Telegram/Hermes/Harness) = **snapshot của Frigate**

Bản gọn (26 mục cấu hình / 4 nhóm) — thay cho bản đầy đủ 93 mục.
Luồng: **Frigate phát hiện (person/dog/cat/face) → chụp ảnh + (tùy chọn) video ngắn → AI Task phân tích
→ gửi ẢNH + MÔ TẢ CỦA AI tới Điện thoại / Zalo / Telegram / Hermes / Harness.**
Nếu Double Take/Frigate đã nhận diện khuôn mặt (Ngọc Thụy, Mẹ…) thì **AI biết đó là người đã được xác định**
và dùng đúng tên.

| | |
|---|---|
| Blueprint | `/config/blueprints/automation/thuyee/frigate-ai-multichannel.yaml` (1.473 dòng, 72 KB) |
| Tên trong HA | **Frigate AI Task - Thông báo đa kênh** |
| Bản đầy đủ cũ | `frigate-ai-multichannel.yaml.bak-full-20260926` (không được HA nạp) |
| Script sinh file | `build-lean-frigate-blueprint.py` |
| Kiểm thử | `hass --script check_config` với automation mẫu đầy đủ → **0 error** |

## 1. Toàn bộ cấu hình cần điền (26 mục)

**🎥 Frigate**
| Mục | Mặc định | Ghi chú |
|---|---|---|
| Camera Frigate | — | chọn 1 hoặc nhiều camera |
| 🏷️ Nhãn cần thông báo | person, dog, cat | thêm `face`, `car`, `bicycle`, `package` nếu muốn |
| 📱 Điện thoại nhận thông báo | — | thiết bị mobile_app |
| 🧭 Chỉ báo khi vào vùng | trống = mọi vùng | tên zone Frigate, cách nhau dấu phẩy |
| 🚫 Bỏ qua vùng | trống | sự kiện trong vùng này không thông báo |
| 🎯 Điểm tin cậy tối thiểu | 0.6 | |
| 🧹 Bỏ qua cảnh báo giả | bật | |
| ⏳ Cooldown | 0 | khoảng cách tối thiểu giữa 2 thông báo |
| ⏳ Helper cooldown | trống | helper Date and time, chỉ cần khi đặt Cooldown |

**🤖 AI Task**
| Mục | Mặc định | Ghi chú |
|---|---|---|
| ✅ Bật phân tích AI | bật | tắt = chỉ cảnh báo Frigate |
| 🤖 AI Task Entity | `ai_task.google_ai_task` | |
| 📝 Kiểu đầu ra của AI | Structured Output | đổi sang JSON nếu dùng model khác |
| 🖼️🎞️ Đưa gì cho AI? | Ảnh snapshot | Ảnh · Video ngắn · Cả hai |
| ⌛ Số ảnh chụp cho AI | 1 | 1–5 khung, giúp AI suy luận đi vào/đi ra |
| 🚶 Chỉ gửi khi AI thấy có người | bật | lọc cảnh báo ảo |
| 👤 Cho AI biết người đã nhận diện | bật | đưa sub_label vào câu lệnh cho AI |
| 🧑 Map tên khuôn mặt (JSON) | trống | `{"thuy": "Ngọc Thụy", "me": "Mẹ"}` |

**📣 Kênh gửi thông báo**
| Mục | Mặc định |
|---|---|
| ✅ Zalo · 📞 Số điện thoại Zalo Bot · 💬 Zalo Thread ID | tắt / — / — |
| ✅ Telegram · ✈️ Telegram notify entity | tắt / [] |
| 🎞️ Gửi kèm video clip (Zalo, Telegram, link cho Hermes/Harness) | tắt |
| ✅ Hermes bot (`@hermesnt10_bot`) | tắt |
| ✅ Harness bot (`@harness_chat_bot`) | tắt |

**⚙️ Nâng cao (tùy chọn)**
| Mục | Ghi chú |
|---|---|
| 🗺️ Map tên camera (JSON) | **cần cho máy anh**: `{"camera.cam_san":"cam_fc367872"}` |
| 🔒 Helper khoá AI | `input_boolean.frigate` — tránh 2 camera gọi AI cùng lúc |
| 🐞 Ghi log gỡ lỗi | |

## 2. Automation mẫu (dán vào `automations.yaml` rồi Reload automations)

```yaml
- id: 'frigate-ai-multichannel-01'
  alias: Frigate AI Task - Đa kênh
  use_blueprint:
    path: thuyee/frigate-ai-multichannel.yaml
    input:
      camera: [camera.cam_san]
      camera_name_map: '{"camera.cam_san":"cam_fc367872"}'
      labels: [person, dog, cat, face]
      notify_device: [ca816efd5eb112b08e47b18b4a392e50]
      min_score: 0.6
      require_not_false_positive: true
      ai_enabled: true
      ai_task_entity: ai_task.google_ai_task
      ai_task_output: is_structured
      ai_task_media: both        # ảnh + video ngắn
      num_snapshots: 3
      ai_require_person: true
      ai_identity_aware: true
      face_name_map: '{"thuy":"Ngọc Thụy","me":"Mẹ"}'
      zalo_enable: true
      zalo_account: '+84868837123'
      zalo_thread_id: '1058896116335801995'
      hermes_enable: true
      harness_enable: true
      helper: input_boolean.frigate
```

## 4. 🐞 Bug của blueprint gốc đã sửa (quan trọng!)

Bản gốc `frigate-ai-notification zalo.yaml` (sam2kb) có **lỗi kiểu dữ liệu** khiến **cảnh báo ban đầu
(initial notification) KHÔNG BAO GIỜ được gửi** khi chưa cấu hình helper:

```jinja
cooldown_active:  '{% if ent|length == 0 %}false {% else %}...{% endif %}'   → trả về CHUỖI " false " (không phải bool)
camera_silenced:  '{% if not silence_table_enabled %}false{% else %}...{% endif %}'  → CHUỖI "false"
```

Chuỗi `"false"` là **truthy** trong Jinja ⇒ điều kiện `not cooldown_active` / `not camera_silenced` luôn **SAI**
⇒ automation dừng ngay trước bước gửi thông báo ⇒ không chụp ảnh, không gọi AI, không gửi Zalo/Hermes/Harness.
(Chỉ khi tạo đủ helper cooldown + silence table thì 2 biến mới trả về bool thật.)

Đã sửa trong bản gọn (3 chỗ):

```jinja
cooldown_active:      {{ false if ent | length == 0 else ((now_ts - last_notification_ts|int(0)) < (cooldown_secs|int(0))) }}
camera_silenced:      {{ false if not silence_table_enabled else (now_ts < until) }}
camera_silenced_now:  {{ false if not silence_table_enabled else ((as_timestamp(now())|int(0)) < until) }}
```


## 4b. 🔧 3 lỗi thực tế đã sửa sau khi test với điện thoại/Zalo (26/09/2026)

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| Thông báo điện thoại **không có ảnh**, lại còn bị **tách thành nhiều thông báo** | (a) HA 2026.9 **không còn entity `homeassistant`** ⇒ `state_attr('homeassistant','external_url')` rỗng ⇒ mọi URL ảnh/clip thành **relative** (`/api/...`) ⇒ app không tải được ảnh. (b) Khối "end" gửi thêm 1 thông báo với `tag` khác nên bị tách riêng | (a) Thêm input **🏠 Địa chỉ Home Assistant** (`ha_url`) để tạo URL tuyệt đối. (b) **Bỏ hẳn thông báo "end"**; còn 2 thông báo cùng `tag` (cảnh báo đầu → thay bằng thông báo AI) ⇒ điện thoại chỉ hiện **1 thông báo có cả ảnh + text** |
| **Zalo không nhận được** | `image_path` gửi đi là đường dẫn tương đối `/media/...` (do lỗi URL ở trên) → zalobot không lấy được ảnh, và gửi file local qua zalobot rất chậm/dễ timeout 60s | Ảnh được lưu vào `/config/www/frigate_ai/` và gửi Zalo bằng **URL công khai** `https://ngocthuyhome.com/local/frigate_ai/...jpg` — test gửi **tức thì (0.0s)** |
| Bấm **Xem Clip / Mở Frigate / Xem Live** không mở được | Link bị relative; `Open Frigate` trỏ vào `/api/frigate/review` (HA không có endpoint này); `View Live` dùng `entityId:camera.…` (app Android không mở được) | Thêm input **🌐 Địa chỉ Frigate** (`frigate_url`). Giờ: Xem Clip → `https://ngocthuyhome.com/api/frigate/notifications/<id>/<cam>/clip.mp4` (đã test **200 OK** với sự kiện thật); Mở Frigate → `https://frigate.ngocthuyhome.com/review?camera=<cam>&id=<id>`; Xem Live → `https://frigate.ngocthuyhome.com/` |

> Frigate của anh có bảo vệ bằng nginx (401) nên lần đầu mở link Frigate, trình duyệt sẽ hỏi tài khoản/mật khẩu Frigate — đó là bình thường.
> Riêng link Clip đi qua HA (`ngocthuyhome.com/api/frigate/...`) nên **không cần đăng nhập**.

## 4c. 📸 Ảnh/video lấy từ đâu? Có cần tự xoá không?

| Loại | Nguồn | Nơi lưu |
|---|---|---|
| Ảnh gửi kèm thông báo AI | **Home Assistant tự chụp** (`camera.snapshot` từ camera `camera.cam_san` của Frigate) | 2 bản: `/config/www/frigate_ai/frigate_ai_<cam>_1.jpg` (để lấy URL công khai + gửi Telegram/Hermes/Harness) và `/media/snapshots/frigate_ai_<cam>_1.jpg` (để đưa cho AI Task) |
| Ảnh ở cảnh báo đầu tiên | **Frigate** (thumbnail của sự kiện) | Frigate tự quản lý |
| Video ngắn cho AI (nếu bật `ai_task_media: clip/both`) | **HA tự ghi** (`camera.record`, 8s + lookback 3s) | `/media/snapshots/frigate_ai_<cam>_rec.mp4` |
| Clip trong thông báo | **Frigate** (bản ghi của Frigate) | Frigate tự quản lý + tự xoá theo `record.retain` |

**Không cần tự xoá**: tên file đặt theo **từng camera** (`frigate_ai_cam_fc367872_1.jpg`) nên mỗi sự kiện **ghi đè** file cũ — tối đa 2 file/camera (1 ở www + 1 ở media).


## 4d. 🎥 Bản cập nhật 2 (26/09/2026): AI dùng thẳng MEDIA CỦA FRIGATE

**Bỏ hoàn toàn việc HA tự chụp/ghi hình.** Nguyên nhân trước đây phải tự chụp: **Frigate của anh đang tắt snapshots**
(`snapshots.enabled: false` ⇒ mọi event đều `has_snapshot: false`, endpoint `snapshot.jpg` trả 404). Đã bật lại trong
`/home/ngocthuyserver/docker/frigate/config/config.yaml`:

```yaml
snapshots:
  enabled: true
  timestamp: false
  bounding_box: true
  crop: false
  quality: 70
  retain:
    default: 10
```
(đã backup config cũ, đã restart Frigate, kiểm tra `/api/config` báo `enabled: true`)

**Luồng mới:**
1. Frigate phát hiện → gửi thông báo nhanh (ảnh = **thumbnail của Frigate**) + link.
2. Khi sự kiện kết thúc (clip sẵn sàng), HA gọi `shell_command.frigate_download` để **tải snapshot + clip của chính Frigate** về `/media/snapshots/` rồi đưa cho AI Task:
   - `ai_task_media: snapshot` → tải `/api/frigate/notifications/<id>/snapshot.jpg`
   - `ai_task_media: clip` / `both` → tải thêm clip `.mp4` của Frigate (Gemini phân tích được video)
3. Ảnh trong thông báo AI = **snapshot của Frigate** (`https://ngocthuyhome.com/api/frigate/notifications/<id>/snapshot.jpg`)
   → Zalo (URL công khai), Telegram/Hermes/Harness (file đã tải), điện thoại (URL).

> Vì sao phải "tải về"? `ai_task.generate_data` chỉ nhận media **cục bộ** (media-source), không nhận URL — nên HA phải tải media của Frigate về rồi mới đưa cho AI. Ảnh/video vẫn là **của Frigate**, HA không tự chụp nữa.

**Shell command mới** (đã thêm & reload, không cần restart HA): `/config/packages/frigate_ai_media.yaml`
```yaml
shell_command:
  frigate_download: >-
    curl -sS -m 180 -L -o '{{ dest }}' '{{ url }}'
```

## 4e. 👤 Prompt nhận diện danh tính (đã siết chặt)

Trước đây AI có thể tả "một nam thanh niên…" dù Double Take đã xác định là Ngọc Thụy. Prompt gửi cho AI giờ là:

> *DANH TÍNH ĐÃ ĐƯỢC XÁC NHẬN: Hệ thống nhận diện khuôn mặt (Frigate + Double Take) đã xác định người trong ảnh/video này là **Ngọc Thụy**.
> BẮT BUỘC: (1) Luôn gọi người này bằng tên **Ngọc Thụy**, ví dụ: "**Ngọc Thụy** đang đi vào sân, tay cầm túi xách".
> (2) TUYỆT ĐỐI KHÔNG mô tả chung chung kiểu "một người đàn ông", "một nam thanh niên", "một phụ nữ" thay cho tên.
> (3) Mọi hành động, trang phục, đồ vật phải gắn liền với **Ngọc Thụy**.*

## 4f. 🔗 Nút trên thông báo (đã sửa)

| Nút | URL |
|---|---|
| Xem Clip | `https://ngocthuyhome.com/api/frigate/notifications/<id>/<cam>/clip.mp4` (**không cần đăng nhập**, đã test 200 với sự kiện cũ lẫn mới) |
| Mở Frigate | `https://frigate.ngocthuyhome.com/review?camera=<cam>&id=<id>` |
| Xem Live | `https://frigate.ngocthuyhome.com/` |
| Bấm vào thân thông báo | Mở trang Frigate của sự kiện |

> API của Frigate (`frigate.ngocthuyhome.com/api/...`) yêu cầu đăng nhập (401) nên **clip đi qua HA** để bấm là chạy ngay;
> còn trang UI Frigate thì public (200) — anh cứ bấm thử, lần đầu trình duyệt có thể hỏi mật khẩu Frigate.


## 4g. 🐞 3 lỗi cuối đã sửa (26/09/2026) — Zalo & ảnh Hermes/Harness

| Triệu chứng | Nguyên nhân | Sửa |
|---|---|---|
| **Zalo không nhận được gì** (kể cả text) | HA tự **ép kiểu** giá trị template: `'+84868837123'` → số `84868837123` (**mất dấu +**) và thread_id → số. Component zalo_bot tra tài khoản theo `+84868837123` nên **không tìm thấy** → service lỗi, bị `continue_on_error` nuốt mất. (Bản gốc sam2kb cũng bị y hệt.) | Dùng `!input` trực tiếp (không qua template) cho `thread_id`/`account_selection`, `type: '1'` cố định ⇒ giá trị giữ nguyên **chuỗi** |
| **Hermes/Harness chỉ có text, không có ảnh** | (a) Biến `has_image` bị khai báo **trước** `snapshot_public_url` ⇒ luôn `False` ⇒ rơi vào nhánh `*_notify` (text). (b) Riêng Harness còn dùng biến cũ `image_file_local` (đã bỏ) ⇒ `photo` rỗng | Đảo thứ tự khai báo `has_image`; sửa `photo` của Harness sang `snapshot_media_file` |
| Điện thoại | (không lỗi) | — |

**Bằng chứng sau khi sửa** (sự kiện thật, `sub_label: Mẹ`):
```
zalo_bot.send_image: {"thread_id": "1058896116335801995", "account_selection": "+84868837123", "type": "1",
                      "image_path": "https://ngocthuyhome.com/api/frigate/notifications/<id>/snapshot.jpg"}
shell_command.hermes_notify_photo: {"photo": "/media/snapshots/frigate_ai_cam_fc367872_1.jpg"}
shell_command.harness_notify_photo: {"photo": "/media/snapshots/frigate_ai_cam_fc367872_1.jpg"}
```
Và tin nhắn **đã có trong nhóm Zalo**:
```
Mẹ detected by Cam Fc367872.

Mẹ đang đi xe máy màu tối, đội mũ bảo hiểm màu đỏ, mặc áo xanh, di chuyển từ phía cổng vào sân nhà.

👤 Đã nhận diện: Mẹ
```
→ AI đã **gắn hành động với đúng danh tính** (Mẹ), không còn kiểu "một nam thanh niên…".


## 4h. ⚡ Bản cập nhật 3: AI chạy NGAY (không chờ sự kiện kết thúc) + các lỗi đã sửa

| Vấn đề | Sửa |
|---|---|
| Thông báo gửi đi nhưng **mất phần AI phân tích** | Google AI Task thỉnh thoảng trả lỗi *"Error with structured response"* → đã thêm **tự động fallback sang chế độ JSON** khi structured lỗi |
| Phải **chờ sự kiện kết thúc** mới có AI (có sự kiện kéo dài 5 phút) | **Bỏ hẳn vòng chờ**; AI chạy ngay sau cảnh báo đầu (delay 15s cho Frigate ghi snapshot). Run thực tế: **29 giây** |
| Ở chế độ `clip`, nếu payload chưa có `has_clip=true` thì **không tải gì** → AI không có media | **Luôn tải ảnh snapshot** của Frigate làm nền; tải thêm clip khi có |
| 5 sự kiện liên tiếp = 5 lần gửi thông báo | Thêm **cooldown 2 phút** (helper `input_datetime.frigate_notification_cooldown`), chặn cả thông báo AI |
| `sub_label` dạng `["Mẹ", 0.94]` hiển thị thành *"Mẹ, 0.944574691409858"* | Chỉ lấy **phần tử đầu** (tên) |

**Test bằng dữ liệu 100% thật** (event `1790396303.343313-rj7i0n` lúc 11:18:23, Frigate trả `sub_label: 'Mẹ'`):
```
⏱ 29 giây | AI: "Mẹ mặc áo phông đen và quần đùi hồng, đang cầm cây lau nhà di chuyển trong sân."
Kênh đã gửi: điện thoại + Zalo (ảnh) + Hermes (ảnh) + Harness (ảnh)
```
> ⚠️ Lưu ý về test: các tin "Mẹ" lúc **11:35-11:36** là do **em tiêm `sub_label` giả** vào payload MQTT để thử nhánh nhận diện —
> không phải Frigate nhận diện. Dữ liệu thật: trong 12 sự kiện gần nhất Frigate chỉ nhận diện "Mẹ" ở **2 sự kiện**
> (11:18:23 và 10:44:46), các sự kiện khác `sub_label = None`.

## 5. ✅ Kết quả test thật trên camera Sân (26/09/2026)

Automation đã tạo trong `automations.yaml`:
`automation.frigate_ai_task_camera_san_test` — alias **“Frigate AI Task - Camera Sân (test)”**
(camera `camera.cam_san` ↔ Frigate `cam_fc367872`, `ai_task.google_ai_task`, Zalo + Hermes + Harness + điện thoại, `ai_require_person: false` để test).

Bắn 3 sự kiện Frigate giả qua MQTT `frigate/events` (new + end):

| Bước | Kết quả |
|---|---|
| Cảnh báo ban đầu | ✅ gửi điện thoại + logbook `INFO (initial sent)` |
| Chụp ảnh | ✅ `/media/snapshots/frigate_ai_cam_fc367872_<id>_1.jpg` (189 KB) |
| AI Task (Google) | ✅ trả về: *“Không có người xuất hiện trong khung hình. Chỉ có xe máy và các vật dụng tĩnh.”* (`humans_detected: 0`) |
| Zalo | ✅ `zalo_bot.send_image` ảnh + caption (không lỗi trong log) |
| Hermes | ✅ `shell_command.hermes_notify_photo` |
| Harness | ✅ `shell_command.harness_notify_photo` |
| Điện thoại | ✅ `notify.mobile_app_asus_ai2201_a` |
| Telegram | ⏭️ bỏ qua (đang tắt trong test) |
| Nhận diện khuôn mặt (test `sub_label: "Ngọc Thụy"`) | ✅ `👤 Đã nhận diện: Ngọc Thụy` + câu lệnh gửi AI có kèm tên |

Caption thực tế đã gửi:

```
Ngọc Thụy detected by Cam Fc367872.

Không phát hiện người nào trong khung hình. Ảnh chỉ hiển thị sân nhà với một chiếc xe máy và các chậu cây cảnh.

👤 Đã nhận diện: Ngọc Thụy

🎥 /api/frigate/notifications/<event_id>/cam_fc367872/clip.mp4
```

> Lưu ý: `input_boolean.frigate` (helper khoá AI mà automation AI Smart Camera dùng) đang bị kẹt ở trạng thái **on**.
> Nếu sau này anh điền `helper: input_boolean.frigate` cho blueprint này thì AI sẽ chờ tối đa 3 phút rồi dừng —
> nên tắt helper đó trước.

## 6. Đã bỏ khỏi bản đầy đủ (không dùng tới)

| Nhóm bị bỏ | Vì sao |
|---|---|
| LLMVision (provider, model, prompt, max_tokens, temperature, max_frames, target_width, retry, expose_images, generate_title…) | anh dùng AI Task; máy chưa có provider LLMVision |
| Conversation Agent | không cần |
| Discord, TTS ra loa | không dùng |
| Nút Xác nhận / Bỏ qua trên thông báo | không cần |
| Custom Actions + Global Conditions | không cần |
| Kênh cảnh báo nhanh (text trước khi AI xong) | điện thoại đã nhận cảnh báo nhanh sẵn |
| Snapshot nâng cao (thư mục, camera ghi đè, delay, lookback/duration) | đã đặt mặc định hợp lý |
| suppress_known_faces | **ngược** với yêu cầu nhận diện người quen |
| require_clip, base/local/frigate URL, notification_timeout, ios_live_view, append/expand tên camera, dashboard, log helper, silence table, zone_match_type/zone_logic | đã gán hằng số mặc định bên trong blueprint |

> Trong file vẫn giữ nguyên toàn bộ logic Frigate gốc (lọc zone, cooldown, tự tắt 25 giây theo camera,
> cập nhật khi clip sẵn sàng, thông báo kết thúc, nút Silence trên điện thoại) — chỉ **không hiện ra UI** nữa.

## 4. Muốn thêm lại mục nào?

Nói em biết, em bật lại trong 1 phút: ví dụ *"thêm lại chọn LLMVision"*, *"thêm Discord/TTS"*,
*"cho sửa prompt của AI"*, *"thêm nút Xác nhận/Bỏ qua"*, *"thêm chọn số ảnh/delay"*, *"thêm silence table riêng từng camera"*.
