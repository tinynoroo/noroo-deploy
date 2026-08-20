# Mac 서버 구축 가이드 (Caddy + Docker + GitHub Actions)

Apple Silicon Mac(arm64, macOS 26.5)을 상시 가동 웹 서버로 운영하기 위한 셋업 문서.

**구성 요약**

```
인터넷 ──► 공유기(80/443 포트포워딩) ──► Mac
                                          ├─ Caddy (호스트, 443/80)  ← TLS 종료 + 리버스 프록시
                                          ├─ Docker (앱 컨테이너, 127.0.0.1:3000 등)
                                          └─ GitHub Actions self-hosted runner (배포 실행)
```

| 항목 | 선택 |
|---|---|
| 컨테이너 런타임 | colima + docker CLI (GUI 앱 없음) |
| 리버스 프록시 | Caddy (Homebrew, 호스트에서 실행) |
| TLS | Let's Encrypt 자동 발급/갱신 (Caddy 내장) |
| 배포 | GitHub Actions **self-hosted runner** (아웃바운드 전용, 인바운드 SSH 불필요) |

> 이 문서의 `example.com`, `myapp`, `youruser` 는 실제 값으로 치환할 것.

---

## 0. 사전 준비 체크리스트

- [ ] 고정 사설 IP (공유기 DHCP 예약)
- [ ] 공유기에서 80/443 포트포워딩
- [ ] 도메인 소유 및 DNS A 레코드 설정 권한
- [ ] **ISP가 80/443 인바운드를 막지 않는지 확인** — 국내 가정용 회선은 차단되는 경우가 있다.
      막혀 있으면 이 구성 전체가 성립하지 않으므로 **가장 먼저** 검증할 것 (§0-1)
- [ ] 공인 IP가 유동이면 DDNS 준비

### 0-1. 80/443 개방 여부 먼저 검증

포트포워딩까지 끝낸 뒤 인증서 발급 단계에서 막히면 원인 파악이 어렵다. 셋업 전에 확인한다.

```bash
# Mac에서 임시 서버를 띄우고
sudo python3 -m http.server 80
```

그 상태로 **휴대폰 모바일 데이터(Wi-Fi 끄기)** 에서 `http://<공인 IP>` 접속.
- 열리면 → 진행
- 안 열리면 → 공유기 포트포워딩 재확인 → 그래도 안 되면 ISP 차단

ISP 차단이 확인되면 통신사에 문의해 해제하거나, 80/443 개방이 가능한 회선/VPS로 옮겨야 한다.
비표준 포트(8080/8443)로 우회할 수는 있으나 **Let's Encrypt HTTP-01 챌린지는 80번을 요구**하므로
그 경우 DNS-01 챌린지(§4-7)로 전환해야 한다.

---

## 1. macOS 서버화 기본 설정

### 1-1. 절전 끄기 / 정전 후 자동 부팅

```bash
sudo pmset -a sleep 0            # 시스템 잠자기 안 함
sudo pmset -a disksleep 0        # 디스크 잠자기 안 함
sudo pmset -a displaysleep 10    # 화면만 10분 후 꺼짐 (전력 절감)
sudo pmset -a autorestart 1      # 정전 복구 시 자동 부팅
sudo pmset -a womp 1             # Wake on LAN
sudo pmset -a powernap 0
sudo pmset -a standby 0 hibernatemode 0

pmset -g custom                  # 확인
```

노트북(MacBook)이면 뚜껑을 닫아도 안 자도록 전원 연결 상태에서 위 설정 + 클램셸 모드 사용.

### 1-2. 자동 로그인 (재부팅 후 무인 복구용)

colima(`brew services`)와 Actions runner(`svc.sh`)는 둘 다 **사용자 LaunchAgent**라
로그인 세션에서만 로드된다.
`시스템 설정 → 사용자 및 그룹 → 자동 로그인`에서 서버 계정 자동 로그인 활성화.

> 트레이드오프: 물리적 접근자가 곧바로 데스크톱에 접근 가능. 디스크는 FileVault로 보호되지만
> 자동 로그인을 켜면 부팅 시 자동 잠금 해제되므로 물리 보안이 확보된 장소에만 둘 것.
> 물리 보안이 불안하면 자동 로그인을 끄고, 재부팅 후 SSH로 접속해
> `colima start`와 `~/actions-runner/svc.sh start`를 수동 실행하는 운용도 가능하다.

### 1-3. 원격 접속

```bash
sudo systemsetup -setremotelogin on     # SSH 활성화
```

- 관리 접속은 **Tailscale** 권장 (SSH 포트를 인터넷에 열지 않음)
  ```bash
  brew install --cask tailscale
  ```
