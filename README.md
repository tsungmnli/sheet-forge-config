# sheet-forge-config

[sheet-forge](https://sheet.tsungmn.com)를 Kubernetes에 배포하기 위한 매니페스트입니다. [Kustomize](https://kustomize.io)로 구성했고 [Argo CD](https://argo-cd.readthedocs.io)가 이 저장소를 보고 배포합니다(GitOps).

sheet-forge는 수학 시험지 사진에서 문항을 골라 한글(hwpx) 문제지로 내보내는 서비스입니다.

| 저장소 | 역할 |
| --- | --- |
| [sheet-forge-api](https://github.com/tsungmnli/sheet-forge-api) | API 서버, 작업자 |
| [sheet-forge-web](https://github.com/tsungmnli/sheet-forge-web) | 웹 화면 |
| sheet-forge-config (이 저장소) | 배포 설정 |

## 무엇이 배포되나요

한 호스트에서 `/`는 웹, `/api`는 API로 나뉩니다. Ingress가 `/api` 접두어를 떼고 API로 넘깁니다.

| 리소스 | 설명 |
| --- | --- |
| `api-deployment`, Service, Ingress | API 서버(2개). `/healthz`, `/readyz` 프로브 |
| `worker-deployment` | 사진 보정·문제 칸 나누기·문제 읽기·정기 정리를 하는 작업자(1개). 없으면 사진이 처리되지 않습니다 |
| `sheet-forge-migrate` Job | 배포할 때마다 DB 마이그레이션(`alembic upgrade head`)을 먼저 실행 |
| `web-deployment`, Service, Ingress | 웹 화면 |
| ConfigMap `api-config`, Secret `api-secret` | 설정과 비밀 값 |

```
base/api/       API, 작업자, 마이그레이션의 공통 정의
base/web/       웹의 공통 정의
overlays/api/   API에 적용할 값: 도메인, 설정, 비밀, 이미지 태그
overlays/web/   웹에 적용할 값: 도메인, 이미지 태그
```

## 준비물

- Kubernetes 클러스터(1.27 이상 권장), ingress-nginx, cert-manager(`ClusterIssuer` 하나)
- Argo CD. 비밀 값을 Vault에서 가져오려면 `argocd-vault-plugin`도 필요합니다(아래 참고)
- PostgreSQL (15 이상)
- S3 호환 저장소(MinIO 등). 사진과 그림이 저장됩니다
- 메일 발송용 SMTP(예: Resend). 가입 초대와 비밀번호 재설정 메일에 씁니다
- 외부 LLM API 키(기본 모델은 Gemini 계열). 사진 인식에 씁니다
- 도메인 하나

## 내 환경에 맞춰 설정하기

바꿀 곳은 모두 `overlays/` 아래입니다.

| 무엇을 | 어디서 |
| --- | --- |
| 도메인 | `overlays/api/domain.json`, `overlays/web/domain.json`의 `value` (두 파일 모두 같은 값) |
| 네임스페이스 | 두 overlay의 `kustomization.yaml`의 `namespace` (api와 web은 같아야 인증서를 같이 씁니다) |
| 인증서 발급기 이름 | `base/api/ingress.yaml`, `base/web/ingress.yaml`의 `cert-manager.io/cluster-issuer` |
| 이미지 | 두 overlay의 `kustomization.yaml`의 `images` (`newName`, `newTag`). `placeholder`를 실제 태그로 바꿔야 합니다 |
| 일반 설정 | `overlays/api/config.yaml` (웹 주소, 메일 발신자, 저장소 버킷 이름, 인식 엔진 등) |
| 비밀 값 | `overlays/api/secret.yaml` |

### 비밀 값

`overlays/api/secret.yaml`은 아래 값을 Vault(`kv/sheet-forge-tsungmn`)에서 가져오도록 되어 있습니다. 하나라도 없으면 Argo CD 렌더링이 실패합니다.

| 키 | 내용 |
| --- | --- |
| `DATABASE_URL` | `postgresql://사용자:비밀번호@호스트:5432/DB이름` (비밀번호의 특수문자는 URL 인코딩) |
| `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` | 저장소 접근 키 |
| `LLM_API_KEY` | 외부 LLM API 키 |
| `SMTP_PASSWORD` | 메일 서비스 비밀번호(Resend라면 API 키) |

Vault를 쓰지 않는다면 `secret.yaml`을 일반 `Secret`(또는 Sealed Secrets 등)으로 바꾸고, `argocd-vault-plugin` 설정은 건너뛰면 됩니다. 키 이름은 `SHEET_FORGE_` 접두어가 붙은 그대로 유지하세요.

### 주요 설정 값 (`overlays/api/config.yaml`)

| 키 | 설명 |
| --- | --- |
| `SHEET_FORGE_WEB_BASE_URL` | 서비스 주소. 메일 속 링크의 앞부분 |
| `SHEET_FORGE_AUTH_REQUIRED` | `"1"`이면 로그인한 사람만 사용(운영에서는 항상 켜 두세요) |
| `SHEET_FORGE_RECOGNITION_ENGINE` | `real`(실제 인식, LLM 키 필요) 또는 `fake`(고정된 가짜 결과, 시험용) |
| `SHEET_FORGE_MINIO_ENDPOINT` | 저장소 `호스트:포트` (S3 API 포트, 보통 9000) |
| `SHEET_FORGE_MINIO_BUCKET` | 저장소 버킷 이름 |
| `SHEET_FORGE_SMTP_*` | 메일 서버 주소·포트·사용자·보내는 사람 |

전체 환경 변수는 [sheet-forge-api](https://github.com/tsungmnli/sheet-forge-api)의 README를 보세요.

## 배포하기

1. **데이터베이스**를 만듭니다. 서비스 전용 사용자와 DB를 쓰는 것을 권합니다.
   ```sql
   CREATE ROLE sheet_forge_app LOGIN PASSWORD '<비밀번호>';
   CREATE DATABASE sheet_forge OWNER sheet_forge_app;
   ```
   클러스터에서 DB 서버로 접속할 수 있어야 합니다(`pg_hba.conf` 등).
2. **저장소 버킷**을 만들고(`sheet-forge`), 그 버킷에만 접근할 수 있는 전용 키를 발급합니다.
3. **메일 서비스**에서 보내는 도메인을 인증합니다. `config.yaml`의 `SHEET_FORGE_SMTP_FROM`이 그 도메인 주소여야 메일이 갑니다.
4. 위 **비밀 값**을 Vault에 넣습니다.
5. 이미지를 준비합니다. 기본은 `tsungmn/sheet-forge-api`, `tsungmn/sheet-forge-web`이며, 직접 빌드했다면 `images`를 바꿉니다. `newTag`가 `placeholder`인 채로 배포하면 이미지를 받지 못합니다.
6. 이 저장소를 Argo CD에 **Application 두 개**로 등록합니다. Auto-Create Namespace를 켜고 Destination namespace를 `namespace`와 같게 적습니다.

   | | sheet-forge-api | sheet-forge-web |
   | --- | --- | --- |
   | Path | `overlays/api` | `overlays/web` |
   | Plugin | `argocd-vault-plugin-kustomize` | 없음 |

7. 도메인(DNS)을 Ingress로 연결합니다.

## 첫 사용자 초대

가입은 초대로만 됩니다. 배포가 끝나면 실행 중인 API 파드에서 초대 메일을 보냅니다.

```bash
kubectl -n <네임스페이스> exec deploy/api-deployment -- python scripts/invite.py 선생님@example.com
```

메일이 가지 않으면 같은 명령의 출력(로그)에 가입 링크가 남습니다. 비밀번호 재설정 메일을 못 받는 사람은 `scripts/reset_password.py`로 직접 바꿔 줄 수 있습니다.

## 잘 배포됐는지 확인하기

1. `kubectl -n <네임스페이스> get job,pods` — 마이그레이션 Job이 `Complete`, API 2개와 작업자 1개가 `Running`
2. `https://<도메인>/api/readyz`가 200 (`db`, `storage` 모두 `true`)
3. 초대 링크로 가입 → 로그인 → 사진 올리기 → 문제 칸이 나뉘면 작업자도 정상입니다

## 새 버전 배포와 되돌리기

- 새 버전은 `overlays/api`, `overlays/web`의 `newTag`를 바꾸고 커밋하면 Argo CD가 반영합니다. 마이그레이션 Job이 먼저 끝난 뒤 새 파드가 올라옵니다. Auto-Sync가 꺼져 있으면 Argo CD에서 직접 Sync를 누르세요.
- 되돌릴 때는 `newTag` 커밋을 `git revert`합니다. DB 마이그레이션은 자동으로 되돌리지 않습니다.
- `config.yaml`만 바꾸면 파드가 저절로 재시작되지 않습니다.
  ```bash
  kubectl -n <네임스페이스> rollout restart deploy/api-deployment deploy/worker-deployment
  ```
  인식 엔진이나 LLM 키를 바꿀 때도 같습니다.

## 문제가 생기면

| 증상 | 확인할 것 |
| --- | --- |
| 배포가 마이그레이션 Job에서 멈춤 | `kubectl logs job/sheet-forge-migrate`. 대부분 DB 접속 정보나 Vault 키 누락입니다. 고친 뒤 다시 Sync |
| 렌더링 오류 | Vault에 키 5개가 모두 있는지, KV 버전에 맞는 경로(`kv/data/...` 또는 `kv/...`)인지 |
| `/api/readyz`가 503 | 응답의 `checks`에서 `db` 또는 `storage`가 `false`인지 보고 접속 정보와 네트워크를 확인 |
| 사진이 처리 중에서 안 끝남 | 작업자 로그(`kubectl logs deploy/worker-deployment`). 작업자가 떠 있는지, LLM 키가 맞는지 |
| 사진 업로드가 실패 | Ingress의 `proxy-body-size`(기본 40m). 값을 올릴 때는 API의 `SHEET_FORGE_MAX_BODY_BYTES`도 함께 |
| 로그인 후에도 풀림 | 서비스를 `https`로 열었는지(세션 쿠키가 `Secure`입니다) |

## 알아 두세요

- 인식 엔진이 `real`이면 시험지 사진의 축소본과 문제 부분 이미지가 외부 LLM 서비스로 전송됩니다. 사용자에게 안내할 수 있도록 약관과 개인정보 안내를 준비하세요.
- 모든 컨테이너는 비root로 실행되고 루트 파일시스템이 읽기 전용입니다(웹 제외). 시작할 때 쓰기 오류가 나면 해당 컨테이너에 쓰기 가능한 볼륨이 필요한 것입니다.
