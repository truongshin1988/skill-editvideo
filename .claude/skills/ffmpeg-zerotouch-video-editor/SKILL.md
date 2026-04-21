---
name: ffmpeg-zerotouch-video-editor
description: Pipeline tự động render video bằng FFmpeg zero-touch — tự execute từ Edit Plan đến verify output, chỉ xin phép user ở 2 điểm (chọn mode + duyệt plan), không hỏi permission từng bước. Quy định chế độ CUT mode (40% Camera / 40% B-roll / 20% Remotion) và NO-CUT mode (60% Remotion overlay), Edit Plan bắt buộc 9 cột, render single-pass FFmpeg duy nhất (không batch trung gian), typography Inter Bold tiếng Việt IN HOA vàng #ffd000, subtitle pop-up hoặc karaoke .ass, zoom controller 1.0×–1.3× xen kẽ cho Camera, Remotion overlay trắng qua PTS shift, AI B-roll từ Gemini Chrome (không API) với Ken Burns bắt buộc, và 5-point verify sau render. Luôn dùng skill này khi user muốn render/edit video tự động bằng FFmpeg, tạo Edit Plan, chọn CUT/NO-CUT, thêm subtitle karaoke/.ass, chèn AI B-roll, overlay Remotion, drawtext tiếng Việt, hoặc bất kỳ tác vụ video automation zero-touch nào — kể cả khi không nhắc cụ thể "FFmpeg".
---
---
name: ffmpeg-zerotouch-video-editor
description: Pipeline tự động render video bằng FFmpeg zero-touch — tự execute từ Edit Plan đến verify output, chỉ xin phép user ở 2 điểm (chọn mode + duyệt plan), không hỏi permission từng bước. Quy định chế độ CUT mode (40% Camera / 40% B-roll / 20% Remotion) và NO-CUT mode (60% Remotion overlay), Edit Plan bắt buộc 9 cột, render single-pass FFmpeg duy nhất (không batch trung gian), typography Inter Bold tiếng Việt IN HOA vàng #ffd000, subtitle pop-up hoặc karaoke .ass, zoom controller 1.0×–1.3× xen kẽ cho Camera, Remotion overlay trắng qua PTS shift, AI B-roll từ Gemini Chrome (không API) với Ken Burns bắt buộc, và 5-point verify sau render. Luôn dùng skill này khi user muốn render/edit video tự động bằng FFmpeg, tạo Edit Plan, chọn CUT/NO-CUT, thêm subtitle karaoke/.ass, chèn AI B-roll, overlay Remotion, drawtext tiếng Việt, hoặc bất kỳ tác vụ video automation zero-touch nào — kể cả khi không nhắc cụ thể "FFmpeg".
---

# AI Video Editor Zero-Touch (FFmpeg)

Pipeline tự động render video bằng FFmpeg cho workflow SOFIN. Input: source clip + clean WAV + Whisper script. Output: `<input>_v2.mp4` đã verify, qua một lệnh FFmpeg single-pass duy nhất.

## Nguyên tắc tối thượng

1. **Zero-touch** — user chỉ duyệt 2 lần: chọn mode + duyệt Edit Plan. Ngoài ra execute hết.
2. **Single-pass** — một filter_complex duy nhất cho toàn bộ overlay. Không batch đa bước, không file trung gian.
3. **Không phá source** — output ra file mới, tuyệt đối không ghi đè input.
4. **Verify trước khi báo xong** — chưa pass 5-point check = chưa xong.

---

## 1. Mở đầu — BẮT BUỘC hỏi mode trước mọi việc

Ngay khi skill trigger, tin nhắn đầu tiên:

> "Bạn muốn bắt đầu với **CUT mode** (40% Camera / 40% B-roll / 20% Remotion) hay **NO-CUT mode** (60% Remotion overlay)?"

Nếu user yêu cầu tỉ lệ khác (bỏ B-roll, bỏ text overlay, chỉ Remotion…), ghi nhận và điều chỉnh Edit Plan tương ứng. Không tự chọn, không giả định.

## 2. Báo cáo tiến độ

Mọi step > 5 giây phải hiện % progress. Format: `[██████░░░░] 60% — đang render overlay Remotion`. Không im lặng.