- 화면 공유가 필요하면 `시스템 설정 → 일반 → 공유 → 화면 공유` (역시 Tailscale 경유로만)

### 1-4. 방화벽

```bash
# 애플리케이션 방화벽 켜기 (Caddy가 요청하면 수신 허용)
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
```

### 1-5. 호스트명

```bash
sudo scutil --set ComputerName  mac-server
sudo scutil --set HostName      mac-server
sudo scutil --set LocalHostName mac-server
```

---

## 2. Docker 설치 — colima + docker CLI

GUI 앱 없이 CLI만으로 구성한다. Docker Desktop / OrbStack 같은 데스크톱 앱은 첫 실행 마법사와
로그인 세션을 요구하고, Docker Desktop은 상업적 사용 시 유료 라이선스가 필요해 제외했다.

```bash
brew install colima docker docker-compose
```

### 2-1. compose 플러그인 경로 등록

Homebrew의 `docker-compose`는 별도 경로에 설치되어 docker CLI가 자동으로 못 찾는다.
`~/.docker/config.json`에 등록해야 `docker compose` (하이픈 없는 형태)가 동작한다.

```json
{
  "cliPluginsExtraDirs": [
    "/opt/homebrew/lib/docker/cli-plugins"
  ]
}
```

> Docker Desktop/OrbStack을 쓰다가 넘어왔다면 이 파일에 `"currentContext": "orbstack"` 같은
> 잔재가 남아 있을 수 있다. 없는 컨텍스트를 가리키면 docker CLI가 데몬을 못 찾으므로 삭제할 것.

### 2-2. VM 기동

```bash
colima start --cpu 4 --memory 8 --disk 100 --vm-type vz --vz-rosetta
```

- `--vm-type vz`: Apple Virtualization.framework 사용 (QEMU보다 빠름)
- `--vz-rosetta`: Rosetta로 amd64 이미지도 실행 가능 (`linux/amd64`, `linux/386` 지원 추가)
- 리소스 값은 맥 사양에 맞게 조정. 변경하려면 `colima stop && colima start --cpu N ...`

확인:
```bash
docker version --format '{{.Server.Version}} ({{.Server.Os}}/{{.Server.Arch}})'
docker compose version
docker run --rm hello-world
```

### 2-3. 부팅 시 자동 시작

```bash
brew services start colima
```

> **주의: 이것은 `~/Library/LaunchAgents/`에 등록되는 사용자 LaunchAgent다.**
> GUI 앱은 필요 없어졌지만 **로그인 세션은 여전히 필요**하므로, 재부팅 후 무인 복구를 하려면
> §1-2의 자동 로그인이 켜져 있어야 한다. colima를 `sudo`로 띄우면 VM 상태와 이미지가
> root 홈으로 분리되어 사용자 계정의 docker CLI와 어긋나므로 권장하지 않는다.

### 이미지 아키텍처 주의

Mac은 arm64다. CI에서 빌드해 배포한다면 **반드시 `linux/arm64`로 빌드**하거나 멀티아치 이미지를 쓴다.
(self-hosted runner가 이 Mac에서 직접 빌드하면 자연히 arm64가 되므로 문제 없음.)

---

## 3. 앱 배치 구조

배포 대상 디렉터리를 하나 정해두고 GitHub Actions가 여기를 갱신한다.

```bash
mkdir -p ~/srv/myapp
```

```
~/srv/myapp/
├── docker-compose.yml
├── .env                 # 시크릿 (git 미포함, 수동 배치)
└── (앱 소스는 runner의 workspace에서 빌드)
```

**`~/srv/myapp/docker-compose.yml` 예시**

```yaml
services:
  app:
    image: myapp:latest
    restart: unless-stopped
    env_file: .env
    ports:
      - "127.0.0.1:3000:3000"   # ★ 루프백에만 바인딩 — 외부 노출은 Caddy만 담당
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  db:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    # ports를 열지 않음 → 컨테이너 네트워크 내부에서만 접근

volumes:
  pgdata:
```

> 핵심 규칙: **컨테이너 포트는 `127.0.0.1:`로만 바인딩**한다. `- "3000:3000"`으로 쓰면
> 0.0.0.0에 열려 Caddy의 TLS를 우회한 평문 접근이 가능해진다.

---

## 4. Caddy 설치 및 설정

### 4-1. 설치

```bash
brew install caddy
```

### 4-2. DNS 설정

도메인 관리 콘솔에서 A 레코드를 공인 IP로 지정.

```
A    example.com        →  <공인 IP>
A    www.example.com    →  <공인 IP>
```

