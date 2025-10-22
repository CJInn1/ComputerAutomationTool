# ComputerAutomationTool

## OmniParser V2 on Windows 11 (CPU‑only) – End‑to‑End Setup

This guide reproduces the exact sequence used to get OmniParser V2 running locally on a Dell Inspiron 15 (Windows 11, Intel Iris Xe, CPU‑only) in the folder `C:\Users\JDL\Documents\GitHub\ComputerAutomationTool`.

### Conventions
- **Project root**: `C:\Users\JDL\Documents\GitHub\ComputerAutomationTool`
- **Repo path**: `C:\Users\JDL\Documents\GitHub\ComputerAutomationTool\OmniParser`
- **App to open for commands**: Windows PowerShell
- **Environment**: Conda env named `omni` (Python 3.12)
- **Demo URL**: `http://127.0.0.1:7861`

### 1) Enable Windows Long Paths (one‑time)
App: Registry Editor (`regedit`).
- Navigate to `Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem`.
- Set `LongPathsEnabled` from `0` to `1`.
- Reboot Windows.

### 2) Ensure Conda is available on PATH
App: PowerShell.
```powershell
conda --version
```

**If not recognized, add to system PATH permanently:**
1. Open **Control Panel** → **System** → **Advanced System Settings**
2. Click **Environment Variables**
3. Under **System Variables**, select **Path** and click **Edit**
4. Click **New** and add these paths:
   - `C:\Users\JDL\AppData\Local\anaconda3`
   - `C:\Users\JDL\AppData\Local\anaconda3\Scripts`
5. Click **OK** to save
6. **Restart PowerShell** for changes to take effect

**Alternative (temporary for current session only):**
```powershell
$env:Path = "C:\Users\JDL\AppData\Local\anaconda3;$($env:Path)"
$env:Path = "C:\Users\JDL\AppData\Local\anaconda3\Scripts;$($env:Path)"
```

### 3) Create a clean environment (Python 3.12)
App: PowerShell (any directory).
```powershell
conda create -n omni python==3.12 -y
conda activate omni
python --version
```

### 4) Clone OmniParser into the project directory
App: PowerShell.
```powershell
cd "C:\Users\JDL\Documents\GitHub\ComputerAutomationTool"
git clone https://github.com/microsoft/OmniParser
cd .\OmniParser
```

### 5) Install Hugging Face CLI (login optional)
App: PowerShell, directory: `OmniParser`.
```powershell
pip install "huggingface_hub[cli]"
# Optional if you use a token
huggingface-cli login --token YOUR_TOKEN_HERE
```

### 6) Install base requirements
App: PowerShell, directory: `OmniParser`.
```powershell
pip install -r requirements.txt
```

### 7) Pin versions known to work on CPU‑only Windows
App: PowerShell, directory: `OmniParser`.
```powershell
pip install fastapi==0.104.1 uvicorn[standard]
pip install python-multipart==0.0.20
pip install matplotlib==3.7.2 python-dateutil==2.8.2
pip install python-bidi==0.4.0
pip install paddlepaddle==2.6.2 paddleocr==2.7.3
pip install protobuf==3.20.2 --force-reinstall
pip install transformers==4.40.2 tokenizers==0.19.1
pip uninstall -y gradio gradio_client gradio-client
pip install gradio==3.50.2 gradio-client==0.6.1
```

### 8) Download V2 weights to `weights` and rename caption folder
App: PowerShell, directory: `OmniParser`.
```powershell
mkdir -Force weights | Out-Null

# icon_detect
huggingface-cli download microsoft/OmniParser-v2.0 "icon_detect/train_args.yaml" --local-dir weights
huggingface-cli download microsoft/OmniParser-v2.0 "icon_detect/model.pt"          --local-dir weights
huggingface-cli download microsoft/OmniParser-v2.0 "icon_detect/model.yaml"        --local-dir weights

# icon_caption (will be renamed to icon_caption_florence)
huggingface-cli download microsoft/OmniParser-v2.0 "icon_caption/config.json"            --local-dir weights
huggingface-cli download microsoft/OmniParser-v2.0 "icon_caption/generation_config.json" --local-dir weights
huggingface-cli download microsoft/OmniParser-v2.0 "icon_caption/model.safetensors"      --local-dir weights

if (Test-Path ".\weights\icon_caption_florence") { Remove-Item ".\weights\icon_caption_florence" -Recurse -Force }
Rename-Item -Path ".\weights\icon_caption" -NewName "icon_caption_florence"
```

### 9) CPU‑only workaround: mock `flash_attn`
App: PowerShell (env `omni` active). Paste exactly:
```powershell
@"
# This is a mock flash_attn module to bypass CUDA requirements on CPU-only systems.
class FlashAttention:
    def __init__(self, *args, **kwargs):
        print("Warning: Using mock FlashAttention. This will not provide GPU acceleration.")
        pass
    def __call__(self, *args, **kwargs):
        raise NotImplementedError("FlashAttention is not implemented for CPU-only mode.")
class FlashMHA:
    def __init__(self, *args, **kwargs):
        print("Warning: Using mock FlashMHA. This will not provide GPU acceleration.")
        pass
    def __call__(self, *args, **kwargs):
        raise NotImplementedError("FlashMHA is not implemented for CPU-only mode.")
def flash_attn_func(*args, **kwargs):
    print("Warning: Using mock flash_attn_func. This will not provide GPU acceleration.")
    raise NotImplementedError("flash_attn_func is not implemented for CPU-only mode.")
"@ | Out-File -FilePath "C:\Users\JDL\AppData\Local\anaconda3\envs\omni\Lib\site-packages\flash_attn.py" -Encoding UTF8
```

### 10) Run the demo
App: PowerShell, directory: `OmniParser`.
```powershell
python gradio_demo.py
```
Open in your browser:
```
http://127.0.0.1:7861
```

### Troubleshooting notes (already reflected above)
- Protobuf import/type errors: use `protobuf==3.20.2` and a clean env.
- EasyOCR `bidi`/matplotlib `dateutil` issues: pinned compatible versions.
- Florence‑2/transformers flash_attn import: use `transformers==4.40.2` and the mock `flash_attn.py`.
- Gradio schema TypeError: use `gradio==3.50.2` with `gradio-client==0.6.1`.
