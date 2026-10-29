# mattermost-selfhost

Docker Compose로 [Mattermost](https://mattermost.com) 채팅 서버를 직접 운영하고, [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)로 외부에 HTTPS로 공개하는 구성입니다.
공유기 포트포워딩이나 인증서 발급 없이 내 도메인(예: `https://chat.example.com`)으로 접속할 수 있습니다.

## 구성

```
사용자 → https://chat.example.com → Cloudflare → cloudflared → mattermost:8065 → postgres:5432
```

| 서비스 | 이미지 | 역할 |
|---|---|---|
| `postgres` | `postgres:16` | Mattermost 데이터베이스 |
| `mattermost` | `mattermost/mattermost-enterprise-edition:latest` | 채팅 서버 (호스트 `8065` 포트) |
| `cloudflared` | `cloudflare/cloudflared:latest` | Cloudflare Tunnel 클라이언트 |

데이터(DB, 설정, 업로드 파일, 로그, 플러그인)는 Docker named volume에 저장되어 컨테이너를 다시 만들어도 유지됩니다.

## 준비물

- Docker 및 Docker Compose v2 (`docker compose version`으로 확인)
- Cloudflare 계정과 Cloudflare에 연결된 도메인

## 설치

### 1. 저장소 받기

```bash
git clone https://github.com/ssuntol916/mattermost-selfhost.git
cd mattermost-selfhost
```

### 2. Cloudflare Tunnel 만들기

1. [Cloudflare Zero Trust 대시보드](https://one.dash.cloudflare.com) → **Networks → Tunnels → Create a tunnel**
2. 커넥터 유형은 **Cloudflared**를 선택하고 이름을 정합니다.
3. 설치 안내 화면의 명령어에 있는 토큰(`eyJ...`로 시작하는 긴 문자열)을 복사해 둡니다. 직접 설치할 필요는 없습니다.
4. **Public Hostname**을 추가합니다.
   - Subdomain / Domain: 사용할 주소 (예: `chat` / `example.com`)
   - Service: Type `HTTP`, URL `mattermost:8065`

> URL의 `mattermost`는 `compose.yml`의 서비스 이름입니다. cloudflared 컨테이너가 같은 Docker 네트워크 안에서 이 이름으로 Mattermost에 접속합니다.

### 3. 환경 변수 설정

```bash
cp .env.example .env
```

`.env`를 열어 값을 채웁니다.

| 변수 | 설명 |
|---|---|
| `TUNNEL_TOKEN` | 2단계에서 복사한 터널 토큰 |
| `POSTGRES_PASSWORD` | DB 비밀번호. 접속 URL에 들어가므로 `@` `/` `:` 문자는 쓰지 마세요. |
| `SITE_URL` | 2단계에서 정한 외부 접속 주소 (예: `https://chat.example.com`) |

랜덤 비밀번호가 필요하면 `openssl rand -hex 24`를 사용하세요.

> `.env`는 `.gitignore`에 포함되어 있어 커밋되지 않습니다. 절대 공개 저장소에 올리지 마세요.

### 4. 실행

```bash
docker compose up -d
docker compose ps        # 세 컨테이너가 모두 running인지 확인
```

처음 실행하면 DB 초기화 때문에 Mattermost가 뜨기까지 1분 정도 걸릴 수 있습니다.

### 5. 초기 설정

`SITE_URL`(또는 같은 컴퓨터에서 `http://localhost:8065`)에 접속하면 관리자 계정 생성 화면이 나옵니다.
**처음 만드는 계정이 시스템 관리자가 되므로**, 외부에 공개되기 전에 바로 만들어 두세요.

## 운영

```bash
docker compose logs -f mattermost   # 로그 보기
docker compose restart mattermost   # 재시작
docker compose down                 # 중지 (데이터 유지)
docker compose pull && docker compose up -d   # 이미지 업데이트
```

### 백업

```bash
# DB
docker exec mattermost-postgres pg_dump -U mmuser -d mattermost > backup-$(date +%F).sql

# 업로드 파일 (mattermost-data 볼륨)
docker run --rm -v "$(basename "$PWD")_mattermost-data:/data" -v "$PWD:/backup" alpine \
  tar czf /backup/mattermost-data-$(date +%F).tar.gz -C /data .
```

### DB 비밀번호 변경

Postgres는 **처음 초기화할 때만** `POSTGRES_PASSWORD`를 읽습니다. 설치 후 `.env`만 바꾸면 Mattermost가 DB에 접속하지 못하니 아래 순서로 바꾸세요.

```bash
docker exec -it mattermost-postgres psql -U mmuser -d mattermost \
  -c "ALTER USER mmuser PASSWORD '새비밀번호';"
# .env의 POSTGRES_PASSWORD를 같은 값으로 수정한 뒤
docker compose up -d
```

## 참고

- **라이선스**: Enterprise Edition 이미지는 라이선스 없이도 무료 기능으로 동작합니다. 순수 오픈소스판을 원하면 `compose.yml`의 이미지를 `mattermost/mattermost-team-edition`으로 바꾸세요.
- **버전 고정**: `:latest` 태그는 업데이트 시 메이저 버전이 바뀔 수 있습니다. 운영 환경에서는 `mattermost/mattermost-enterprise-edition:10.x.x`처럼 버전을 고정하는 것을 권장합니다.
- **포트 노출**: Tunnel로만 접속한다면 `compose.yml`의 `"8065:8065"`를 `"127.0.0.1:8065:8065"`로 바꾸거나 지워서 외부 노출을 줄일 수 있습니다.
- **문제 해결**: 접속이 안 되면 `docker compose logs cloudflared`에서 터널 연결 여부를, `docker compose logs mattermost`에서 DB 연결 오류를 확인하세요. Cloudflare 대시보드에서 터널 상태가 **HEALTHY**인지도 확인하세요.