확인:
```bash
dig +short example.com
curl -s https://api.ipify.org      # 현재 공인 IP
```

유동 IP라면 DDNS 사용 (DNS 제공자 API + cron, 또는 공유기 내장 DDNS 기능).

### 4-3. 포트포워딩

공유기 관리 페이지에서 외부 80 → Mac 사설 IP:80, 외부 443 → Mac 사설 IP:443.
**80번도 반드시 열어야 한다** — Let's Encrypt HTTP-01 챌린지와 HTTP→HTTPS 리다이렉트에 필요.

### 4-4. Caddyfile

`/opt/homebrew/etc/Caddyfile`:

```caddyfile
{
	email you@example.com          # Let's Encrypt 계정 (만료 알림 수신)
	# 테스트 중에는 아래 주석을 풀어 스테이징 CA로 rate limit 회피
	# acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}

example.com, www.example.com {
	encode zstd gzip

	reverse_proxy 127.0.0.1:3000 {
		health_uri /health
		health_interval 10s
	}

	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "strict-origin-when-cross-origin"
		-Server
	}

	log {
		output file /opt/homebrew/var/log/caddy/example.com.log {
			roll_size 50MiB
			roll_keep 10
		}
		format json
	}
}

# 서브도메인으로 다른 컨테이너 붙이는 예
# api.example.com {
# 	reverse_proxy 127.0.0.1:4000
# }
```

문법 검증 및 포맷:
```bash
caddy fmt --overwrite /opt/homebrew/etc/Caddyfile
caddy validate --config /opt/homebrew/etc/Caddyfile
```

### 4-5. 실행 (80/443 바인딩 때문에 root 필요)

```bash
mkdir -p /opt/homebrew/var/log/caddy
sudo brew services start caddy      # ★ sudo 필수: 1024 미만 포트 바인딩
sudo brew services list
```

> `sudo` 없이 `brew services start caddy`를 하면 LaunchAgent로 등록되어 **로그인해야만 뜬다.**
> 반드시 `sudo`(LaunchDaemon)로 띄워 로그인과 무관하게 부팅 시 기동되게 할 것.
> (흔한 오해: macOS는 1024 미만 포트를 root로 제한하지 않아 포트 권한 때문은 아니다.)
> 인증서/상태 저장 경로는 root 실행 여부와 무관하게 `/opt/homebrew/var/lib/caddy`다.
> Homebrew의 plist가 `HOME`/`XDG_DATA_HOME`을 `/opt/homebrew/var/lib`로 덮어쓰기 때문
> (Caddy 기본 경로인 `~/Library/Application Support/Caddy`가 아니다). 백업 대상에 포함할 것.

설정 변경 후 무중단 리로드:
```bash
sudo brew services reload caddy
# 또는
sudo caddy reload --config /opt/homebrew/etc/Caddyfile
```

### 4-6. 검증

```bash
curl -I http://example.com          # 308 → https 리다이렉트 확인
curl -I https://example.com         # 200 + HSTS 헤더 확인
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates
```

인증서 발급이 실패하면 로그부터:
```bash
sudo tail -f /opt/homebrew/var/log/caddy.log
```

### 4-7. (예외) 80번을 못 여는 경우 — DNS-01 챌린지

§0-1에서 ISP가 80번을 막은 것으로 확인됐을 때만 해당한다. 정상적으로 80/443이 열렸다면 건너뛴다.

DNS-01은 HTTP 접근 없이 DNS TXT 레코드로 도메인 소유를 증명하므로 80번이 필요 없다.
다만 **Homebrew의 기본 caddy 바이너리에는 DNS 제공자 플러그인이 포함돼 있지 않아** 직접 빌드해야 한다.

```bash
brew install xcaddy go
xcaddy build --with github.com/caddy-dns/cloudflare   # 제공자에 맞는 모듈로 교체
sudo mv ./caddy /opt/homebrew/bin/caddy
```

> 사용 중인 DNS 제공자용 모듈은 https://github.com/caddy-dns 에서 확인.
> 직접 빌드한 바이너리는 `brew upgrade caddy` 시 덮어써지므로 업그레이드 후 재빌드가 필요하다.

Caddyfile:

```caddyfile
example.com {
	tls {
		dns cloudflare {env.CF_API_TOKEN}
	}
	reverse_proxy 127.0.0.1:3000
}
```

API 토큰은 해당 존의 **DNS 편집 권한만** 부여하고 LaunchDaemon 환경변수로 주입한다.

