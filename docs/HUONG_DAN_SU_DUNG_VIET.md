# Hướng Dẫn Sử Dụng MangaTranslator

## Mục Lục
- [Giới thiệu](#giới-thiệu)
- [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
- [Cài đặt](#cài-đặt)
- [Thiết lập sau cài đặt](#thiết-lập-sau-cài-đặt)
- [Cách sử dụng](#cách-sử-dụng)
- [Các tùy chọn nâng cao](#các-tùy-chọn-nâng-cao)
- [Xử lý sự cố](#xử-lý-sự-cố)

---

## Giới thiệu

**MangaTranslator** là một ứng dụng web được xây dựng trên Gradio để tự động dịch các trang manga/comic bằng AI. Ứng dụng có thể:

- **Phát hiện bong bóm thoại** - Sử dụng YOLO + SAM 2.1/3 để phát hiện và phân đoạn các bong bóm nói
- **Làm sạch nền** - Xóa text khỏi bong bóm bằng Flux.2 Klein, Flux.1 Kontext hoặc OpenCV
- **Dịch** - Sử dụng LLM để OCR (nhận dạng văn bản) và dịch hỗ trợ **59 ngôn ngữ**
- **Hiển thị text** - Render text với canh chỉnh và các gói font tùy chỉnh
- **Nâng cấp độ phân giải** - Sử dụng 2x-AnimeSharpV4 để tăng chất lượng ảnh
- **Xử lý hàng loạt** - Dịch một trang hoặc nhiều trang cùng lúc, hỗ trợ file ZIP
- **Giao diện đa dạng** - Web UI (Gradio) hoặc dòng lệnh (CLI)

### Ưu điểm:
✅ Dịch tự động một cú click  
✅ Không cần can thiệp thủ công  
✅ Hỗ trợ nhiều ngôn ngữ  
✅ Chất lượng output cao  
✅ Xử lý hàng loạt nhanh chóng  

---

## Yêu cầu hệ thống

### Yêu cầu tối thiểu:
- **Python**: 3.10 trở lên
- **PyTorch**: Hỗ trợ CPU, CUDA, ROCm, XPU, MPS
- **Gói Font**: Các file `.ttf` hoặc `.otf` (đã bao gồm trong gói Portable)
- **LLM/VLM**: Cần API hoặc model local

### Yêu cầu phần cứng (khuyến cáo):
- **GPU**: RTX 3060+ được khuyến cáo (có thể chạy trên CPU nhưng chậm)
- **RAM**: 8GB tối thiểu, 16GB được khuyến cáo
- **Dung lượng ổ đĩa**: 20-50GB cho models
- Xem [HARDWARE_REQUIREMENTS.md](HARDWARE_REQUIREMENTS.md) để biết chi tiết

---

## Cài đặt

### 🎯 Cách 1: Gói Portable (Khuyến cáo)

**Lợi ích**: Không cần cài đặt Python, đã có sẵn mọi thứ

1. **Tải gói Portable**
   - Truy cập: https://github.com/meangrinch/MangaTranslator/releases/tag/portable
   - Tải file ZIP phiên bản mới nhất

2. **Giải nén file ZIP**
   - Windows: Bấm chuột phải → Extract All
   - Linux/macOS: `unzip MangaTranslator.zip`

3. **Chạy setup script**
   - **Windows**: Đôi cấp vào file `setup.bat` trong thư mục vừa giải nén
   - **Linux/macOS**: Mở Terminal, chạy `./setup.sh`
   - Chương trình sẽ tự động:
     - Phát hiện PyTorch phù hợp với GPU của bạn
     - Cài đặt tất cả dependencies
     - Tải các model cần thiết

4. **Khởi động ứng dụng**
   - **Windows**: Đôi cấp vào file `start-webui.bat` trong thư mục `MangaTranslator`
   - **Linux/macOS**: Chạy `./start-webui.sh`
   - Trình duyệt sẽ tự động mở (hoặc vào `http://localhost:7676`)

**Gói font bao gồm:**
- Komika (text thường)
- Cookies (text ngoài bong bóm)
- Comicka (cả hai)
- Roboto (hỗ trợ các kí tự đặc biệt)
- Noto Sans SC (hỗ trợ Tiếng Trung Giản thể)

---

### 🛠️ Cách 2: Cài đặt Thủ công (Dành cho nhà phát triển)

**Yêu cầu**: Python 3.10+, Git đã cài đặt

1. **Clone repository**
   ```bash
   git clone https://github.com/meangrinch/MangaTranslator.git
   cd MangaTranslator
   ```

2. **Tạo virtual environment (khuyến cáo)**
   ```bash
   # Windows
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   
   # Linux/macOS
   python -m venv .venv
   source venv/bin/activate
   ```

3. **Cài đặt PyTorch**
   
   Chọn phù hợp với GPU của bạn tại: https://pytorch.org/get-started/locally/
   
   **Ví dụ cho CUDA 13.0:**
   ```bash
   pip install torch==2.10.0+cu130 torchvision==0.25.0+cu130 --extra-index-url https://download.pytorch.org/whl/cu130
   ```
   
   **Ví dụ cho ROCm 7.1:**
   ```bash
   pip install torch==2.10.0+rocm7.1 torchvision==0.25.0+rocm7.1 --extra-index-url https://download.pytorch.org/whl/rocm7.1
   ```
   
   **Ví dụ cho CPU hoặc M1/M2 Mac:**
   ```bash
   pip install torch==2.10.0 torchvision==0.25.0
   ```

4. **Cài đặt Dependencies**
   ```bash
   pip install -r requirements.txt
   ```

5. **Cài đặt Nunchaku (tùy chọn)**
   - Chỉ cần nếu muốn sử dụng Flux.1 Kontext với Nunchaku
   - Chỉ hỗ trợ CUDA, card RTX 2000-series trở lên
   ```bash
   # Ví dụ Windows Python 3.13 CUDA 13.0
   pip install https://github.com/nunchaku-ai/nunchaku/releases/download/v1.2.1/nunchaku-1.2.1+cu13.0torch2.10-cp313-cp313-win_amd64.whl
   ```

---

## Thiết lập sau cài đặt

### 1. Cấu hình Models

✅ **Tự động**: Ứng dụng sẽ tự động tải và sử dụng tất cả các model cần thiết

Thư mục models sẽ được lưu trong: `./models/`

### 2. Cấu hình Font

Models phần mềm sẽ tìm kiếm font trong thư mục `./fonts/`

**Cách sắp xếp fonts:**

1. Tạo thư mục con cho mỗi bộ font trong `fonts/`
2. Để các file `.ttf` hoặc `.otf` vào thư mục đó
3. Đặt tên file bao gồm `italic`, `bold` để chương trình nhận diện các biến thể

**Ví dụ cấu trúc:**
```
fonts/
├─ CC Wild Words/
│  ├─ CCWildWords-Regular.otf
│  ├─ CCWildWords-Italic.otf
│  ├─ CCWildWords-Bold.otf
│  └─ CCWildWords-BoldItalic.otf
├─ Komika/
│  ├─ KOMIKA-HAND.ttf
│  └─ KOMIKA-HANDBOLD.ttf
└─ Roboto/
   ├─ Roboto-Regular.ttf
   └─ Roboto-Bold.ttf
```

### 3. Cấu hình LLM (Quan trọng!)

Ứng dụng hỗ trợ các nhà cung cấp LLM:
- Google Gemini
- OpenAI GPT
- Anthropic Claude
- xAI Grok
- DeepSeek
- Z.ai
- Moonshot AI
- OpenRouter
- OpenAI-Compatible (llama.cpp, v.v.)

**Cách cấu hình trong Web UI:**
1. Mở ứng dụng web (http://localhost:7676)
2. Vào tab **Config**
3. Chọn nhà cung cấp LLM
4. Nhập API key của bạn
5. Chọn model
6. Nhấn Save để lưu cấu hình

**Cách cấu hình qua Environment Variables (CLI):**
```bash
# Google
export GOOGLE_API_KEY=your_key_here

# OpenAI
export OPENAI_API_KEY=your_key_here

# Anthropic
export ANTHROPIC_API_KEY=your_key_here

# xAI
export XAI_API_KEY=your_key_here

# Khác
export DEEPSEEK_API_KEY=...
export ZAI_API_KEY=...
export MOONSHOT_API_KEY=...
export OPENROUTER_API_KEY=...
export OPENAI_COMPATIBLE_API_KEY=...
export HUGGINGFACE_TOKEN=...
```

### 4. Cấu hình OSB Text Pipeline (Tùy chọn)

OSB = Outside Speech Bubble text (text ngoài bong bóm)

**Nếu muốn sử dụng:**
1. Tạo tài khoản Hugging Face: https://huggingface.co/
2. Chấp nhận điều khoản của các model:
   - [AnimeText_yolo](https://huggingface.co/deepghs/AnimeText_yolo)
   - [FLUX.1 Kontext](https://huggingface.co/black-forest-labs/FLUX.1-Kontext-dev) (nếu dùng)
   - [SAM 3](https://huggingface.co/facebook/sam3) (nếu dùng)
3. Tạo Hugging Face token với quyền "Read access to gated repos"
4. Trong Web UI, vào tab Config và nhập HF token
5. Hoặc thiết lập biến môi trường: `HUGGINGFACE_TOKEN=your_token`

---

## Cách sử dụng

### 🌐 Phương pháp 1: Web UI (Gradio) - Dễ nhất

**Khởi động:**
```bash
# Cách 1: Portable package
# Windows: Bấp đôi start-webui.bat
# Linux/macOS: Chạy ./start-webui.sh

# Cách 2: Manual install
python app.py --open-browser
```

**Các tùy chọn khởi động:**
```bash
python app.py \
  --port 7676 \                    # Port (mặc định: 7676)
  --models ./models \              # Đường dẫn thư mục models
  --fonts ./fonts \                # Đường dẫn thư mục fonts
  --open-browser \                 # Tự động mở trình duyệt
  --cpu                            # Bắt buộc dùng CPU (bỏ qua GPU)
```

**Hướng dẫn sử dụng Web UI:**

1. **Mở giao diện**: Truy cập http://localhost:7676
2. **Cấu hình LLM** (lần đầu):
   - Nhấp vào tab **Config** 
   - Chọn nhà cung cấp LLM
   - Nhập API key
   - Chọn model
   - Nhấn "Save Config"

3. **Dịch ảnh**:
   - Quay lại tab **Translator**
   - Tải ảnh (kéo thả hoặc bấm chọn)
   - Chọn ngôn ngữ nguồn (thường là Japanese)
   - Chọn ngôn ngữ đích (thường là English)
   - Nhấn **Translate** và chờ
   - Kết quả sẽ hiển thị ở bên phải

4. **Tùy chọn bổ sung**:
   - **OSB Text**: Nếu bạn muốn dịch text ngoài bong bóm
   - **Font Selection**: Chọn font cho text tiếng Anh
   - **Upscale**: Nâng cấp độ phân giải ảnh

⏱️ **Lần đầu chạy:** Chương trình sẽ tải models (khoảng 1-2 phút)

---

### 💻 Phương pháp 2: Dòng lệnh (CLI)

CLI hữu ích khi bạn muốn xử lý hàng loạt hoặc tự động hóa.

**Dịch một ảnh:**
```bash
python main.py --input image.jpg \
  --output translated.jpg \
  --provider Google \
  --google-api-key YOUR_API_KEY \
  --font-dir "./fonts/Komika"
```

**Dịch một thư mục (Batch):**
```bash
python main.py --input ./manga_pages \
  --output ./translated_output \
  --batch \
  --provider OpenAI \
  --openai-api-key YOUR_API_KEY \
  --font-dir "./fonts/Komika"
```

**Dịch file ZIP:**
```bash
python main.py --input manga_collection.zip \
  --output ./output \
  --batch \
  --provider Google \
  --google-api-key YOUR_API_KEY
```

**Xử lý song song (tối đa 10 ảnh cùng lúc):**
```bash
python main.py --input ./manga_pages \
  --batch \
  --parallel-requests 4 \
  --provider OpenAI \
  --openai-api-key YOUR_API_KEY
```

**Thêm ngữ cảnh từ các trang trước:**
```bash
python main.py --input ./manga_pages \
  --batch \
  --batch-previous-context-images 2 \
  --batch-previous-context-texts 5 \
  --provider Google \
  --google-api-key YOUR_API_KEY
```

**Chỉ làm sạch (không dịch):**
```bash
python main.py --input image.jpg --cleaning-only
```

**Chỉ nâng cấp (không phát hiện/dịch):**
```bash
python main.py --input image.jpg \
  --upscaling-only \
  --image-upscale-factor 2.0 \
  --image-upscale-mode final
```

**Chế độ test (render placeholder, không dịch):**
```bash
python main.py --input image.jpg --test-mode
```

**Xem tất cả các tùy chọn:**
```bash
python main.py --help
```

---

## Các tùy chọn nâng cao

### Các tham số chính:

| Tham số | Mô tả | Ví dụ |
|---------|-------|-------|
| `--input` | Đường dẫn ảnh/thư mục | `image.jpg` hoặc `./folder` |
| `--output` | Nơi lưu kết quả | `./output` |
| `--batch` | Xử lý thư mục/ZIP | `--batch` |
| `--parallel-requests` | Số ảnh xử lý song song (1-10) | `--parallel-requests 4` |
| `--input-language` | Ngôn ngữ nguồn | `Japanese`, `Chinese`, ... |
| `--output-language` | Ngôn ngữ đích | `English`, `Vietnamese`, ... |
| `--provider` | Nhà cung cấp LLM | `Google`, `OpenAI`, `Anthropic` |
| `--font-dir` | Đường dẫn thư mục font | `./fonts/Komika` |
| `--cpu` | Bắt buộc dùng CPU | `--cpu` |
| `--cleaning-only` | Chỉ làm sạch | `--cleaning-only` |
| `--upscaling-only` | Chỉ nâng cấp | `--upscaling-only` |
| `--test-mode` | Chế độ test | `--test-mode` |

### Các loại LLM được hỗ trợ:

**Google Gemini:**
```bash
--provider Google --google-api-key YOUR_KEY
```

**OpenAI (GPT-4, GPT-4o):**
```bash
--provider OpenAI --openai-api-key YOUR_KEY --openai-model gpt-4o
```

**Anthropic Claude:**
```bash
--provider Anthropic --anthropic-api-key YOUR_KEY
```

**xAI Grok:**
```bash
--provider xAI --xai-api-key YOUR_KEY
```

**DeepSeek:**
```bash
--provider DeepSeek --deepseek-api-key YOUR_KEY
```

**OpenAI-Compatible (llama.cpp, local models):**
```bash
--provider OpenAI-Compatible \
  --openai-compatible-url http://localhost:8080/v1 \
  --openai-compatible-api-key optional_key
```

### Cấu hình Inpainting (Làm sạch):

- **SDNQ (mặc định)**: Nhanh, chất lượng tốt
- **Flux.2 Klein**: Chất lượng cao hơn
- **Flux.1 Kontext**: Chất lượng nhất, chậm nhất
- **OpenCV**: Cơ bản, không sử dụng AI

---

## Xử lý sự cố

### ❓ Vấn đề: "CUDA out of memory"

**Giải pháp:**
```bash
# Sử dụng CPU thay vì GPU
python app.py --cpu

# Hoặc cho CLI
python main.py --input image.jpg --cpu
```

### ❓ Vấn đề: Models không tải

**Giải pháp:**
1. Kiểm tra kết nối internet
2. Xóa thư mục `./models` và để ứng dụng tải lại
3. Kiểm tra dung lượng ổ đĩa (cần ~50GB)

### ❓ Vấn đề: API key lỗi

**Giải pháp:**
1. Kiểm tra lại API key (có khoảng trắng thừa?)
2. Đảm bảo API key còn hạn sử dụng
3. Kiểm tra quota API của bạn

### ❓ Vấn đề: Font không hiển thị

**Giải pháp:**
1. Đảm bảo file font nằm trong thư mục `./fonts/`
2. Kiểm tra tên file bao gồm `.ttf` hoặc `.otf`
3. Reload ứng dụng sau khi thêm font

### ❓ Vấn đề: OCR nhận dạng sai

**Giải pháp:**
1. Thử một LLM khác (Google, OpenAI, v.v.)
2. Thử model khác (GPT-4 chính xác hơn GPT-4o)
3. Đảm bảo hình ảnh có độ phân giải cao (1000x1000+ pixels)

### ❓ Vấn đề: Ứng dụng chạy chậm

**Giải pháp:**
- Sử dụng GPU (NVIDIA CUDA hơn CPU 10x)
- Giảm `--parallel-requests` xuống 1
- Đóng các ứng dụng khác để giải phóng RAM
- Xem [HARDWARE_REQUIREMENTS.md](HARDWARE_REQUIREMENTS.md)

### ❓ Vấn đề: Lỗi "OSB text pipeline not available"

**Giải pháp:**
1. Đảm bảo bạn đã cài đặt `--osb-enable`
2. Cấu hình Hugging Face token
3. Chấp nhận điều khoản trên Hugging Face

---

## Cập nhật ứng dụng

### Gói Portable:
```bash
# Windows
update.bat

# Linux/macOS
./update.sh
```

### Cài đặt Thủ công:
```bash
git pull
pip install -r requirements.txt
```

---

## Tài liệu Thêm

- [Yêu cầu Phần cứng](HARDWARE_REQUIREMENTS.md)
- [Font Được Khuyến cáo](FONTS.md)
- [Xử lý Sự cố Chi tiết](TROUBLESHOOTING.md)

---

## Liên hệ và Hỗ trợ

- **GitHub Issues**: https://github.com/meangrinch/MangaTranslator/issues
- **Discussions**: https://github.com/meangrinch/MangaTranslator/discussions

---

## Giấy phép

- **Giấy phép**: Apache-2.0
- **Tác giả**: [grinnch](https://github.com/meangrinch)

Thành công! 🎉
