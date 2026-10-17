# Frontend Installation Guide

> [!WARNING]
> * **설치 버전은 각 팀의 상호간의 협의로 설정하고 각 맞는 버전을 구성하는 것을 권장한다. 대부분은 Docker에서 일관된 환경을 제공을 하지만 개발 단계에서 각기 다른 버전을 사용한다면 일부 라이브러리 혹은 코드 구조가 상이할 수 있으니 주의해야 한다.**
> * **Windows 11 `Home`, `Pro`, `Pro N`과 상관없이 SAC (`Smart Application Controle`)에서 서명되지 않은 `*.exe`, `*.msi` 등 실행을 막아 개발 언어 바이너리나 컴파일을 `막히는 현상` (`Block`)이 발생하니 반드시 확인해야 한다.**

# 1. Windows Node.js Install

### **Winget search Node.js**

```ps
winget search "node.js"
```

```ps
Name                                       Id                                    Version         Match           Source
-----------------------------------------------------------------------------------------------------------------------
Node.js                                    OpenJS.NodeJS                         26.7.0          Moniker: nodejs winget
<...>
Node.js 20                                 OpenJS.NodeJS.20                      20.20.2         Tag: nodejs     winget
Bun                                        Oven-sh.Bun                           1.4.2           Tag: nodejs     winget
Bun Baseline                               Oven-sh.Bun.Baseline                  1.4.2           Tag: nodejs     winget
Bun Baseline Profile                       Oven-sh.Bun.BaselineProfile           1.4.2           Tag: nodejs     winget
Bun Profile                                Oven-sh.Bun.Profile                   1.4.2           Tag: nodejs     winget
<...>
Node.js 10                                 OpenJS.NodeJS.10                      10.24.1                         winget
Node.js 12                                 OpenJS.NodeJS.12                      12.22.12                        winget
Node.js 14                                 OpenJS.NodeJS.14                      14.21.3                         winget
Node.js 15                                 OpenJS.NodeJS.15                      15.14.0                         winget
Node.js 16                                 OpenJS.NodeJS.16                      16.20.2                         winget
Node.js 17                                 OpenJS.NodeJS.17                      17.9.1                          winget
Node.js 18                                 OpenJS.NodeJS.18                      18.20.8                         winget
Node.js 19                                 OpenJS.NodeJS.19                      19.9.0                          winget
Node.js 21                                 OpenJS.NodeJS.21                      21.7.3                          winget
Node.js 22                                 OpenJS.NodeJS.22                      22.23.2                         winget
Node.js 23                                 OpenJS.NodeJS.23                      23.11.0                         winget
Node.js 4                                  OpenJS.NodeJS.4                       4.8.7                           winget
Node.js 6                                  OpenJS.NodeJS.6                       6.17.0                          winget
Node.js 7                                  OpenJS.NodeJS.7                       7.1.0                           winget
Node.js 8                                  OpenJS.NodeJS.8                       8.11.3                          winget
Node.js (LTS)                              OpenJS.NodeJS.LTS                     24.19.0                         winget
```

### **Winget install Node.js**

```ps
winget install "Node.js <version>"
```

> [!NOTE]
> **Winget으로 설치하는 Runtime이나 Application은 대부분 `C:\Program File\**` 으로 설치 되어 별도의 `Storage`를 증설하여 사용하는 사용자에게는 [Offical Home](https://nodejs.org/en)에서 `LTS Install` 혹은 `Other Install` 통해 경로를 변경하여 설치 진행 하는 것을 추천한다.**

# 2. React with Vite Setup

### Node.js Installed Version Check

```bash
node --version
```

### Npm Package Lib of Installed Version Check

```bash
npm -v
```

> [!NOTE]
> * **npm의 버전이 `11.*.*` 일경우 Terminal에서 Major 버전이 릴리즈 했다는 문구가 나오는데, 이는 사용자의 선택에 따라 버전을 유지할 것인지 아니면 업데이트를 할 것인지 정하면된다.**
> * **작성 기점으로 PS에서는 npm cmd가 받아들이지 못하여 bash를 통해 확인을 했으며, Windows 11 체제 변환과 유저 정책으로 인해 차단 되었거나 일시적인 `Blocking` 현상으로 파악된다.**

### Vite Create

```bash
npm <option> <lib name>@<version> <directory name> -- --<option> <framework name>
```

```bash
npm create vite@latest <directory_name> -- --template react
```

### Lint select

* ESLint [표준]

* Oxlint [확장]

### React [Vite] installing package

```bash
Need to install the following packages:
create-vite@9.2.1
Ok to proceed? (y) y
npm notice run npx
npm notice run create-vite react-web-lab --template react
│
◇  Which linter to use?
│  ESLint
│
◇  Install with npm and start now?
│  Yes
│
◇  Scaffolding project in C:\Users\...\Global-Academy\react-web-lab...
│
◇  Installing dependencies with npm...

added 143 packages, and audited 144 packages in 8s

31 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
│
◇  Starting dev server...
npm notice run react-web-lab@0.0.0 dev
npm notice run vite

  VITE v8.3.1  ready in 2220 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  press h + enter to show help
```

> [!NOTE]
> **Vite는 기본적으로 Port Number를 5173으로 사용하여 접속이 가능하도록 열려있지만 Windows 11 기준으로 Port Number가 10000번대 까지는 Operating System에서 사용하는 Port가 많기 때문에 vite.config.json에서 별도의 Port Number를 명시하여 우회할 수 있도록 해야한다.**

---

# ETC. 참조

* [1. Node.js Offical Home](https://nodejs.org/en)
* [2. Winget Offical Home](https://winget.ragerworks.com/)
