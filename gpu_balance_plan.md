# Kế hoạch cân bằng tải GPU trên 2 T4 Kaggle

## Bối cảnh

Kaggle cung cấp 2 GPU T4 (mỗi cái 16GB VRAM). Code hiện tại phân chia cứng:
- **GPU 0 (`cuda:0`)**: YOLO models + Flux text encoder (T5-XXL ~4.5GB)
- **GPU 1 (`cuda:1`)**: Flux transformer (~5–6GB)

Vấn đề: SAM, Upscaler, Manga-OCR, PaddleOCR-VL đều dùng `self.device` (= `cuda` = GPU 0), khiến GPU 0 quá tải (~12GB+) trong khi GPU 1 còn nhiều VRAM trống khi không chạy Flux.

---

## Phân tích VRAM

### Trước khi thay đổi

| Model | VRAM | GPU hiện tại |
|---|---|---|
| YOLO ×4 (speech bubble, conjoined, panel, OSB) | ~6.5GB | GPU 0 |
| Flux T5-XXL text encoder | ~4.5GB | GPU 0 |
| SAM 2.1 | ~2.5GB | GPU 0 ❌ |
| Upscaler RCAN | ~1.5GB | GPU 0 ❌ |
| Manga-OCR | ~2GB | GPU 0 ❌ |
| PaddleOCR-VL | ~4–6GB | GPU 0 ❌ |
| **GPU 0 tổng** | **~17–21GB** ❌ | |
| Flux transformer (SDNQ INT4) | ~5.5GB | GPU 1 |
| **GPU 1 tổng** | **~5.5GB** (lãng phí) | |

### Sau khi thay đổi

| GPU 0 | VRAM | GPU 1 | VRAM |
|---|---|---|---|
| YOLO ×4 | ~6.5GB | Flux transformer | ~5.5GB |
| Flux T5-XXL | ~4.5GB | SAM 2.1 | ~2.5GB |
| | | Upscaler RCAN | ~1.5GB |
| | | Manga-OCR | ~2GB |
| **Tổng** | **~11GB** ✅ | **Tổng** | **~11.5GB** ✅ |

> PaddleOCR-VL (~4–6GB) cũng chuyển sang GPU 1. Khi dùng PaddleOCR thay Manga-OCR, GPU 1 ~13–14GB — vẫn trong giới hạn 16GB.

---

## Tính an toàn — không xung đột VRAM

Pipeline chạy tuần tự, các model trên GPU 1 không bao giờ chạy đồng thời với nhau:

```
YOLO (GPU 0) → SAM (GPU 1) → OCR (GPU 1) → Flux (GPU 0 + GPU 1) → Upscaler (GPU 1)
     ↑ detection          ↑ trước Flux              ↑ sau Flux
```

- SAM chạy trong bước detection, kết thúc trước khi Flux bắt đầu
- OCR chạy trước Flux inpainting
- Upscaler chạy sau khi Flux đã xong
- `flux_inference_lock` đã serialize Flux inference → không có race condition

---

## Các thay đổi cần thực hiện

### File 1: `core/device.py`

**Thêm 3 hàm mới** sau `get_flux_text_encoder_device()`:

```python
def get_sam_device() -> torch.device:
    if is_kaggle_environment() and torch.cuda.is_available() and torch.cuda.device_count() >= 2:
        return torch.device("cuda:1")
    if is_kaggle_environment() and torch.cuda.is_available():
        return torch.device("cuda:0")
    return get_best_device()


def get_upscale_device() -> torch.device:
    if is_kaggle_environment() and torch.cuda.is_available() and torch.cuda.device_count() >= 2:
        return torch.device("cuda:1")
    if is_kaggle_environment() and torch.cuda.is_available():
        return torch.device("cuda:0")
    return get_best_device()


def get_ocr_device() -> torch.device:
    if is_kaggle_environment() and torch.cuda.is_available() and torch.cuda.device_count() >= 2:
        return torch.device("cuda:1")
    if is_kaggle_environment() and torch.cuda.is_available():
        return torch.device("cuda:0")
    return get_best_device()
```

**Sửa `get_device_info()`** để nhận đúng device index (hiện tại hardcode GPU 0):

```python
# Dòng 154–161 hiện tại dùng torch.cuda.memory_allocated() không có index
# Sửa thành:
device_index = device.index if device.index is not None else 0
allocated = torch.cuda.memory_allocated(device_index) / 1024**3
reserved = torch.cuda.memory_reserved(device_index) / 1024**3
return {
    "device": torch.cuda.get_device_name(device_index),
    "allocated_gb": f"{allocated:.2f}",
    "reserved_gb": f"{reserved:.2f}",
}
```

---

### File 2: `core/ml/model_manager.py`

**Thêm import** (dòng 20–28):
```python
from core.device import (
    ...
    get_ocr_device,
    get_sam_device,
    get_upscale_device,
)
```

**Thêm device fields trong `__init__`** (sau dòng 81 `self.flux_device = get_flux_device()`):
```python
self.sam_device = get_sam_device()
self.upscale_device = get_upscale_device()
self.ocr_device = get_ocr_device()
```

**Thêm log** (sau dòng 108, theo pattern hiện có):
```python
if self.sam_device != self.device:
    log_message(f"SAM models will use device: {self.sam_device}", always_print=True)
if self.upscale_device != self.device:
    log_message(f"Upscale models will use device: {self.upscale_device}", always_print=True)
if self.ocr_device != self.device:
    log_message(f"OCR models will use device: {self.ocr_device}", always_print=True)
```

**Cập nhật 5 hàm load** — thay `self.device` bằng device tương ứng:

| Hàm | Dòng hiện tại | Thay đổi |
|---|---|---|
| `load_upscale()` | `.to(self.device)` | `.to(self.upscale_device)` |
| `load_upscale_lite()` | `.to(self.device)` | `.to(self.upscale_device)` |
| `load_sam2()` | `.to(self.device)` | `.to(self.sam_device)` |
| `load_sam3()` | `.to(self.device)` | `.to(self.sam_device)` |
| `load_manga_ocr()` | `get_best_device()` / `self.device` | `self.ocr_device` |
| `load_paddle_ocr_vl()` | `self.device` / `get_best_device()` | `self.ocr_device` |

---

## Verification

Sau khi thực hiện, kiểm tra trên Kaggle:

1. **Log khởi động** — phải thấy:
   ```
   SAM models will use device: cuda:1
   Upscale models will use device: cuda:1
   OCR models will use device: cuda:1
   ```

2. **Trong quá trình xử lý** — chạy `!nvidia-smi` trong Kaggle notebook:
   - Cả 2 GPU phải có memory usage > 0
   - Không GPU nào vượt ~14GB

3. **Test với Flux enabled** — xử lý 1 ảnh có bubble màu (kích hoạt Flux inpainting) → không OOM

4. **Test không có Flux** — xử lý 1 ảnh thông thường → GPU 1 vẫn được dùng (OCR/SAM)
