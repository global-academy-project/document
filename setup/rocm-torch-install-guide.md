# Torch ROCm GPU LLM

Windows에서 AMD GPU와 ROCm 지원 PyTorch를 사용하는 설치 안내입니다.

## 사전 조건

- Windows 11 64-bit
- Python 3.12 64-bit
- ROCm for Radeon에서 지원하는 AMD GPU와 드라이버
- PowerShell

PyTorch wheel 파일명에 `cp312`가 포함되어 있어 Python 3.12가 필요합니다. 아래 명령은 `py` 디렉터리에서 실행합니다.

## 설치

### 1. 가상환경 만들기 및 활성화

```powershell
python -m venv llm-venv
.\llm-venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

PowerShell에서 스크립트 실행이 차단되면 현재 사용자 정책을 확인하거나, 아래 명령으로 현재 프로세스에서만 실행을 허용한 뒤 가상환경을 활성화합니다.

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\llm-venv\Scripts\Activate.ps1
```

명령 프롬프트를 사용하는 경우에는 `.\llm-venv\Scripts\activate.bat`을 실행합니다.

### 2. ROCm SDK 설치

```powershell
python -m pip install --no-cache-dir `
    https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm_sdk_core-7.2.1-py3-none-win_amd64.whl `
    https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm_sdk_devel-7.2.1-py3-none-win_amd64.whl `
    https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm_sdk_libraries_custom-7.2.1-py3-none-win_amd64.whl `
    https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm-7.2.1.tar.gz
```

### 3. ROCm용 PyTorch wheel 설치

```powershell
python -m pip install --no-cache-dir `
    "https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/torch-2.9.1%2Brocm7.2.1-cp312-cp312-win_amd64.whl" `
    "https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/torchaudio-2.9.1%2Brocm7.2.1-cp312-cp312-win_amd64.whl" `
    "https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/torchvision-0.24.1%2Brocm7.2.1-cp312-cp312-win_amd64.whl"
```

### 4. 프로젝트 의존성 설치

`requirements.txt`는 ROCm PyTorch wheel을 설치한 다음 실행합니다. 먼저 실행하면 일반 PyPI의 CPU용 PyTorch가 설치될 수 있습니다.

```powershell
python -m pip install -r requirements.txt
```

`requirements.lock`은 현재 설치 패키지 목록을 기록한 파일이며, pip 설치 입력 파일로 사용하지 않습니다.

### 5. 설치 및 GPU 동작 확인

```powershell
python -c "import torch; print('PyTorch:', torch.__version__); print('HIP:', torch.version.hip); print('GPU available:', torch.cuda.is_available()); print('GPU count:', torch.cuda.device_count()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'not available'); x = torch.tensor([1.0, 2.0, 3.0], device='cuda'); print('GPU tensor:', (x * 2).cpu())"
```

정상 설치라면 PyTorch 버전에 `+rocm7.2.1`이 표시되고, HIP 버전이 출력되며, GPU 사용 가능 여부가 `True`입니다. AMD GPU도 PyTorch의 호환 API 이름에 따라 `torch.cuda` 및 `device="cuda"`로 확인합니다.

상세 환경 정보는 다음 명령으로 확인할 수 있습니다.

```powershell
python -m torch.utils.collect_env
```

## 문제 해결

### `offload-arch`를 찾지 못했다는 경고

가상환경을 활성화한 PowerShell에서 실행 중인지 확인합니다. 해당 실행 파일이 venv의 `Scripts`에 있는지 확인하고, 필요하면 현재 세션의 PATH에 추가합니다.

```powershell
Get-Command offload-arch -ErrorAction SilentlyContinue
Get-ChildItem .\llm-venv\Scripts -Filter offload-arch.exe
$env:Path = "$PWD\llm-venv\Scripts;$env:Path"
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

### ROCm SDK 설치 중 경로 길이 오류

설치 로그에 Windows의 긴 파일 경로로 인한 `No such file or directory`가 표시되면, 관리자 권한 PowerShell에서 Windows 긴 경로 지원을 활성화하고 새 터미널에서 설치를 다시 시도합니다.

```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

레지스트리 변경에는 관리자 권한이 필요합니다. 조직 정책상 변경할 수 없다면 관리자에게 문의하세요.

## 참고 자료

- [AMD ROCm for Radeon 설치 안내](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/install/installrad/windows/howto_windows.html)
- [PyTorch HIP semantics](https://docs.pytorch.org/docs/main/notes/hip.html)
- [Windows 긴 경로 지원](https://learn.microsoft.com/windows/win32/fileio/maximum-file-path-limitation#enable-long-paths-in-windows-10-version-1607-and-later)
- [ROCm 호환성 매트릭스](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html?fam=radeon&gpu=amd-radeon-rx-9070-xt&os=windows&gfx=gfx1201)
