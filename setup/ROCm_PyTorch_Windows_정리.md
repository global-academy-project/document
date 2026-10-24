# Windows에서 PyTorch ROCm `offload-arch` 경고 해결 정리

## 상황
PowerShell에서 아래 명령 실행 시 경고가 발생:

```powershell
python -c "import torch"
```

출력:
```text
[WARNING] failed to run offload-arch: binary not found.
```

---

## 핵심 결론
- `offload-arch` **바이너리 미설치**가 아니라, 대부분 **PATH 미등록** 문제.
- 실제로는 `import torch`가 성공했기 때문에 치명적 오류가 아님.
- 경고 제거는 `offload-arch.exe`가 있는 경로를 PATH에 추가하면 해결됨.

---

## 확인된 환경
- GPU: AMD Radeon RX 9070 XT
- CPU: Ryzen 7 9700X
- Python: 3.12 (Microsoft Store 설치 경로)
- PyTorch: `2.9.1+rocm7.2.1`
- HIP: `7.2.53211-158bd99533`
- CUDA: `None` (정상, ROCm 빌드이므로)

---

## 설치 확인 명령
```powershell
python -c "import torch; print('IMPORT_OK'); print('torch:', torch.__version__); print('HIP:', torch.version.hip); print('CUDA:', torch.version.cuda)"
```

정상 예시:
```text
IMPORT_OK
torch: 2.9.1+rocm7.2.1
HIP: 7.2.xxxxx
CUDA: None
```

---

## 원인 진단
### 1) `offload-arch.exe` 실제 위치 탐색
```powershell
$base = "$env:LOCALAPPDATA\Packages\PythonSoftwareFoundation.Python.3.12_qbz5n2kfra8p0\LocalCache\local-packages\Python312"
Get-ChildItem -Path $base -Recurse -Filter offload-arch.exe -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
```

확인된 위치:
- `...\Python312\Scripts\offload-arch.exe`
- `...\site-packages\_rocm_sdk_core\lib\llvm\bin\offload-arch.exe`

### 2) PATH 인식 확인
```powershell
where.exe offload-arch
```

문제 상태:
```text
INFO: Could not find files for the given pattern(s).
```

---

## 해결 방법
### 현재 세션에 즉시 반영 (임시)
```powershell
$scripts = "$base\Scripts"
$env:Path = "$env:Path;$scripts"
```

검증:
```powershell
where.exe offload-arch
python -c "import torch; print('IMPORT_OK')"
```

정상 상태:
- `where.exe offload-arch`가 경로 출력
- `import torch` 시 경고 사라지고 `IMPORT_OK` 출력

---

## 영구 적용 (권장)
```powershell
$scripts = "$env:LOCALAPPDATA\Packages\PythonSoftwareFoundation.Python.3.12_qbz5n2kfra8p0\LocalCache\local-packages\Python312\Scripts"
$userPath = [Environment]::GetEnvironmentVariable("Path","User")
if ($userPath -notlike "*$scripts*") {
  [Environment]::SetEnvironmentVariable("Path", "$userPath;$scripts", "User")
}
```

> 적용 후 **새 PowerShell 창**에서 다시 확인:
```powershell
where.exe offload-arch
python -c "import torch; print('IMPORT_OK')"
```

---

## 자주 헷갈린 포인트
- `python -c "import torch" 2>nul && echo Success || echo Failure`  
  → 경고가 떠도 종료코드가 0이면 `Success`가 맞음.
- PowerShell에서 오류 리다이렉션은 `2>$null` 형태를 주로 사용.
- `Write-Output "ExitCode=$LASTEXITCODE"where offload-arch`처럼 붙여 쓰면 오동작하므로 줄 분리 필요.

---

## 최종 상태
✅ `offload-arch` 경고 원인 파악 완료  
✅ PATH 설정으로 경고 제거 완료  
✅ `PyTorch ROCm import` 정상 동작 확인 (`IMPORT_OK`)