# Backend Python Installation Guide

> [!WARNING]
> * **설치 버전은 각 팀의 상호간의 협의로 설정하고 각 맞는 버전을 구성하는 것을 권장한다. 대부분은 Docker에서 일관된 환경을 제공을 하지만 개발 단계에서 각기 다른 버전을 사용한다면 일부 라이브러리 혹은 코드 구조가 상이할 수 있으니 주의해야 한다.**
> * **Windows 11 `Home`, `Pro`, `Pro N`과 상관없이 SAC (`Smart Application Controle`)에서 서명되지 않은 `*.exe`, `*.msi` 등 실행을 막아 개발 언어 바이너리나 컴파일을 `막히는 현상` (`Block`)이 발생하니 반드시 확인해야 한다.**

# 1. Python Install

### Winget Python

```ps
winget search python
```

```ps
Name                           Id                             Version           Match                           Source
-----------------------------------------------------------------------------------------------------------------------
<...>
Python Launcher                Python.Launcher                3.14.7            Command: python                 winget
Python 2                       Python.Python.2                2.7.18150         Command: python                 winget
Python 3.0                     Python.Python.3.0              3.0.1             Command: python                 winget
Python 3.1                     Python.Python.3.1              3.1.4             Command: python                 winget
Python 3.10                    Python.Python.3.10             3.10.11           Command: python                 winget
Python 3.11                    Python.Python.3.11             3.11.9            Command: python                 winget
Python 3.12                    Python.Python.3.12             3.12.10           Command: python                 winget
Python 3.13                    Python.Python.3.13             3.13.15           Command: python                 winget
Python 3.14                    Python.Python.3.14             3.14.7            Command: python                 winget
Python 3.2                     Python.Python.3.2              3.2.5             Command: python                 winget
Python 3.3                     Python.Python.3.3              3.3.5             Command: python                 winget
Python 3.4                     Python.Python.3.4              3.4.4             Command: python                 winget
Python 3.5                     Python.Python.3.5              3.5.4             Command: python                 winget
Python 3.6                     Python.Python.3.6              3.6.8             Command: python                 winget
Python 3.7                     Python.Python.3.7              3.7.9             Command: python                 winget
Python 3.8                     Python.Python.3.8              3.8.10            Command: python                 winget
Python 3.9                     Python.Python.3.9              3.9.13            Command: python                 winget
Python Install Manager         Python.PythonInstallManager    26.3.240.0        Command: python                 winget
<...>
```

```ps
winget install "python {version}"
```


# 2. FastAPI Framework Install

### Install FastAPI

```bash
pip install fastapi
```

### FastAPI Run

```bash
python -m uvicorn main:app --<option> --reload
```

```bash
python -m uvicorn main:app --port 60001 --reload
```

> [!NOTE]
> **이렇게 Python Module을 사용하는 이유는 현재 해당 문서 작성자의 Terminal에서 uvicorn을 단독으로 사용 불가 현상이 발생하고 있어, 임시적으로 우회하는 방법으로 사용중이다. 또한 기본적으로 `--port`을 별도로 옵션을 사용하지 않으면 port가 바인딩 되지 않으므로 배포시 옵션을 필수로 기입해야한다.**

---

# ETC. 참조

* [1. Python Offical Home](https://www.python.org/)
* [2. Winget Offical Home](https://winget.ragerworks.com/)