---

## 3. Edit Plan — deliverable bắt buộc trước khi render

Sau khi chốt mode, tạo Edit Plan đúng **9 cột**. Render không bắt đầu cho đến khi user duyệt.

| # | Timecode | Dur | Cắt/Giữ | Zoom | Camera | Remotion | AI B-roll | Text Overlay | Script |
|---|----------|-----|---------|------|--------|----------|-----------|--------------|--------|
| 1 | 00:00–00:08 | 8s | Giữ | 1.0→1.2 | ✓ | | | "XIN CHÀO" @00:02 | … |
| 2 | 00:08–00:14 | 6s | Cắt | – | | ✓ | | – | … |
| 3 | 00:14–00:25 | 11s | Giữ | 1.0→1.3 | | | ✓ (counter) | "3 BƯỚC" @00:16 | … |

Sau bảng, hiển thị:
- Tổng duration
- Số scene: Camera / B-roll / Remotion
- Tổng text overlay
- Prompt: **"Duyệt để render? [Y/N]"**

Nếu user sửa plan → cập nhật bảng, hỏi lại Y/N. Không render nửa chừng.

---

## 4. FFmpeg render rules (cốt lõi)

### 4.1 Single-pass

Tất cả overlay, subtitle, zoom, audio, crossfade nằm trong **một `filter_complex` duy nhất**.

Lý do: tránh re-encode nhiều lần làm mất chất lượng, tránh file trung gian gây lệch sync, dễ verify duration/FPS.

**Không viết** batch multi-step kiểu `render_step1.mp4 → render_step2.mp4 → final.mp4`.

### 4.2 Đồng bộ FPS & duration

- Output FPS = source FPS. Dùng `-r <source_fps>` (lấy từ `ffprobe`).
- Output duration = source duration chính xác. Dùng `-t <source_duration>`.
- Không frame-blend, không duplicate frame.

### 4.3 Audio

- Input clean WAV trên channel riêng, map `-map 1:a`.
- **Bật denoise** trên clean WAV trước khi mix (`afftdn` hoặc `arnndn`).
- **Không resample** — giữ nguyên sample rate source.
- Crossfade 15ms tại mọi điểm cắt ghép: `acrossfade=d=0.015`.

### 4.4 Output & format

- Dependency cài sẵn: `libfreetype` (drawtext), `libass` (subtitle .ass).
- File output: `<input>_v2.mp4` — **không bao giờ ghi đè input**.
- Batch nhiều clip: dùng `render-v2.sh` chứa lệnh single-pass cho từng clip (không chứa logic multi-step bên trong từng lệnh).

### 4.5 Post-render verify — 5-point check

Sau khi FFmpeg exit code 0, tự động chạy đủ 5 check. Fail bất kỳ point nào = chưa xong, báo rõ cho user:

1. **Duration match** — `ffprobe` output duration = source ±0.04s
2. **A/V sync** — audio start/end offset < 40ms so với video
3. **FPS** — khớp source
4. **Frame count** — `ffprobe -count_frames` khớp ±2 frame
5. **Text xuất hiện** — extract sample frame tại timestamp overlay, xác nhận text visible

---

## 5. Typography

### 5.1 Font — độc quyền

Chỉ dùng duy nhất **Inter Bold** cho mọi overlay, Remotion, subtitle. Không mix font, không dùng regular/medium/black variant.

### 5.2 Tiếng Việt

- **IN HOA** toàn bộ
- Dấu đầy đủ, chính tả tuyệt đối
- Escape cho FFmpeg `drawtext`:
  - `:` → `\:`
  - `%` → `%%`
  - `'` → `\'`
  - Path Windows: double-escape backslash

### 5.3 Text overlay tĩnh

| Thông số | Giá trị |
|---|---|
| Font | Inter Bold |
| Size | 75 px |
| Màu chữ | `#ffd000` (vàng) |
| Viền | Đen, `borderw=2` |
| Nền (box) | **Không** dùng |
| X | `(w-text_w)/2` |
| Y (video ngang 16:9) | `h*0.82` |
| Y (video dọc 9:16) | `h*0.68` |
| On-screen | `startMs` → `startMs + 5000ms` |