주의: 이 경우 인증서는 받을 수 있어도 **443번이 함께 막혀 있으면 외부 접속 자체가 불가능**하다.
80만 막히고 443은 열린 회선에서만 유효한 우회책이다.

---

## 5. GitHub Actions Self-hosted Runner

GitHub-hosted 러너에서 SSH로 들어오는 방식과 달리, 러너가 GitHub에 **아웃바운드로 롱폴링**하므로
인바운드 포트/SSH 키 노출이 없다. 가정용 NAT 환경에 가장 적합.

### 5-1. 보안 전제

> **self-hosted runner는 public 저장소에 절대 붙이지 말 것.** 누구나 PR로 임의 코드를
> 이 Mac에서 실행시킬 수 있다. private 저장소에서만 사용하고, 필요하면
> `Settings → Actions → General → Fork pull request workflows`를 비활성화한다.

### 5-2. 설치

GitHub 저장소 → `Settings → Actions → Runners → New self-hosted runner → macOS / ARM64`
페이지에 표시되는 토큰을 사용해:

```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner-osx-arm64.tar.gz -L \
  https://github.com/actions/runner/releases/latest/download/actions-runner-osx-arm64-<VERSION>.tar.gz
tar xzf actions-runner-osx-arm64.tar.gz

./config.sh --url https://github.com/<OWNER>/<REPO> \
            --token <REGISTRATION_TOKEN> \
            --name mac-server \
            --labels self-hosted,macos,arm64,prod \
            --work _work \
            --unattended
```

> 최신 버전 번호는 https://github.com/actions/runner/releases 에서 확인.

### 5-3. 서비스 등록 (부팅 시 자동 시작)

```bash
./svc.sh install       # ~/Library/LaunchAgents/ 에 LaunchAgent 등록
./svc.sh start
./svc.sh status
```

LaunchAgent이므로 **로그인 세션에서 동작**한다. §1-2 자동 로그인이 켜져 있어야 재부팅 후 자동 복구된다.

### 5-4. PATH 문제 (자주 걸림)

LaunchAgent는 로그인 셸 환경을 상속하지 않아 `docker`, `brew`를 못 찾는 경우가 많다.
`~/actions-runner/.env` 파일에 명시:

```bash
cat > ~/actions-runner/.env <<'EOF'
PATH=/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
EOF
./svc.sh stop && ./svc.sh start
```

colima 구성에서는 `/opt/homebrew/bin/docker`이므로 위 PATH로 충분하다.
추가로 runner가 colima 소켓에 붙으려면 colima가 먼저 기동돼 있어야 한다.

---

## 6. 배포 워크플로

`.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-prod
  cancel-in-progress: false   # 배포는 중간에 끊지 않는다

jobs:
  deploy:
    runs-on: [self-hosted, macos, arm64, prod]
    timeout-minutes: 20

    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build \
            --tag myapp:${{ github.sha }} \
            --tag myapp:latest \
            .

      - name: Deploy
        working-directory: /Users/youruser/srv/myapp
        run: |
          docker compose up -d --remove-orphans
          docker compose ps

      - name: Health check
        run: |
          for i in $(seq 1 30); do
            if curl -fsS http://127.0.0.1:3000/health; then
              echo "healthy"; exit 0
            fi
            sleep 2
          done
          echo "::error::health check failed"
          docker compose -f /Users/youruser/srv/myapp/docker-compose.yml logs --tail=100
          exit 1

      - name: Rollback on failure
        if: failure()
        working-directory: /Users/youruser/srv/myapp
        run: |
          docker tag myapp:previous myapp:latest 2>/dev/null || true
          docker compose up -d

      - name: Tag current as previous
        if: success()
        run: docker tag myapp:latest myapp:previous

      - name: Prune old images
        if: always()
        run: docker image prune -af --filter "until=168h"
```

### 시크릿 처리

- 앱 시크릿은 `~/srv/myapp/.env`에 **수동 배치**하고 git/워크플로에서 다루지 않는다 (파일 권한 `chmod 600`).
- CI에서 주입해야 하면 `Settings → Secrets and variables → Actions`에 등록 후 `${{ secrets.X }}`로 참조.
- self-hosted runner의 `_work` 디렉터리에는 체크아웃된 소스가 남으므로, 저장소에 평문 시크릿을 커밋하지 말 것.

---

## 7. 운영

### 로그

```bash
sudo tail -f /opt/homebrew/var/log/caddy.log            # Caddy 프로세스 로그
tail -f /opt/homebrew/var/log/caddy/example.com.log     # 액세스 로그(JSON)
docker compose -f ~/srv/myapp/docker-compose.yml logs -f app
tail -f ~/actions-runner/_diag/Runner_*.log             # runner 진단 로그
```

