# 인프라 규칙서

이 저장소는 **이 서버를 다중 사이트 호스팅 서버로 운영**하기 위한 설정을 관리한다.
Claude Code는 이 폴더에서 서버 구성 작업을 수행하며, 아래 규칙을 **위반해서는 안 된다**.

상세 셋업 절차는 [mac-server-setup.md](mac-server-setup.md) 참조 — **단, 그 문서는 예전 macOS
서버를 전제로 쓰였고 지금은 갱신되지 않았다.** 이 문서(CLAUDE.md)가 실제 서버와 어긋나면
이쪽을 따른다.

---

## 1. 하드웨어 / 플랫폼 (고정 전제)

> 이 서버는 원래 macOS(Mac mini, arm64) 전제로 설계된 도구(`deployctl`)와 문서 위에서
> 운영된다. 2026-09-06 작업으로 도구 쪽을 이 플랫폼에 맞게 고쳤고(§11 참고), 하드웨어
> 아키텍처만큼은 소프트웨어로 바꿀 수 없어 `deployctl`이 **실행 시점에 자동판별**하도록
> 했다. 아래 규칙의 "amd64"라는 표현은 이 서버가 x86_64이기 때문이며, 호스트가 바뀌면
> 도구는 따라가지만 이 문서는 그렇지 않으니 실제 하드웨어가 바뀌면 1절부터 다시 써야 한다.

---

## 2. 설치된 스택 (2026-09-07 기준)

| 소프트웨어 | 버전 | 설치 방식 | 상태 |
|---|---|---|---|
| Docker Engine | 29.8.0 | `apt` (네이티브, VM 불필요) | 실행 중 (`docker.service`, enabled) |
| Docker Compose | v5.5.1 | Docker CLI 플러그인 | — |
| Caddy | v2.11.4 | `apt` (네이티브) | 실행 중 (`caddy.service`, enabled) |
| PostgreSQL | 17.11-alpine | `~/srv/postgres` compose (컨테이너) | **실행 중** (healthy, amd64, 슈퍼유저 `postgres`) |

- **colima는 없다.** 리눅스는 컨테이너를 네이티브로 돌리므로 macOS처럼 VM이 필요 없다.
  Docker Desktop / OrbStack도 쓰지 않는다 — 애초에 필요 없다.
- Docker·Caddy 모두 `systemd`가 관리하는 **시스템 서비스**다(사용자 서비스 아님). 부팅 시
  로그인 없이 자동 기동된다(§9).
- `deploy`는 `docker` 그룹과 `caddy` 그룹 소속이다(후자는 2026-09-06에 추가, §6 참고).
  **그룹 변경은 새 로그인/새 프로세스부터 반영된다** — 이미 떠 있는 셸에서는 안 보인다.

---

## 3. 이미지 아키텍처 — amd64 (x86_64) 전용

> **규칙 3-1.** 모든 컨테이너 이미지는 이 서버의 네이티브 아키텍처(**`linux/amd64`**)여야
> 한다.

- 이미지 선택 시 amd64 지원 여부를 **먼저 확인**한다. 미지원이면 그 이미지를 쓰지 말고
  대안을 찾는다.
- `Dockerfile` 베이스 이미지는 멀티아치 태그를 쓴다. 다른 아키텍처로 고정된 태그(`arm64-*`
  등) 금지.
- compose에 `platform:`을 이 서버와 다른 값으로 **적지 않는다.**
- 빌드는 이 서버에서 수행하므로 별도 플래그 없이 amd64가 된다. `--platform` 지정 불필요.
- 에뮬레이션(Rosetta에 해당하는 것)은 **설정돼 있지 않다.** arm64 전용 이미지밖에 없는
  소프트웨어는 이 서버에서 못 돌린다 — amd64 대안을 찾거나 포기한다.

확인 명령:
```bash
docker image inspect <image> --format '{{.Os}}/{{.Architecture}}'   # linux/amd64 이어야 함
docker buildx imagetools inspect <image>                            # 원격 이미지의 지원 아키텍처
```

`deployctl`은 이 값을 하드코딩하지 않고 `platform.machine()`으로 실행 시점에 판별한다
(`HOST_ARCH`, `deployctl:52-53`). 이 서버가 바뀌어도 도구는 따라가지만, 위 규칙 문구는
사람이 읽는 것이라 자동으로 안 바뀐다.

---

## 4. 데이터베이스 — 공유 Postgres

> **규칙 4-1.** Postgres는 **`postgres:17-alpine` 인스턴스 1개**만 띄우고 **전 사이트가
> 공유**한다. 사이트마다 별도 Postgres 컨테이너를 만들지 않는다.

> **규칙 4-2.** 사이트마다 **전용 데이터베이스 + 전용 유저**를 만든다.
> 각 유저는 **자기 DB에만** 접근 권한을 가진다.

> **규칙 4-3.** 앱의 커넥션 풀은 **사이트당 최대 5개**로 제한한다.

