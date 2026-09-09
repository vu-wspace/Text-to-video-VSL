# Text-to-Video VSL — Vietnamese Sign Language Synthesis

Hệ thống chuyển văn bản tiếng Việt thành video ngôn ngữ ký hiệu (VSL - Vietnamese Sign Language), sử dụng phương pháp dictionary-based concatenative synthesis kết hợp thuật toán Forward Maximum Matching (FMM) cho việc phân đoạn từ.

> Project cuối kỳ, đồng thời dùng cho hồ sơ ứng tuyển thực tập AI Engineering / Data Science.

## Demo

<!-- Chèn 1 GIF ngắn hoặc link video demo ở đây -- ấn tượng đầu tiên rất quan trọng -->

## Tổng quan pipeline

```
Text đầu vào
    │
    ▼
Gloss Sequence      -- Tokenize + Forward Maximum Matching + xử lý OOV (stopword/fingerspelling)
    │
    ▼
ID Sequence         -- Tra cứu video ID theo gloss, hỗ trợ chọn biến thể vùng miền (Bắc/Nam/Trung)
    │
    ▼
Video Demo          -- Resize, ghép nối các clip thành video hoàn chỉnh
```

Nhánh mở rộng (đang phát triển): trích xuất pose/hand keypoints bằng MediaPipe Holistic, phục vụ cho hướng nghiên cứu nhận diện/sinh chuyển động dựa trên skeleton thay vì pixel thô.

## Tính năng chính

- **Forward Maximum Matching (FMM)**: phân đoạn từ dựa trên chính dictionary VSL, khắc phục lỗi ghép nhầm cụm từ khi dùng tokenizer tổng quát (ví dụ: tránh tách sai "Miến Điện" thành 2 ký hiệu không liên quan).
- **Đồng bộ chính tả cũ/mới**: xử lý các biến thể chính tả tiếng Việt (`hoà`/`hòa`, `khoẻ`/`khỏe`...) để tránh OOV giả.
- **OOV fallback 2 tầng**: loại bỏ từ chức năng (stopword, đã lọc chéo với dictionary để tránh chặn nhầm ký hiệu có thật), và đánh vần từng chữ cái (fingerspelling) cho từ mang nghĩa nhưng thiếu ký hiệu riêng.
- **Hỗ trợ biến thể vùng miền**: chọn ưu tiên video theo giọng Bắc/Nam/Trung khi 1 gloss có nhiều video tương ứng.
- **Skeleton extraction (MediaPipe)**: trích xuất toạ độ pose + tay từng frame, tách biệt dữ liệu thô (toạ độ) khỏi sản phẩm hiển thị (video overlay) để tối ưu tái sử dụng.

## Kết quả đánh giá

Đánh giá bằng round-trip test (khôi phục lại chính các gloss trong dictionary từ dạng văn bản thô):

| Chỉ số | Trước khi sửa | Sau khi sửa |
|---|---|---|
| Tỉ lệ OOV (theo từ) | 0.68% | 0% |
| Gloss khôi phục sai/thiếu | 22/3,304 (0.67%) | 0/3,304 |

## Công nghệ sử dụng

`Python` · `pandas` · `underthesea` (NLP tiếng Việt) · `moviepy` + `opencv-python` (xử lý video) · `mediapipe==0.10.13` (pose/hand estimation)

## Cài đặt

```bash
pip install -r requirements.txt
```

**Lưu ý quan trọng**: `mediapipe` cần ghim đúng phiên bản `0.10.13` (bản mới hơn đã bỏ API `mp.solutions`), kèm `protobuf==4.25.9`. Nếu chạy trên môi trường có sẵn `tensorflow` (như Kaggle), cần gỡ `tensorflow` để tránh xung đột phiên bản `protobuf` (xem chi tiết trong `docs/ARCHITECTURE.md`).

## Sử dụng

```python
from src.video_demo import video_demo

video_demo(
    text="Cho tôi xin địa chỉ của bạn ở tỉnh nào",
    region="N",   # tuỳ chọn: 'B' (Bắc) / 'N' (Nam) / 'T' (Trung)
    output_path="output_demo.mp4",
)
```

## Hạn chế đã biết

- **Chưa xử lý khác biệt ngữ pháp Text ↔ VSL**: hệ thống hiện là "word substitution" (giữ nguyên thứ tự từ), chưa phải dịch đúng cấu trúc ngữ pháp ký hiệu (topic-comment).
- **Đổi người ký hiệu giữa các gloss**: do dictionary được quay bởi nhiều người ở nhiều thời điểm khác nhau — hạn chế cố hữu của phương pháp concatenative.
- **Fingerspelling làm gián đoạn nhịp điệu tự nhiên** của câu ký hiệu khi gặp từ ngoài dictionary.

## Hướng phát triển tiếp theo

- Seq2seq/Transformer học sắp xếp lại gloss theo đúng ngữ pháp VSL thay vì word-substitution.
- Dùng dữ liệu skeleton đã trích xuất để huấn luyện model nhận diện/sinh chuyển động.
- Bidirectional Maximum Matching thay vì chỉ Forward, giảm rủi ro ghép sai cụm từ mơ hồ.

## Tác giả

<!-- Tên bạn, liên hệ, link project cuối kỳ nếu công khai được -->
