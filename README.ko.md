# noroo-deploy

[English](README.md) | **한국어**

서버 한 대에 여러 사이트를 올리기 위한 배포 도구 `deployctl`과, 그 서버를 운영하는 규칙집입니다.

저장소를 고르면 명령 하나로 포트 배정부터 HTTPS 응답 확인까지 끝납니다. 도중에 실패하면 그 실행에서 만든 것만 되돌립니다.

```
deployctl init
```

## 구성

| 파일 | 내용 |
|---|---|
| [`deployctl`](deployctl) | 배포 도구. Python 표준 라이브러리만 쓰는 파일 하나입니다 |
| [`CLAUDE.md`](CLAUDE.md) | 서버 규칙집. 이 서버에서 작업할 때 Claude Code가 따르는 규칙과, `deployctl`이 지금처럼 동작하는 이유가 담겨 있습니다 |
| [`mac-server-setup.md`](mac-server-setup.md) | Mac mini에서 처음 시작할 때의 설정 기록. 더 이상 관리하지 않습니다 |

## 구조

```
인터넷 ──► Caddy (80/443, 인증서 자동 발급)
              │  127.0.0.1:9000–9099
              ▼
        사이트마다 컨테이너 하나 (Docker Compose)
              │  dbnet
              ▼
        공용 Postgres (사이트마다 전용 데이터베이스와 사용자)
```

- 인터넷에 열린 포트는 80과 443뿐이고, Caddy만 씁니다. 컨테이너는 `127.0.0.1`에만 바인딩합니다.
- Postgres는 포트를 열지 않습니다. 각 사이트는 자기 데이터베이스에만 접근할 수 있고, 데이터베이스를 만든 직후 격리가 제대로 됐는지 확인합니다.
- 사이트는 `~/srv/<사이트>/app.json` 파일 하나로 정의되고, Caddy 설정은 이 파일에서 생성됩니다.
- 배포는 저장소마다 하나씩 둔 GitHub Actions self-hosted 러너에서 `deployctl deploy`를 호출해 실행됩니다.

## 명령

| 명령 | 하는 일 |
|---|---|
| `init` | 사이트 등록: 저장소 선택 → 클론 → 포트 배정 → 데이터베이스 생성 → 워크플로와 러너 설정 → 첫 배포 |
| `validate` | 규칙 검사: 포트·도메인 충돌, DNS, 이미지 아키텍처, `127.0.0.1` 바인딩, 접속 수 여유 등 |
| `deploy` | 검사 → 빌드 → 기동 → Caddy 등록 → HTTPS 확인. 실패하면 되돌립니다 |
| `status` | 컨테이너, Caddy 등록, HTTPS, 접속 사용량 |
| `scaffold` | 없는 `Dockerfile`이나 compose 파일의 초안을 만듭니다. Dockerfile은 `claude -p`가 쓰고 `docker build`로 검증합니다 |
| `workflow` | `.github/workflows/deploy.yml`을 생성합니다 |
| `runner` | 저장소 전용 Actions 러너를 등록하거나 제거합니다 |
| `remove` | 컨테이너와 Caddy 사이트를 내립니다. 데이터베이스와 매니페스트는 남겨 둡니다 |

옵션은 `deployctl <명령> --help`로 볼 수 있습니다. 자세한 내용과 그 이유는 [`CLAUDE.md` §11](CLAUDE.md)에 있습니다.

## 설계 원칙

- **전부 아니면 전무.** `init`이 중간에 실패하면 그 실행에서 만든 컨테이너, Caddy 파일, 데이터베이스, 매니페스트만 지웁니다. 원래 있던 것은 건드리지 않습니다.
- **기존 파일은 덮어쓰지 않습니다.** 직접 작성한 `Dockerfile`, compose 파일, 워크플로는 `--force`를 주지 않는 한 그대로 둡니다. 규칙 위반은 고치지 않고 알려만 줍니다.
- **규칙은 기계가 검사합니다.** 사람이 기억해야 했을 것들을 `validate`가 배포 때마다 미리 확인합니다.
- **배포 경로에 LLM이 없습니다.** `claude`는 초안을 만드는 `scaffold`에서만 씁니다. 러너에서 하는 배포는 재현 가능해야 하기 때문입니다.
- **self-hosted 러너는 비공개 저장소에만 붙입니다.** 공개 저장소는 `init` 목록에서 빠집니다.

## 필요 조건

`deployctl`은 범용 도구가 아닙니다. 서버 한 대의 운영 방식을 코드로 적어 둔 것입니다. 그대로 쓰려면 다음이 필요합니다.

- Linux(amd64/arm64) 또는 macOS, Python 3, Compose 플러그인이 있는 Docker
- 호스트에서 실행 중인 Caddy, 관리 API는 `127.0.0.1:2019`에 열려 있어야 합니다
- `~/srv/postgres`에 있는 공용 Postgres. `dbnet` 네트워크에서 `postgres`라는 이름의 컨테이너로 실행 중이어야 합니다
- 로그인된 `gh` CLI (저장소 목록, 클론, 러너 등록에 씁니다)
- Caddy 설정 경로는 `/opt/homebrew/etc/`로 고정되어 있습니다. Linux에서는 이 경로에 `/etc/caddy`를 가리키는 심볼릭 링크를 두세요([`CLAUDE.md` §6-1](CLAUDE.md))

경로, 포트 범위 같은 값은 `deployctl` 맨 위에 상수로 정의되어 있습니다.

## 문서 속 이름에 대해

공개하면서 계정 이름, 도메인, 사이트 이름을 `deploy`, `example.com`, `myapp`으로 바꿨습니다. 문서에 있는 날짜별 작업 기록은 서버를 macOS에서 Linux로 옮긴 실제 기록입니다.