### 4-1. 공유 인스턴스 정의

`~/srv/postgres/compose.yaml`:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_SUPERUSER_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    # ports 를 열지 않는다 — 외부/호스트 노출 금지, dbnet 네트워크로만 접근
    networks: [dbnet]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 5s
      retries: 3
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

volumes:
  pgdata:

networks:
  # 이미 존재하는 네트워크를 그대로 쓴다 (사이트 compose들도 동일하게 external 로 참여).
  dbnet:
    external: true
```

- `container_name: postgres`를 **반드시 명시한다.** `deployctl`이 `docker exec -i postgres
  ...`로 이 이름을 그대로 참조한다(`PG_CONTAINER = "postgres"`, 하드코딩). compose
  프로젝트명 추정에 맡기면 이름이 달라져 깨진다.
- `dbnet` 네트워크는 **이 compose가 만들지 않는다** — 이미 떠 있는 브리지 네트워크를
  `external: true`로 참여만 한다(사이트 compose들과 동일한 패턴). 처음부터 없다면
  `docker network create dbnet`으로 한 번 만들어 둔다.
- 슈퍼유저는 `postgres`다. `POSTGRES_USER`를 커스텀 값으로 주면 공식 이미지가 `postgres`
  롤 자체를 안 만드므로 **절대 `POSTGRES_USER`를 지정하지 않는다.**

각 사이트의 compose는 이 네트워크에 **외부 네트워크로 참여**한다:

```yaml
networks:
  dbnet:
    external: true
```

### 4-2. 사이트 추가 시 DB 프로비저닝

새 사이트 `foo`를 올릴 때 아래를 그대로 수행한다.

```sql
-- 1) 전용 유저 + 전용 DB (유저를 DB 소유자로 지정)
CREATE ROLE foo_user LOGIN PASSWORD '<강한 난수 암호>';
CREATE DATABASE foo OWNER foo_user;

-- 2) 다른 사이트 유저의 접근 차단 (PUBLIC 기본 CONNECT 권한 회수)
REVOKE ALL ON DATABASE foo FROM PUBLIC;
GRANT CONNECT ON DATABASE foo TO foo_user;
```

> **왜 `OWNER`로 만드는가**: Postgres 15+에서 `public` 스키마는 `pg_database_owner`가 소유한다.
> DB 소유자로 지정하면 자기 DB 안에서는 별도 GRANT 없이 완전한 권한을 갖는다.

> **왜 `REVOKE ... FROM PUBLIC`이 필수인가**: Postgres는 기본적으로 **모든 롤에게 모든 DB의
> CONNECT 권한**을 준다. 이걸 회수하지 않으면 `foo_user`가 `bar` DB에 접속할 수 있다.
> **새 DB를 만들 때마다 반드시 실행한다.**

**기본 DB에 대한 1회성 조치 (적용됨, 2026-09-06)**

`postgres`와 `template1`도 PUBLIC에 CONNECT가 열려 있어 사이트 유저가 접속 가능했다. 회수 완료:

```sql
REVOKE CONNECT ON DATABASE postgres FROM PUBLIC;
REVOKE CONNECT ON DATABASE template1 FROM PUBLIC;
```

superuser(`postgres`)는 권한 검사를 우회하므로 관리·백업 작업에는 영향이 없다.
**볼륨을 새로 만들어 재초기화하는 경우 이 두 줄을 다시 실행해야 한다.**

**격리의 한계 (알고 있을 것)**: 위 조치로 타 사이트 DB의 **데이터 접근은 차단**되지만,
`pg_database` 카탈로그는 여전히 읽을 수 있어 **다른 DB의 이름 목록은 보인다.**
Postgres 구조상 막기 어렵다. DB명에 민감한 정보를 넣지 말 것.

검증 (반드시 수행):
```bash
# 자기 DB 접속 → 성공해야 함
docker exec -i postgres psql -U foo_user -d foo -c 'SELECT current_database();'
# 남의 DB 접속 → 반드시 실패해야 함
docker exec -i postgres psql -U foo_user -d bar -c 'SELECT 1;'
```

### 4-3. 커넥션 풀 제한

사이트당 **5개**를 넘기지 않는다. 프레임워크별 설정 위치:

| 스택 | 설정 |
|---|---|
| Node (`pg`) | `new Pool({ max: 5 })` |
| Prisma | `?connection_limit=5` (DATABASE_URL 쿼리스트링) |
| SQLAlchemy | `pool_size=5, max_overflow=0` |
| Django | `CONN_MAX_AGE` + 외부 풀러 |

`max_overflow`처럼 상한을 초과시키는 옵션은 **0으로 고정**한다.

Postgres 쪽 상한도 함께 관리한다:
```
max_connections >= (사이트 수 x 5) + 10   # +10은 관리/백업용 여유분
```
기본값은 100이라 사이트 몇 개로는 여유롭다. 사이트를 추가할 때 이 값을 재확인하고,
부족하면 `max_connections`를 올린 뒤 재시작한다.

### 4-4. 접속 문자열

컨테이너 간 통신이므로 호스트명은 서비스명 `postgres`, 포트는 5432 그대로 쓴다.

```
postgresql://foo_user:<암호>@postgres:5432/foo?connection_limit=5
```

암호는 사이트별 `.env`에 두고 **git에 커밋하지 않는다** (`chmod 600`).

---

## 5. 네트워크 / 포트

> **규칙 5-1.** 컨테이너 포트는 **`127.0.0.1:`에만 바인딩**한다.

```yaml
ports:
  - "127.0.0.1:3000:3000"    # O
  - "3000:3000"              # X — 0.0.0.0에 열려 Caddy의 TLS를 우회하는 평문 경로가 생김
