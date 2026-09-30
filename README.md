# docker-images

REPLWorks 프로젝트들이 공통으로 쓰는 Docker 베이스 이미지를 관리하는 레포입니다.
이미지는 GitHub Actions로 빌드해서 GHCR(`ghcr.io/replworks/...`)에 올립니다.

## 구조

```text
docker-images/
├── wifinote-runtime/
│   └── Dockerfile
├── .github/workflows/build.yml
└── README.md
```

이미지 하나당 디렉터리 하나입니다. 디렉터리 이름이 이미지 이름입니다.

## 이미지 목록

| 이미지                               | 내용                                               | 사용처                          |
| ------------------------------------ | -------------------------------------------------- | ------------------------------- |
| `ghcr.io/replworks/wifinote-runtime` | PHP 8.4 + Apache, PHP 확장(intl, gd, redis 등), uv | wifinote 앱 Dockerfile의 `FROM` |

앱 코드, vendor, 시크릿(Nova 자격증명 등)은 베이스 이미지에 넣지 않습니다. 앱 레포의 Dockerfile에서만 다룹니다.

## 태그 규칙

한 번 빌드한 결과물에 태그 두 개가 붙습니다. 내용은 같습니다.

| 태그         | 성격                                           | 용도                           |
| ------------ | ---------------------------------------------- | ------------------------------ |
| `php8.4`     | 빌드할 때마다 최신으로 덮어씀                  | 평소 앱 Dockerfile이 읽는 태그 |
| `php8.4-<N>` | 고정, 덮어쓰지 않음 (`N` = 워크플로 실행 번호) | 롤백용                         |

`N`은 `github.run_number`입니다. 이 워크플로의 실행 횟수이고, 실패한 실행과 수동/자동 실행도 모두 번호를 소비하므로 중간에 번호가 비어 있을 수 있습니다.

## 빌드가 돌아가는 시점

- `wifinote-runtime/**` 파일이 main에 push될 때
- 매월 1일 03:00 UTC (업스트림 PHP 이미지의 보안 패치 반영)
- Actions 탭에서 수동 실행 (**Actions → Build images → Run workflow**)

x86 러너에서 QEMU로 arm64를 빌드하므로 확장을 컴파일하는 빌드는 20분 안팎 걸립니다. 앱 릴리스 빌드에서는 이 컴파일이 없습니다.

## 앱에서 사용

앱 레포의 Dockerfile에서 베이스로 지정합니다.

```dockerfile
FROM ghcr.io/replworks/wifinote-runtime:php8.4 AS app
```

앱 이미지를 빌드하려면 베이스 이미지가 먼저 GHCR에 올라가 있어야 합니다.

## 첫 세팅 체크리스트

1. 이 레포에 Dockerfile과 워크플로를 push하고 Actions에서 빌드 성공 확인
2. GitHub 프로필/조직의 **Packages** 탭에서 `wifinote-runtime`과 태그 두 개 확인
3. 패키지 **Package settings**에서 가시성 결정
    - Public: 베이스에는 시크릿이 없으므로 가장 단순, 별도 설정 없이 pull 가능
    - Private: **Manage Actions access**에 앱 레포(wifinote)를 추가해야 pull 가능. 빠지면 `denied` 에러

## 롤백

월 1회 자동 재빌드나 Dockerfile 수정으로 새 베이스가 올라간 뒤 앱이 깨졌을 때의 절차입니다.

### 1. 정상이던 번호 찾기

- **Packages → wifinote-runtime**의 태그 목록에서 `php8.4-<N>` 확인
- 어느 번호가 정상이었는지는 Actions 탭의 실행 목록(`#N`)과 커밋 이력으로 판단

### 2-A. 앱 Dockerfile의 FROM을 고정 (기본 방법)

앱 레포에서 `FROM`을 정상이던 번호로 바꾸고 다시 릴리스/배포합니다.

```dockerfile
FROM ghcr.io/replworks/wifinote-runtime:php8.4-3 AS app
```

원인을 고친 새 베이스를 올린 뒤 `php8.4`로 되돌립니다.

### 2-B. `php8.4` 태그 자체를 옛 버전으로 되돌리기

앱 Dockerfile을 건드리지 않고 움직이는 태그를 옛 이미지에 다시 붙입니다. 재빌드 없이 태그만 옮깁니다.

```bash
echo $GHCR_TOKEN | docker login ghcr.io -u <깃헙아이디> --password-stdin
docker buildx imagetools create \
  -t ghcr.io/replworks/wifinote-runtime:php8.4 \
  ghcr.io/replworks/wifinote-runtime:php8.4-3
```

`GHCR_TOKEN`은 `write:packages` 권한이 있는 Personal Access Token입니다.

### 3. 앱 이미지 다시 빌드/배포

롤백해도 이미 배포된 앱 이미지는 옛 베이스로 바뀌지 않습니다. 앱 레포의 워크플로를 **Run workflow**로 실행하고 `release_tag`에 배포할 릴리스 태그를 넣어 다시 빌드/배포합니다.

### 주의

- 옛 실행을 Actions에서 **Re-run** 하는 것은 롤백이 아닙니다. 그 시점이 아니라 현재 레포 상태로 다시 빌드되고, 같은 번호 태그를 덮어씁니다.
- 롤백에 쓸 `php8.4-<N>` 태그를 지우지 마세요. `FROM`이 특정 번호를 참조 중이면 그 태그를 지우는 순간 앱 빌드가 실패합니다.

## 정리 (삭제)

GHCR은 오래된 태그를 자동으로 지우지 않습니다. 당분간은 그냥 둬도 됩니다 (레이어는 공유되어 실제 용량 증가는 작음).
필요해지면 Packages 탭에서 오래된 버전을 수동으로 삭제하되, 롤백용으로 최근 몇 개는 남겨 두세요.

## 이름 규칙

디렉터리 이름 = 이미지 이름입니다.

**원칙: 앱 레포 하나당 베이스 이미지는 최대 1개이고, 릴리스 빌드에서 빼내야 할 때만 만듭니다.**

- 빼내는 기준: 거의 안 바뀌면서 오래 걸리는 단계(PHP 확장 컴파일 등)가 릴리스 빌드를 느리게 할 때. 그런 이유가 없으면 이미지를 만들지 않고 앱 Dockerfile 하나로 둡니다.
- 이름: `<앱 레포 이름>-runtime` (예: 앱 레포 `wifinote` → `wifinote-runtime`)
    - 앱 이미지는 `ghcr.io/replworks/<앱>`, 베이스는 `ghcr.io/replworks/<앱>-runtime`으로 짝이 맞고, 둘이 같은 패키지에 섞이지 않습니다.
- 이미지를 더 쪼개지 않습니다. 빌드용/실행용 분리는 앱 Dockerfile의 멀티스테이지로 처리합니다.
- 태그는 `<스택><버전>` 형식 (예: `php8.4`, `node22`)과 고정용 `<스택><버전>-<run_number>`

## 이미지 추가

1. 새 디렉터리 생성 (이름 규칙 참고, 예: `myapp-runtime/Dockerfile`)
2. `build.yml`의 `on.push.paths`에 `"laravel-apache/**"` 추가
3. `jobs` 아래에 `wifinote-runtime` 잡과 같은 형태의 잡을 추가하고 `context`와 `tags`를 새 이미지 이름으로 수정
4. 첫 빌드 후 패키지 가시성 확인 (가시성은 이미지별로 따로 설정됨)
