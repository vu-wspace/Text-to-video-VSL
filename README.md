# Text-to-Video VSL — Chuyển văn bản tiếng Việt sang video Ngôn ngữ Ký hiệu

Hệ thống chuyển một câu tiếng Việt thành video Ngôn ngữ Ký hiệu Việt Nam (VSL — Vietnamese Sign Language), sử dụng phương pháp **dictionary-based concatenative synthesis**: tách câu thành chuỗi gloss (đơn vị ký hiệu), tra video ký hiệu tương ứng trong từ điển, rồi ghép nối lại thành một video hoàn chỉnh.

## Demo / Nguồn chạy

* Notebook gốc (chạy trên Kaggle, có kết quả thực tế): [kaggle.com/code/vuwspace/text-to-video-vsl](https://www.kaggle.com/code/vuwspace/text-to-video-vsl)
* Video demo 1: [youtu.be/6upUo0nhmNs](https://youtu.be/6upUo0nhmNs) — video demo pixel gốc (output từ pipeline Video Synthesis, chưa qua xử lý skeleton).
* Video demo 2: [youtu.be/uJvQVfxO9_Q](https://youtu.be/uJvQVfxO9_Q) — video demo overlay skeleton, vẽ pose/face/hand landmarks lên đúng video demo 1 ở trên (minh hoạ chất lượng trích xuất keypoints bằng MediaPipe Holistic Landmarker).

## Kiến trúc pipeline

```
Văn bản tiếng Việt
      │
      ▼
[1] Gloss Sequence   — Forward Maximum Matching (FMM) trên từ điển gloss
      │                 + xử lý từ ngoài từ điển (OOV)
      ▼
[2] ID Sequence      — chọn video cụ thể cho mỗi gloss theo vùng miền (Bắc/Trung/Nam)
      ▼
[3] Video Synthesis  — chuẩn hoá kích thước/fps từng clip, ghép nối bằng MoviePy
      ▼
Video VSL hoàn chỉnh (.mp4)
```

### 1. Gloss Sequence — Text → Gloss

- **Chuẩn hoá dữ liệu**: NFC normalization, lowercase, dùng `underthesea` để tokenize và đồng bộ chính tả cũ/mới giữa gloss trong từ điển và văn bản đầu vào.
- **Thuật toán khớp**: Forward Maximum Matching (FMM) — quét câu từ trái sang phải, tại mỗi vị trí thử khớp cụm từ dài nhất có thể với gloss trong từ điển trước khi lùi dần về cụm ngắn hơn.
- **Xử lý từ ngoài từ điển (OOV)**:
  - Loại bỏ từ chức năng (đại từ, giới từ, trợ từ...) không có trong từ điển gloss.
  - Với từ có nghĩa không khớp được gloss nào: fallback sang **đánh vần từng chữ cái** (fingerspelling), dùng từ điển video bảng chữ cái (loại bỏ dấu thanh trước khi tra).
  - Từ vẫn không xử lý được → đưa vào danh sách `unresolved` để theo dõi riêng.

### 2. ID Sequence — Gloss → Video ID

- Mỗi gloss có thể ứng với nhiều video (do nhiều người ký hiệu / nhiều vùng miền quay).
- Hàm `pick_region()` ưu tiên chọn video theo vùng miền được yêu cầu (hậu tố B/N/T); nếu không có, chọn theo thứ tự ưu tiên mặc định.

### 3. Video Synthesis — Ghép video

- Dùng **MoviePy**: resize từng clip theo tỉ lệ khung hình mục tiêu (giữ tỷ lệ gốc, letterbox nền đen), đồng bộ FPS (25fps), rồi `concatenate_videoclips`.
- Có xử lý lỗi khi file video bị thiếu hoặc không load được (bỏ qua, log lại, không crash toàn pipeline).

### 4. Gloss–Skeleton Dictionary (hướng mở rộng đã triển khai một phần)

- Trích xuất **pose / face / hand landmarks** cho từng video bằng **MediaPipe Holistic Landmarker (Tasks API)**, lưu riêng dưới dạng JSON theo từng khung hình.
- Xử lý theo lô (batch) có khả năng resume (bỏ qua video đã trích xuất), tự động nén kết quả từng lô thành `.zip` để tải về theo phần — phù hợp với giới hạn thời gian/dung lượng của môi trường Kaggle.
- Demo overlay: vẽ skeleton (pose/face/tay trái/tay phải) lên video gốc để kiểm tra chất lượng landmark.
- Hướng đi tiếp theo (đã phác thảo, chưa hoàn thiện): ghép video ở dạng skeleton-only theo câu, phục vụ nghiên cứu dựa trên toạ độ khớp thay vì pixel thô — giúp giảm phụ thuộc vào hình ảnh người ký hiệu cụ thể.

## Dữ liệu

**QiPed VSL Dataset**
- 3,304 gloss (nhãn ý nghĩa)
- ~4,360 video ký hiệu (định dạng `.webm`)
- Cột dữ liệu chính: `ID_video`, `Meaning` (gloss), `Pos_tag`
- EDA sơ bộ: phân bố từ loại (POS tag), kiểm tra trùng lặp `ID_video` và `Meaning`, xác định các gloss có nhiều biến thể video (nhiều vùng miền/người ký hiệu).

## Kết quả

- **Round-trip test trên từ điển**: 100% gloss được khôi phục chính xác, 0% OOV — tức là mọi gloss có sẵn trong từ điển đều được thuật toán FMM nhận diện đúng khi đưa ngược lại làm input.
- Đã sinh thành công video demo end-to-end cho câu tiếng Việt tuỳ ý (ví dụ: "xin chào, tôi tên là Vũ") với lựa chọn vùng miền.
- Đã trích xuất keypoints cho tập video mẫu và tạo được demo overlay skeleton.

*Lưu ý:* con số 100%/0% là kết quả round-trip trên chính từ điển (kiểm tra tính nhất quán của thuật toán khớp), **chưa phải** đánh giá trên câu tiếng Việt tự nhiên ngoài từ điển — đây là hướng cần bổ sung (xem phần Hạn chế).

## Hạn chế

- **Chưa xử lý ngữ pháp VSL**: hệ thống giữ nguyên thứ tự từ của câu tiếng Việt gốc, trong khi VSL có cấu trúc ngữ pháp riêng (thứ tự từ, thay thế từ) khác với tiếng Việt viết.
- **Không nhất quán người ký hiệu**: do nguồn video được quay bởi nhiều người/nhiều vùng miền khác nhau, video ghép nối có thể đổi người ký hiệu giữa các gloss liên tiếp, ảnh hưởng tính tự nhiên khi xem.
- Chưa có bộ đánh giá định lượng trên câu ngoài từ điển (chỉ mới round-trip test trên chính từ điển).
- Phần sinh video từ skeleton (thay vì video pixel gốc) mới dừng ở mức prototype, chưa hoàn thiện.

## Tài liệu tham khảo liên quan

Trong quá trình tìm hiểu, phát hiện một số công bố khoa học của tác giả Nguyễn Thị Bích Điệp (luận án tiến sĩ, Học viện Khoa học và Công nghệ) nghiên cứu đúng chủ đề dịch tự động ngôn ngữ ký hiệu Việt Nam — đáng tham khảo để đối chiếu hạn chế đã nêu và định hướng phát triển tiếp theo:

- **Nguyen, T.B.D., Phung, T.N.** (2017). *Some issues on syntax transformation in Vietnamese sign language translation*. IJCSNS, Vol.17 No.6. — Chỉ ra cấu trúc ngữ pháp VSL là **Chủ ngữ → Tân ngữ → Vị ngữ** (khác thứ tự Chủ ngữ → Vị ngữ → Tân ngữ của tiếng Việt nói), cùng quy tắc riêng cho câu hỏi, câu phủ định, và thứ tự danh từ/số từ. Đây đúng là hướng giải quyết cho hạn chế "chưa xử lý ngữ pháp VSL" đã nêu ở trên.

- **Nguyen, T.B.D., Phung, T.N., Vu, T.T.** (2017). *A rule-based method for text shortening in Vietnamese sign language translation*. Springer AISC, Vol. 672. — Đề xuất quy tắc rút gọn câu tiếng Việt trước khi chuyển sang VSL bằng cách loại bỏ giới từ, liên từ, trợ từ tình thái và thay thế từ đồng nghĩa — cùng hướng tiếp cận với bước xử lý OOV (stopword/fingerspelling) đã xây dựng trong project này, nhưng ở mức độ quy tắc ngữ pháp toàn diện hơn.

- **Nguyen, T.B.D., Ho, T.T.** *Data Augmentation Techniques in automatic translation of Vietnamese Sign Language for the deaf*. — Dùng WordNet (quan hệ hypernym/hyponym) để sinh thêm dữ liệu câu Việt-VSL từ tập dữ liệu gốc, phục vụ huấn luyện các model dịch thống kê (Seq2Seq, Transformer) — hướng tham khảo trực tiếp cho việc "xây dựng bộ test câu tiếng Việt tự nhiên" đã nêu trong phần Hướng phát triển.

- **Nguyen, T.B.D. et al.** *Special characters of Vietnamese sign language recognition System based on Virtual Reality Glove*. — Nghiên cứu nhận diện số và ký tự đặc biệt VSL bằng găng tay cảm biến (flex sensor + accelerometer), khác hướng tiếp cận thị giác máy tính (MediaPipe) của project này, nhưng cùng vấn đề: nhận diện chữ cái/ký tự đặc biệt tiếng Việt trong VSL.

## Hướng phát triển

- Hoàn thiện pipeline sinh video từ skeleton (keypoints) để giảm phụ thuộc vào video pixel gốc và đồng nhất "hình ảnh người ký hiệu".
- Nghiên cứu mô hình xử lý ngữ pháp VSL (word reordering/substitution, xem tài liệu tham khảo ở trên) trước bước tra từ điển.
- Xây dựng bộ test câu tiếng Việt tự nhiên (ngoài từ điển) kèm đánh giá tỉ lệ OOV thực tế và đánh giá định tính từ người dùng VSL.

## Công nghệ sử dụng

`Python` · `pandas` · `underthesea` (tokenization tiếng Việt) · `MoviePy` (xử lý/ghép video) · `MediaPipe Holistic Landmarker` (trích xuất pose/face/hand keypoints) · `OpenCV`

## Cấu trúc mã nguồn (theo notebook)

| Phần | Chức năng |
|---|---|
| `normalize_text`, `canonicalize_gloss` | Chuẩn hoá và đồng bộ chính tả gloss/văn bản |
| `gloss_sequence` | Thuật toán FMM + xử lý OOV (stopword, fingerspelling) |
| `pick_region`, `id_sequence` | Ánh xạ gloss → video ID theo vùng miền |
| `normalize_clip`, `video_demo` | Chuẩn hoá và ghép video bằng MoviePy |
| `extract_keypoints`, `skeleton_keypoints_folder`, `run_all_batches` | Trích xuất và lưu trữ keypoints theo lô |
| `draw_keypoints_on_frame`, `skeleton_video_demo_1` | Demo trực quan hoá skeleton overlay |