```

> **규칙 5-2.** Postgres는 `ports`를 **아예 열지 않는다.** `dbnet` 네트워크로만 접근한다.

> **규칙 5-3.** 인터넷에 노출되는 포트는 **80/443 뿐이며, Caddy만 점유**한다.

---

## 6. Caddy

Caddy는 `systemd`가 관리하는 **시스템 서비스**(`caddy.service`, `enabled`)다. 부팅 시
로그인 없이 자동 기동되며, macOS의 LaunchAgent/LaunchDaemon 구분 같은 건 없다 — 이미 가장
간단한 형태다.

```bash
sudo systemctl start|stop|restart|status caddy   # systemctl 은 deploy에게 NOPASSWD sudo 허용됨
```

- 설정 파일(실제): **`/etc/caddy/Caddyfile`**
- 사이트별 설정: `/etc/caddy/caddy-sites/*.caddy` (메인 파일이 `import`)
- 로그: `/var/log/caddy/` (caddy 유저 소유)
- 데몬 실행 계정: 시스템 유저 `caddy` (deploy가 아니다)

### 6-1. `/opt/homebrew` 경로 셔밈 — 왜 리눅스에 이 경로가 있는가

> **규칙 6-1.** `/opt/homebrew` 아래의 심볼릭 링크를 지우거나 옮기지 않는다.
> `deployctl`의 Caddy 자동화 전체가 이걸 통해서만 작동한다.

`deployctl`은 macOS(Homebrew) 전제로 짜여 있어 `CADDYFILE`·`CADDY_SITES` 상수가
`/opt/homebrew/etc/Caddyfile`, `/opt/homebrew/etc/caddy-sites`로 **하드코딩**돼 있다.
이 경로를 실제 위치로 옮기는 대신(§1 결정: 서버가 도구에 맞춘다), 그 자리에 진짜 설정을
가리키는 심볼릭 링크를 심어서 해결했다(2026-09-06):

```
/opt/homebrew/                 deploy:deploy 755 (deploy 소유 — 그 아래는 sudo 없이 자유)
├── etc → /etc/caddy           (디렉터리 심볼릭 링크: 그 밑의 파일 쓰기는 실제로 /etc/caddy 안에 반영됨)
└── var/log/caddy → /var/log/caddy
```

이 심볼릭 링크 덕분에 `deployctl`이 `/opt/homebrew/etc/Caddyfile`에 쓰면 실제로는
`/etc/caddy/Caddyfile`이 바뀐다. `caddy validate --config /opt/homebrew/etc/Caddyfile`도
같은 이유로 동작한다.

### 6-2. 권한 — sudo 없이 쓰기

> **규칙 6-2.** `deployctl`은 Caddy 설정을 **sudo 없이** 쓴다. 그러려면 `/etc/caddy`와
> `Caddyfile`이 `deploy`가 쓸 수 있는 그룹 소유여야 한다.

`/etc/caddy`(디렉터리, `deployctl`이 여기 임시파일을 만들어 원자적 교체함)와
`Caddyfile`을 `root:caddy`로, `deploy`를 `caddy` 그룹에 편입했다(2026-09-06):

```bash
sudo usermod -aG caddy deploy
sudo install -d -o root -g caddy -m 775 /etc/caddy
sudo install -d -o root -g caddy -m 775 /etc/caddy/caddy-sites
sudo install -o root -g caddy -m 664 <Caddyfile 사본> /etc/caddy/Caddyfile   # 내용 보존, 권한만 교체
```

- **이 권한은 되돌아갈 수 있다.** Caddyfile을 `root`로 직접 편집(예: 대화형 `sudo vi`)하면
  `install`/`vi`가 기본 권한(보통 `root:root 644`)으로 다시 써버려 `deployctl`의 쓰기가
  막힌다. 사람이 직접 고쳤다면 **위 `install` 명령으로 그룹·권한을 다시 맞춘다.**
- 볼륨을 밀거나 서버를 재설치하면 이 두 조치(usermod, install) 모두 다시 필요하다.
- 관리 API(`127.0.0.1:2019`)는 `deploy`에게 열려 있어 **reload에는 애초에 권한 문제가
  없다.** `caddy reload --config ...`로 사이트 추가·삭제를 무인 자동화할 수 있다.
- 사이트 추가 시 Caddyfile에 블록을 추가하고 `caddy reload`(관리 API)로 **무중단
  반영**한다. `sudo`는 필요 없다.
- 변경 전 `caddy validate --config /opt/homebrew/etc/Caddyfile`로 검증한다.
- 인증서 실험 시에는 `acme_ca` 스테이징 URL을 쓴다 (Let's Encrypt rate limit 회피).

### 6-3. 로그 파일 소유자 — validate가 만들어 둔 파일을 데몬이 못 여는 문제

> **규칙 6-3.** `/var/log/caddy`도 `deploy`가 쓸 수 있어야 한다 (그룹 `caddy`, 775).
> 그래도 사이트를 새로 추가할 때마다 로그 파일 소유자 문제가 **재발할 수 있다** — 아래를
> 알고 있을 것.

`caddy validate`는 config에 적힌 로그 writer를 실제로 열어서 검증한다. **파일이 없으면
그 자리에서 만든다.** 문제는 `validate`가 `deploy`로 도는 반면 Caddy 데몬 자신은 별도의
시스템 유저 `caddy`로 돈다는 것 — `validate`가 먼저 실행되면 `deploy:deploy 0600`(소유자
전용) 로그 파일이 생기고, 그 다음 `reload`가 데몬에게 같은 파일을 열라고 시키면 데몬이
**그 파일의 소유자가 아니라서** permission denied가 난다. 증상이 "Caddyfile 권한
문제"처럼 보이지만 실제로는 로그 **파일** 하나의 소유자 문제라 헷갈리기 쉽다.

`deployctl`은 `caddy_reload()`에서 `validate`와 `reload` 사이에 `fix_caddy_log_owner()`를
불러 `/var/log/caddy` 아래 `caddy` 소유가 아닌 파일을 찾아 `sudo install -o caddy -g caddy
-m 660`으로 되돌린다(`deployctl` 안의 `sudo` 사용은 이것과 §8의 러너 서비스뿐이다).
손으로 사이트를 추가하거나 이 로직이 없던 옛 버전으로 되돌아갔다면, `reload`가
`permission denied`로 실패할 때 `/var/log/caddy/<도메인>.log`의 소유자부터 확인한다.

이 문제는 애초에 로그 경로를 `/opt/homebrew` 셔밈이 아니라 **실제 경로**
(`/var/log/caddy`, `CADDY_LOG_DIR` 상수)로 직접 가리키게 한 것과는 별개다 — 심볼릭 링크
자체는 문제가 아니었다(Caddy는 `caddy.service`의 `ProtectSystem=full` 샌드박스 안에서
돌기 때문에, 데몬이 여는 경로는 그 데몬이 실제로 접근 권한을 가진 경로를 곧이곧대로
써주는 쪽이 안전하다 — 그래서 로그 경로만 셔밈을 거치지 않는다).

---

## 7. 디렉터리 구조

이 서버에는 코드가 놓이는 위치가 셋이며, **역할이 다르므로 섞지 않는다.**

| 경로 | 역할 | 소스 관리 | 편집 |
|---|---|---|---|
| `~/workspaces/<프로젝트>` | **모든 프로젝트의 작업 공간** | git clone | **여기서만 편집** |
| `~/srv/<사이트>` | 운영 배포물 (compose, `.env`) | git 미포함 | 배포 설정만 |
| `~/actions-runner/_work` | runner가 자동 체크아웃 | 자동 | **직접 건드리지 않는다** |

### 7-1. `~/workspaces` — 프로젝트 작업 공간

> **규칙 7-1.** 모든 프로젝트는 `~/workspaces/` 아래에 clone하고, **코드 수정은 여기서만** 한다.

```
~/workspaces/
├── initial-settings/        # 이 저장소 (인프라 규칙서 + deployctl)
└── myapp/
```

- 새 프로젝트는 `git clone`으로 이 폴더 아래에 받는다.
- `deployctl`도 여기서 고친다. `/usr/local/bin/deployctl`이 이 파일을 실행하는 한 줄짜리
  래퍼 스크립트라(`exec ~/workspaces/initial-settings/deployctl "$@"`) 저장하는 즉시
  반영된다. (macOS의 Homebrew였다면 심볼릭 링크로 됐겠지만, `/usr/local/bin`은 `root`
  소유라 `deploy`가 직접 심볼릭 링크를 못 걸어 래퍼로 대신했다 — 효과는 동일하다.)
  배포 워크플로는 경로 대신 명령 이름(`deployctl`)으로 부른다.
- `~/srv/`나 `~/actions-runner/_work/`의 파일을 직접 수정하지 않는다.
  전자는 배포 설정 전용이고, 후자는 배포 때마다 덮어써진다.

### 7-2. `~/srv` — 운영 배포물

```
~/srv/
├── postgres/            # 공유 DB 인스턴스
│   ├── compose.yaml
│   └── .env
└── <사이트명>/
    ├── compose.yaml (또는 app.json이 가리키는 파일)
    └── .env             # DB 접속 정보 등, git 미포함, chmod 600
```

앱 소스는 여기 두지 않는다. runner가 `~/workspaces`가 아닌 자체 workspace에서 빌드한 이미지를
이 위치의 compose가 실행하는 구조다.

사이트명은 소문자·하이픈만 사용한다. DB명/유저명은 사이트명에서 파생시킨다 (`foo` → DB `foo`, 유저 `foo_user`).

---

## 8. 배포 (GitHub Actions self-hosted runner)

> **규칙 8-1.** self-hosted runner는 **private 저장소에만** 연결한다.
> public 저장소에 붙이면 임의의 PR 코드가 이 서버에서 실행된다.

> **규칙 8-2.** 러너는 **저장소마다 하나씩** 붙인다. `deployctl runner <사이트>`로 등록한다.

`tinynoroo`는 **Organization이 아니라 개인(User) 계정**이라 조직 단위 러너를 쓸 수 없다.
GitHub의 조직 단위 러너는 Organization/Enterprise 계정에만 있다. 따라서 저장소가 늘 때마다
러너 인스턴스가 하나씩 늘어난다. 사이트가 많아져 부담이 되면 Organization을 만들어 저장소를
옮기는 쪽이 답이다.

- 러너는 **`~/actions-runner/<사이트>/`** 아래로 모은다. 홈에 흩어 놓지 않는다.
  ```
  ~/actions-runner/
  ├── actions-runner-linux-x64-<버전>.tar.gz   # 공용 배포본
  └── <사이트명>/
  ```
- 러너 이름은 `linux-<사이트>`, 라벨은 사이트명이다. 예외를 두지 않는다.
- 배포본 tarball은 위 공용 자리의 것을 재사용하고, 없으면 이 호스트(`linux-x64`)용 최신을
  내려받는다.
- 서비스는 `systemd`(시스템 유닛, `/etc/systemd/system/`)로 등록된다. **리눅스의
  `svc.sh install/start/stop/uninstall`은 자기 자신이 `sudo`로 실행되고 있는지부터
  검사한다** (macOS는 사용자 LaunchAgent라 필요 없었지만, 리눅스는 다르다). 처음엔
  `sudo svc.sh ...`로 되겠거니 했는데 실패했다 — **sudoers NOPASSWD 목록은 `systemctl`
  같은 개별 명령 단위로 걸려 있지, `svc.sh`라는 스크립트를 통째로 sudo 실행하는 건
  내부에서 그 명령들을 부르더라도 별개로 취급돼 일반 sudo(비밀번호 필요)로 떨어진다.**
  그래서 `deployctl`은 `svc.sh install`이 실제로 하는 일(유닛 파일 렌더링해 배치 →
  `daemon-reload` → `runsvc.sh` 복사 → `enable`)을 `install`/`systemctl`만으로 그대로
  재현한다 — sudoers를 넓히지 않고 이미 있는 권한만으로 끝낸다
  (`linux_service_install/start/stop/uninstall`, `deployctl:612` 부근).
- 디렉터리를 옮기면 서비스 등록은 따라오지 않는다. `deployctl runner <사이트>`가
  등록이 살아 있는지 보고 서비스만 다시 건다.
- 워크플로의 `runs-on`에 **사이트명 라벨**을 넣어 러너를 특정한다.
  ```yaml
  runs-on: [self-hosted, Linux, X64, <사이트>]
  ```
  라벨이 겹치는 러너가 여러 대면 아무 데서나 돌 수 있다. 쓰지 않는 러너는 반드시 해제한다.
- 러너 `.env`에 PATH를 명시한다. 서비스로 뜬 러너는 로그인 셸 환경을 상속하지 않아
  이게 없으면 `docker: command not found`가 난다.
  ```
  PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
  ```
- 배포 워크플로에는 **docker 데몬 대기 단계**를 넣는다. 재부팅 직후 runner가 docker보다
  먼저 뜰 수 있다.
  ```yaml
  - run: for i in $(seq 30); do docker info >/dev/null 2>&1 && exit 0; sleep 2; done; exit 1
  ```
- 배포 후 **헬스체크**를 하고, 실패 시 이전 이미지로 롤백한다. `deployctl deploy`가
  빌드 직전 이미지 ID를 기억했다가 HTTPS 확인이 실패하면 태그를 되돌리고 다시 띄운다.
  **되돌아가는 것은 이미지뿐이다 — 마이그레이션은 그대로 남는다.** 앞으로만 가는
  마이그레이션(컬럼 추가 등)은 구 이미지가 견디지만, 파괴적 변경은 롤백해도 깨진다.
  첫 배포는 되돌릴 이미지가 없으므로 롤백하지 않고 실패로 끝난다. 끄려면 `--no-rollback`.

**`myapp` 편입 완료 (2026-09-07)**: 처음엔 위 규칙이 자리잡기 전에 손으로
설정돼 있었지만(러너가 `~/actions-runner/` 바로 아래, 자체 워크플로로 `rsync`+
`sudo docker compose` 직접 실행), 지금은 규칙대로 편입됐다:
- `~/srv/myapp/app.json` 매니페스트 생성, `.env`도 `~/workspaces/`에서
  이쪽으로 옮김 (규칙 7-2).
- 러너를 `~/actions-runner/myapp/`에 `linux-myapp`이라는
  이름으로 새로 등록 → 온라인 확인 → 예전 러너(`deploy-myapp`, 옛 위치)는
  서비스 정지·GitHub 등록 해제·디렉터리 정리까지 끝냄. 한 사이트에 러너 하나만 남는다.
- 워크플로의 `Sync source to deploy directory`/`Deploy`/`Health check` 세 단계를
  `deployctl deploy myapp --source "$GITHUB_WORKSPACE"` 한 줄로 교체.
  배포 전 테스트 게이트(`Run tests`)는 `deployctl`이 모르는 단계라 손으로 그대로 남겼다 —
  워크플로를 다시 만들 때 `deployctl workflow --force`를 쓰면 이 단계가 날아가니 주의.
- 실제 push → 새 러너 → `deployctl deploy` → Caddy 등록 → HTTPS 200까지 끝까지 검증됨.
  이 과정에서 §6-3(로그 파일 소유자)과 러너 서비스 sudo 문제(§8 위)를 실제로 겪고 고쳤다.

---

## 9. 자동 시작 / 재부팅

Docker(`docker.service`)·Caddy(`caddy.service`)·러너(`actions.runner.*.service`) 모두
**시스템 단위 `systemd` 서비스**(`enabled`)다. macOS의 사용자 LaunchAgent와 달리 **로그인
세션이나 자동 로그인 없이도 부팅 직후 그대로 뜬다** — 이 부분은 리눅스가 원래 더 단순하다.

```bash
systemctl is-enabled docker caddy   # 둘 다 enabled 여야 함
```

- Docker를 `sudo`로 띄우거나 데이터 디렉터리를 손으로 옮기지 않는다. 표준 `apt` 설치
  그대로 `/var/lib/docker`를 쓴다.

---

## 10. 작업 시 지켜야 할 절차

1. **파괴적 작업 전 확인** — 볼륨 삭제, `docker system prune --volumes`, DB DROP은 실행 전 사용자에게 확인받는다.
   특히 `prune --volumes`는 서비스가 내려간 상태에서 실행하면 DB 데이터가 삭제된다.
2. **사이트 추가는 `deployctl`로 한다** (§11). 위 규칙 대부분을 기계가 검사·수행한다.
   ```bash
   deployctl init                                    # 준비물 안내 후 항목을 하나씩 물어본다
   deployctl init <사이트> --domain <도메인> --db     # 인자를 다 주면 묻지 않는다 (CI용)
   deployctl validate <사이트>                        # 규칙 위반 검사
   deployctl deploy   <사이트>                        # 기동 + Caddy 등록 + HTTPS 확인
   ```
   손으로 할 때만 아래를 직접 확인한다.
   - [ ] 이미지 아키텍처 확인 (이 서버는 amd64)
   - [ ] DB + 유저 생성, `REVOKE ALL ON DATABASE ... FROM PUBLIC` 수행
   - [ ] 타 사이트 DB 접근 차단 검증 (§4-2)
   - [ ] 커넥션 풀 설정, `max_connections` 여유 확인
   - [ ] 포트 `127.0.0.1:` 바인딩 확인
   - [ ] Caddy 사이트 파일 추가 → `caddy validate` → reload
   - [ ] HTTPS 응답 및 인증서 확인
3. **변경 후 이 문서를 갱신한다.** 설치 버전, 사이트 목록, 규칙 변경 사항이 실제와 어긋나면 안 된다.

---

## 11. deployctl — 배포 도구

`deployctl` (PATH에 있음 — `/usr/local/bin/deployctl`, `~/workspaces/initial-settings/deployctl`을
그대로 실행하는 한 줄 래퍼. §7-1 참고).
위 규칙을 사람이 외우는 대신 기계가 검사·수행한다. 표준 라이브러리만 쓴다.
Caddy·DB 작업엔 **sudo가 필요 없다**(Caddy reload는 관리 API로, 파일 쓰기 권한은 §6-2의
그룹 소유로 미리 확보해 뒀다) — 단 로그 파일 소유자를 되돌리는 한 곳(§6-3)과 러너 서비스
등록(§8)만 리눅스에서 예외적으로 `sudo`를 쓴다. 둘 다 sudoers NOPASSWD 화이트리스트
(`install`·`systemctl` 등, 2절)에 있는 명령만 그대로 호출해서 무인으로 된다 — 목록에
없는 명령(`svc.sh` 같은 스크립트 자체)은 sudo로 못 부르니, 그 명령이 실제로 하는 일을
화이트리스트 명령만으로 재현하는 쪽으로 푼다(§8의 `linux_service_*` 함수들이 그 예).

| 명령 | 하는 일 |
|---|---|
| `init` | 매니페스트 생성, 포트 자동 배정(9000–9099), `~/workspaces/<사이트>`로 clone, 전용 DB·유저 생성 + 격리 검증, `.env`에 `DATABASE_URL` 기록(600) |
| `validate` | 사이트명·포트·도메인 중복, 커넥션 여유, DNS, 이미지 아키텍처(호스트와 일치), `127.0.0.1` 바인딩, **게시 포트 == `app.json` 포트**, 전용 Postgres 금지, `dbnet` 참여, **빌드 정의 없는 이미지의 존재 여부** |
| `deploy` | 배포 대상(커밋·브랜치·미커밋) 표시 → 검사 → 빌드 → 기동 → Caddy 등록·reload → HTTPS 폴링 → 실패 시 롤백 |
| `status` | 컨테이너·Caddy 등록·HTTPS·커넥션 현황 |
| `scaffold` | **없는** `Dockerfile`·compose 생성 (Dockerfile은 `claude -p` 초안 → `docker build` 검증, 실패 시 오류 되먹여 재시도) |
| `workflow` | `.github/workflows/deploy.yml` 생성 (docker 대기 → checkout → deploy, `runs-on`에 사이트 라벨). **이미 있으면 `--force` 없이는 덮어쓰지 않는다** |
| `runner` | 저장소 전용 Actions 러너 등록 (private 확인 → 배포본 전개 → 토큰 발급 → systemd 서비스 기동), `--remove`로 해제 |
| `remove` | 컨테이너 정리 + Caddy 사이트 해제 (DB와 매니페스트는 남긴다) |

`init`은 **사이트명과 `--domain`이 둘 다 주어졌을 때만** 조용히 실행된다.
하나라도 빠지면 준비물(private 저장소·A 레코드·Dockerfile+compose.yaml)을 먼저 안내하고
항목을 하나씩 물은 뒤, 요약을 보여 주고 확인을 받는다. 확인 전에는 아무것도 만들지 않는다.
비대화형 환경(러너 등)에서 인자가 빠지면 묻지 않고 오류를 낸다.

첫 항목은 **`gh`로 읽은 저장소 목록**이다. 번호로 고르면 저장소 URL이 정해지고
그 이름이 사이트명 기본값이 된다. 목록에서 빼는 것:

- 이 서버에 이미 등록된 저장소 (`app.json`의 `repository` 또는 사이트명이 겹치는 것).
  컨테이너가 떠 있는지는 보지 않는다 — 기준은 매니페스트 존재 여부다.
- **public 저장소** — 러너에 붙이지 않는다 (규칙 8-1)

각 줄에 루트의 `Dockerfile`·compose 파일 유무를 함께 띄워 배포 준비 여부를 보여 주고,
준비된 저장소를 위로 올린다. `gh`가 없거나 로그인이 안 돼 있으면 URL을 직접 받는다.
`--repo`를 주면 목록을 건너뛴다. **도메인은 목록과 무관하게 직접 입력한다.**

확인을 받으면 아래를 이어서 한다. 사이트 하나를 올리는 데 필요한 것이 `init` 하나로 끝난다.

1. `~/workspaces/<사이트>`로 clone (규칙 7-1, 이미 있으면 건너뜀)
2. `Dockerfile`·compose가 없으면 만들지 물어본다 (§11-4)
3. `.github/workflows/deploy.yml` 생성. 같은 디렉터리의 다른 self-hosted 워크플로가 있으면 알린다
4. 전용 러너 등록 (규칙 8-2)
5. 전용 DB·유저 생성 + `.env`에 `DATABASE_URL` 기록
6. 배포. 러너가 있으면 **커밋·push해서 러너가 하게 하고**(자동 배포 사슬을 한 번에 검증),
   실행을 지켜본다. 잡이 뜨지 않으면 전달 실패로 보고 이 자리에서 배포한다.
   잡이 뜬 뒤 단계에서 죽으면 내용 실패이므로 물러서지 않는다 — 로컬에서 해도 같기 때문이다.
   `--no-push`면 처음부터 이 자리에서 배포한다.

러너 등록이 실패해도 나머지는 진행하고 `deployctl runner <사이트>`로 다시 하라고 안내한다.
비대화형에서는 `--runner`·`--deploy`를 붙여야 각 단계를 한다.

**`init`은 전부 되거나 아무것도 안 된다.** clone 직후 `Dockerfile`과 compose가 있는지 보고,
없으면(스캐폴딩을 거절·실패했으면) **매니페스트도 DB도 러너도 Caddy 등록도 만들지 않고** 끝낸다.
그 뒤 단계에서 실패하면 **이번 실행이 만든 것만** 되돌린다 — 컨테이너·Caddy 사이트 파일·DB·
매니페스트 순으로 지우고, 이미 있던 DB·러너·클론은 건드리지 않는다.
소스와 생성된 초안 파일은 남긴다(고쳐서 다시 시도할 근거가 필요하다).
되돌리기가 실패하면 어느 항목인지 이름을 찍는다.
github 저장소는 `gh repo clone`으로 받는다 — 계정에 설정된 프로토콜(ssh)을 그대로 쓰기 위해서다.
`git clone`에 https URL을 주면 비대화형에서 인증에 막힌다. 받지 않으려면 `--no-clone`.

`--db`로 DB를 만들면 접속 문자열을 `~/srv/<사이트>/.env`의 `DATABASE_URL`에 **직접 기록한다**.
이미 그 키가 있으면 덮어쓰지 않고 새 값을 화면에 출력만 한다.

### 11-1. 단일 진실 원천

사이트 정의는 `~/srv/<사이트>/app.json` 하나다. 포트·도메인·풀 크기를 바꾸려면 이 파일을 고치고
`deploy`를 다시 돌린다. Caddy 사이트 파일은 여기서 생성되므로 **직접 고치지 않는다.**

```json
{ "name": "foo", "domain": "foo.example.com", "port": 9040,
  "healthPath": "/", "composeFile": "source:compose.yaml", "poolSize": 5 }
```

- `composeFile`이 `source:<파일>`이면 러너 체크아웃(`--source`) 기준, 아니면 `~/srv/<사이트>/` 기준이다.
  `--source`를 생략하면 `~/workspaces/<사이트>`를 쓴다. 러너 워크플로는 지금처럼 `--source`를 명시한다.
- 앱 소스 경로는 매니페스트에 두지 않는다. 배포 기준은 **항상 러너 체크아웃 하나**다.

### 11-2. Caddy 파일 구조

`deployctl` 코드가 보는 경로는 `/opt/homebrew/etc/...`이지만, §6-1의 심볼릭 링크를 통해
실제로는 이렇게 반영된다:

```
/opt/homebrew/etc/Caddyfile          # → /etc/caddy/Caddyfile (전역 설정 + import 한 줄만)
/opt/homebrew/etc/caddy-sites/       # → /etc/caddy/caddy-sites/ (도메인당 파일 하나, deployctl 생성)
```

메인 Caddyfile에 사이트 블록을 직접 적지 않는다. 같은 도메인이 양쪽에 있으면 Caddy가
`ambiguous site definition`으로 거부한다. 이미 적혀 있으면 `deploy`가 사이트 파일로 옮기고 백업을 남긴다.

### 11-3. 포트

호스트 Caddy가 `127.0.0.1:<포트>`로 프록시하므로 **사이트마다 포트가 하나씩 필요**하다
(traefik과 달리 컨테이너에 직접 라우팅하지 않는다). `init`이 9000번대에서 비어 있는
포트를 배정하고, `validate`가 중복을 막는다. 손으로 정하지 않는다.

### 11-4. 스캐폴딩 — 없는 것만 만든다

> **규칙 11-4.** 이미 있는 `Dockerfile`·compose는 **덮어쓰지 않는다.** `--force`를 줄 때만 예외.

사람이 쓴 파일에는 생성기가 모르는 판단이 들어 있다. 규칙 위반이 있어도 `validate`가
**짚어만 주고** 고치지 않는다.

| 상태 | 하는 일 |
|---|---|
| 둘 다 있음 | 아무것도 만들지 않음 |
| 둘 다 없음 | Dockerfile 생성 → 그 `EXPOSE`로 compose 생성 |
| Dockerfile만 있음 | compose만 생성 |
| compose만 있음 | Dockerfile만 생성 |

**Dockerfile은 `claude -p`로 만들고 `docker build`로 검증한다.** 빌드가 실패하면 오류를
프롬프트에 되먹여 다시 만든다 (기본 3회). 통과한 초안만 파일로 남는다. Claude에는
`Read,Glob,Grep`만 준다 — 파일은 deployctl이 쓴다. 프롬프트에 규칙을 박는다:
호스트 아키텍처(이 서버는 amd64)에 맞는 멀티아치 베이스(규칙 3-1), `--platform` 금지,
non-root 실행, 시크릿 미포함, `EXPOSE` 명시.

**배포 경로에는 넣지 않는다.** 러너 워크플로는 `claude` 없이 돌아야 하고, 배포는 재현
가능해야 한다. 비대화형에서는 `--scaffold`를 명시할 때만 동작한다.

compose는 LLM 없이 템플릿으로 찍는다. 사이트 `.env`에 `DATABASE_URL`이 있을 때만
필수 변수로 박는다 — 없는데 박으면 기동 전에 죽는다.

**생성물은 초안이다.** 검수하고 커밋하는 것은 사람이 한다.