**Quy tắc xuất hiện**:
- Chỉ trên scene **Camera** hoặc **AI B-roll**
- **Không** đè lên Remotion (Remotion có text riêng, đè vào sẽ chồng chéo)

### 5.4 Subtitle

Subtitle khác text overlay — subtitle chạy **xuyên suốt** video (Camera + B-roll + Remotion), sync 100% với `startMs` Whisper.

**Pop-up mode (mặc định)**
- Cụm 2–4 từ, đổi theo Whisper word timing
- Inter Bold, vàng `#ffd000`, viền đen 2px
- Y = `h*0.88` (ngang) / `h*0.85` (dọc)

**Karaoke mode (khi user yêu cầu)**
- File `.ass` với tag `\k<cs>` từng từ
- `PrimaryColour` vàng `&H0000D0FF` (BGR)
- `SecondaryColour` trắng `&H00FFFFFF` (từ chưa highlight)
- Chia nhóm thông minh: 2–4 từ/nhóm, ngắt tại dấu câu/mệnh đề
- Không để từ đơn lẻ cuối dòng

---

## 6. Overlay layers

### 6.1 Zoom controller — chỉ Camera

Áp dụng zoom **duy nhất** cho scene Camera. Remotion và B-roll không dùng zoom controller.

```
zoompan=z='<expr>':d=<frames>:s=<WxH>:
  enable='between(t,<start>,<end>)'
```

- Range: 1.0× ↔ 1.3×
- Xen kẽ: clip chẵn zoom in (1.0→1.3), clip lẻ zoom out (1.3→1.0)
- Duration: khớp duration clip Camera

### 6.2 Remotion overlay

| Rule | Giá trị |
|---|---|
| Background | Trắng trơn `#FFFFFF` |
| Max duration | **6 giây** |
| Sync | PTS shift + `eof_action=pass` |
| Enable filter | **KHÔNG** dùng `enable='between(...)'` |
| Color space | Pre-convert `yuvj420p` → `yuv420p` qua RGB intermediate (hoặc pre-encode) |

**Không** dùng `enable` cho Remotion vì Remotion đã có timing riêng; thêm `enable` gây desync và duplicate frame ở biên.

### 6.3 AI B-roll

**Nguồn**: Gemini qua **Chrome browser** (profile `tran.tama@gmail.com`). **Không** dùng Gemini API.

**Rule clip B-roll**:
- Không dùng hình tĩnh — mọi B-roll phải có **Ken Burns** (zoompan)
- Áp dụng **xen kẽ** với Camera — Camera speaker không chạy liên tục > 25 giây
- Template: `counter` · `list` · `comparison` · `process` · `quote` · `keyword` · `story` · `tech` (chọn theo nội dung script)

**Fallback chain** khi Gemini fail:
1. Gemini Chrome (primary)
2. → Remotion overlay
3. → Stock video (last resort)

**Show-moment detection**: Nếu speaker đang demo/chỉ trỏ/khoe vật → **skip** B-roll vào đoạn đó, giữ Camera full.

**Section title**: Video > 10 phút chèn section title animation tại điểm chuyển chủ đề.

---

## 7. Tương tác với user

1. **Không xin phép từng step nhỏ** — chỉ 2 điểm cần user duyệt: (a) chọn mode mở đầu, (b) duyệt Edit Plan.
2. **Luôn hiện % tiến độ** cho step dài.
3. **Trình bày fail verify rõ ràng** — không giấu, không tự fix im lặng.
4. **Không tự đổi rule** (font, màu, vị trí, bitrate). Nếu user muốn khác → hỏi lại và cập nhật vào plan.
5. **Khi script Whisper thiếu timing** → yêu cầu user chạy Whisper lại trước. Không tự bịa `startMs`.

---

## 8. Khi skill này không phù hợp

- Color grading sâu (LUT, node-based) → DaVinci Resolve
- VFX/compositing phức tạp → After Effects
- Edit thủ công, không automation → skill `capcut-ai-film-assembly`
- Chưa có clean WAV hoặc Whisper script → phải làm 2 bước đó trước khi trigger skill này