### 상태 점검 스크립트

`~/bin/serverctl` 로 저장하고 `chmod +x`:

```bash
#!/usr/bin/env bash
set -euo pipefail
echo "== power =="   ; pmset -g custom | grep -E 'sleep|autorestart'
echo "== caddy =="   ; sudo brew services list | grep caddy
echo "== docker =="  ; docker compose -f ~/srv/myapp/docker-compose.yml ps
echo "== runner ==" ; ~/actions-runner/svc.sh status | head -5
echo "== tls =="     ; echo | openssl s_client -connect example.com:443 \
  -servername example.com 2>/dev/null | openssl x509 -noout -enddate
echo "== disk =="    ; df -h / | tail -1
```

### 정기 정리 (디스크 확보)

```bash
# crontab -e
0 4 * * 0  /opt/homebrew/bin/docker system prune -af --volumes --filter "until=336h"
```

> `--volumes`는 **미사용 볼륨을 삭제**한다. DB 볼륨이 compose로 계속 사용 중이면 안전하지만,
> 서비스를 내려둔 상태에서 실행하면 데이터가 날아간다. 불안하면 `--volumes`를 빼고 쓸 것.

### 백업

- Docker 볼륨:
  ```bash
  docker run --rm -v myapp_pgdata:/data -v ~/backups:/backup alpine \
    tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .
  ```
- Postgres 논리 백업:
  ```bash
  docker compose exec -T db pg_dump -U postgres myapp | gzip > ~/backups/myapp-$(date +%F).sql.gz
  ```
- Time Machine은 외장 디스크에 별도 구성. `~/srv`, `/opt/homebrew/etc/Caddyfile`,
  `/opt/homebrew/var/lib/caddy`(발급받은 인증서 — 재설치 시 Let's Encrypt rate limit 회피),
  `~/actions-runner/.env`는 반드시 포함.
- 백업은 **복원 테스트를 한 번 해봐야** 백업이다.

### 업데이트

```bash
brew update && brew upgrade caddy
sudo brew services restart caddy

cd ~/actions-runner && ./svc.sh stop
# 새 러너 tar 교체 후
./svc.sh start
```

---

## 8. 트러블슈팅

| 증상 | 원인 / 조치 |
|---|---|
| Caddy가 80/443 바인딩 실패 | `sudo` 없이 실행함. `brew services stop caddy` 후 `sudo brew services start caddy` |
| 인증서 발급 실패 | 80번 포트포워딩 미설정, DNS 전파 미완료, ISP의 80 차단. `dig`/외부망에서 `curl -I http://example.com` 확인 (§0-1). ISP 차단이면 §4-7 |
| Let's Encrypt rate limit | 같은 도메인 주당 5회 실패 시 차단. Caddyfile에 `acme_ca` 스테이징 URL로 먼저 테스트 |
| 외부에서만 접속 불가(내부는 됨) | 공유기 헤어핀 NAT 문제이거나 포트포워딩 미적용. 모바일 데이터로 검증 |
| runner에서 `docker: command not found` | §5-4의 `.env` PATH 설정 |
| 재부팅 후 배포 실패 | 자동 로그인 꺼짐 → colima/runner LaunchAgent 미로드. §1-2 확인 |
| 이미지 아키텍처 오류 (`exec format error`) | amd64 이미지를 arm64에서 실행. `--platform linux/arm64`로 빌드하거나 멀티아치 이미지 사용 |
| `Cannot connect to the Docker daemon` | colima 미기동. `colima status` → `colima start`. `~/.docker/config.json`의 `currentContext` 잔재도 확인 |
| 디스크 가득참 | `docker system df` 확인 후 prune. 액세스 로그 roll 설정 확인 |

---

## 진행 체크리스트

- [ ] §0-1 80번 외부 접속 검증 (모바일 데이터) — **여기서 막히면 나머지가 무의미**
- [ ] §1 pmset / 자동 로그인 / SSH / Tailscale
- [ ] §2 colima 설치 + compose 플러그인 경로 + `brew services start colima`
- [ ] §3 `~/srv/myapp/docker-compose.yml` + `.env` 배치, 컨테이너 기동 확인
- [ ] §4 DNS A 레코드 → 포트포워딩 → Caddyfile → `sudo brew services start caddy` → HTTPS 확인
- [ ] §5 runner 등록 + `svc.sh install/start` + `.env` PATH
- [ ] §6 `deploy.yml` 커밋 후 push하여 배포 성공 확인
- [ ] §7 백업 스크립트 + 복원 1회 테스트
