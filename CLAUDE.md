# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Chạy Web UI:**
```bash
python app.py --open-browser
python app.py --port 7676 --models ./models --fonts ./fonts
```

**Chạy CLI:**
```bash
python main.py --input image.jpg --provider Google --google-api-key YOUR_KEY
python main.py --input ./folder --batch --provider OpenAI --openai-api-key YOUR_KEY
```

**Cài đặt dependencies:**
```bash
pip install -e .
pip install -e ".[dev]"  # Bao gồm black, flake8, isort
```

**Lint:**
```bash
black .
flake8 .
isort .
```

Không có test framework tự động — kiểm thử thủ công qua UI hoặc CLI.

## Architecture

Pipeline dịch manga chạy tuần tự qua các bước:

```
Input Image → Detection → Cleaning → OCR → Translation → Rendering → [Upscaling] → Output
```

### Core modules (`core/`)

- **`pipeline.py`** — Điều phối toàn bộ pipeline. `translate_and_render()` xử lý một ảnh, `batch_translate_images()` xử lý hàng loạt với song song hóa. Đây là file trung tâm kết nối tất cả các module.
- **`config.py`** — Dataclass cấu hình: `DetectionConfig`, `CleaningConfig`, `TranslationConfig`, `RenderingConfig`, `OutsideTextConfig`, `OutputConfig`. Mọi tham số pipeline đều đi qua đây.
- **`device.py`** — Quản lý thiết bị (CUDA/XPU/MPS/CPU), tối ưu multi-GPU cho môi trường Kaggle, chọn dtype tối ưu.
- **`caching.py`** — Hệ thống cache thống nhất cho model và kết quả trung gian.

### Image processing (`core/image/`)

- **`detection.py`** — Phát hiện speech bubble bằng YOLO, panel detection, SAM 2.1/3 segmentation, sắp xếp thứ tự đọc RTL/LTR.
- **`cleaning.py`** — Inpaint bubble đã phát hiện bằng OpenCV (threshold + morphological ops).
- **`inpainting.py`** — Inpaint bằng Flux model: `FluxKleinInpainter` (Flux.2 Klein 4B/9B) và `FluxKontextInpainter` (Flux.1 Kontext với Nunchaku/SDNQ backend).
- **`ocr_detection.py`** — Phát hiện vùng text trong bubble.

### ML (`core/ml/`)

- **`model_manager.py`** — Singleton quản lý vòng đời model (YOLO, SAM, Flux, OCR, upscaler). Tự động tải từ HuggingFace, cache model, hỗ trợ multi-GPU.

### Services (`core/services/`)

- **`translation.py`** — Interface thống nhất cho 9 LLM provider. `call_translation_api_batch()` là entry point chính. Xử lý OCR (LLM/Manga-OCR/PaddleOCR-VL), reasoning model, context ảnh/text, glossary.

### Text rendering (`core/text/`)

- **`text_renderer.py`** — Render text bằng Skia. Hỗ trợ vertical text, RTL, subpixel rendering, supersampling.
- **`layout_engine.py`** — Tính toán layout text trong bubble (hyphenation, wrapping, Hangul).
- **`font_manager.py`** — Quản lý font pack, phát hiện variant (bold/italic).

### LLM endpoints (`utils/endpoints/`)

Mỗi file tương ứng một provider: `google.py`, `openai.py`, `anthropic.py`, `xai.py`, `deepseek.py`, `zai.py`, `moonshot.py`, `openrouter.py`, `openai_compatible.py`. Khi thêm provider mới, tạo file mới ở đây và đăng ký trong `utils/endpoints/__init__.py` và `core/services/translation.py`.

### UI (`ui/`)

- **`layout.py`** — Định nghĩa toàn bộ Gradio UI (tabs: Upload, Config, Advanced, Batch, Results).
- **`callbacks.py`** — Event handler cho UI, gọi vào pipeline.
- **`settings_manager.py`** — Lưu/tải config JSON tại `%LOCALAPPDATA%/MangaTranslator/config.json` (Windows) hoặc `~/.config/MangaTranslator/config.json`.

### Entry points

- **`app.py`** — Khởi động Gradio web server, parse args, gọi `ui/layout.py`.
- **`main.py`** — CLI entry point, parse args, gọi trực tiếp vào pipeline.

## Key conventions

- Model được tải lazy và cache trong `ModelManager` singleton — không khởi tạo model trực tiếp ngoài `model_manager.py`.
- Mọi cấu hình pipeline truyền qua dataclass trong `core/config.py`, không dùng global state.
- Kaggle multi-GPU: YOLO và Flux chạy trên GPU riêng biệt — xem `core/device.py` để biết logic phân bổ.
- `core/outside_text_processor.py` xử lý text ngoài bubble (OSB) như một sub-pipeline độc lập.
