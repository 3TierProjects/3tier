# Spring PetClinic — ECS Fargate 컨테이너 배포 가이드
## Windows 11 + PowerShell 환경

> **이 문서는 AWS 실습 경험 3개월 미만인 학생이 Windows 11에서 그대로 따라 하면 됩니다.**
> Spring PetClinic을 컨테이너로 만들어 ECS Fargate에 배포하고, 오토스케일링과 무중단 배포까지 확인합니다.

---

## 0. 이 문서에서 하는 일

### 30초 요약

| 항목 | 내용 |
|------|------|
| **왜?** | EC2에 톰캣을 직접 설치·패치·관리하는 방식은 손이 많이 가고 확장이 느림 |
| **무엇을?** | 애플리케이션을 컨테이너 이미지로 만들어 서버리스 컨테이너 서비스(Fargate)에서 실행 |
| **어떻게?** | Dockerfile → ECR → ECS Fargate → ALB, 자격증명은 Secrets Manager로 주입 |
| **코드 수정?** | **없음** (Dockerfile만 새로 작성, 애플리케이션 소스는 그대로) |
| **총 소요시간** | 약 5~6시간 (환경 설치 1시간 + 실습 4~5시간) |
| **운영체제** | **Windows 11** |
| **터미널** | **PowerShell** (Windows Terminal 권장) |
| **리전** | `us-west-2` (오레곤) — 문서 전체에서 이 리전만 사용 |

---

### Before / After 비교

```
[ Before — EC2 직접 운영 ]
┌──────────────────────────────────────────┐
│  EC2 인스턴스                              │
│   ├ OS 패치를 내가 함                      │
│   ├ Tomcat 설치·설정을 내가 함             │
│   ├ WAR 배포를 내가 함                     │
│   ├ 서버가 죽으면 내가 살림                 │
│   └ DB 비밀번호가 설정파일에 평문으로 있음   │
│                                          │
│  확장하려면 → AMI 굽고 → ASG 만들고 → 대기  │
└──────────────────────────────────────────┘

[ After — ECS Fargate ]
┌──────────────────────────────────────────┐
│  Fargate 태스크                            │
│   ├ OS 없음 (AWS가 관리)                   │
│   ├ Tomcat이 이미지 안에 포함               │
│   ├ 태스크가 죽으면 ECS가 자동 재시작        │
│   └ DB 비밀번호는 Secrets Manager에서 주입  │
│                                          │
│  확장하려면 → 숫자만 바꿈 (또는 자동)        │
└──────────────────────────────────────────┘
```

---

### 비유로 이해하기

이 실습의 모든 개념을 **음식 배달 프랜차이즈**에 1:1로 대응시킵니다.

```
Dockerfile        =  레시피 (무엇을 어떤 순서로 넣는지)
컨테이너 이미지     =  밀키트 (조리 직전 상태로 포장된 것)
ECR               =  밀키트 창고 (버전별로 보관)
태스크 정의        =  주방 매뉴얼 (밀키트 몇 개, 불 세기 얼마)
태스크            =  실제로 조리 중인 한 그릇
서비스            =  "항상 2그릇은 대기시켜라"는 점장 지시
Fargate           =  주방을 통째로 빌려 쓰는 것 (설비 관리 안 함)
ALB               =  손님을 빈 자리로 안내하는 웨이터
Secrets Manager   =  금고 (비법 소스 레시피 보관)
태스크 실행 역할    =  금고 열쇠를 가진 직원
오토스케일링       =  손님이 몰리면 조리대를 늘리는 규칙
롤링 업데이트      =  새 메뉴로 바꾸되 손님이 끊기지 않게 하나씩 교체
VPC 엔드포인트     =  창고로 가는 전용 통로 (큰길로 안 나감)
CodePipeline      =  레시피가 바뀌면 자동으로 밀키트를 다시 만드는 공장
Docker Desktop    =  내 컴퓨터 안의 작은 조리실 (WSL2라는 리눅스 방에 있음)
```

이 표를 계속 곁에 두고 읽으세요.

---

## 1. 전체 흐름 (그림)

```
                   [ Windows 11 + PowerShell ]
                              │
          ① Dockerfile 작성 · 이미지 빌드 (Docker Desktop)
                              │
                              ▼
                        ┌───────────┐
                   ② push │   ECR     │  밀키트 창고
                        └───────────┘
                              │
                              │ ③ 태스크가 이미지를 pull
                              ▼
┌────────────────────────────────────────────────────────┐
│  VPC  10.0.0.0/16                                      │
│                                                        │
│  ┌──────────────── Public Subnet ─────────────────┐    │
│  │        ⑤ ALB  ←─── 사용자 HTTP 요청            │    │
│  └────────────────────┬───────────────────────────┘    │
│                       │                                │
│  ┌──────────────── Private Subnet (App) ──────────┐    │
│  │   ④ Fargate 태스크  ← ⑦ 오토스케일링으로 증감    │    │
│  │      [task]  [task]                            │    │
│  └────────┬──────────────────┬────────────────────┘    │
│           │                  │                         │
│           │ ⑥ 시크릿 주입    │                         │
│           ▼                  ▼                         │
│  ┌─── VPC Endpoint ───┐   ┌──── Private Subnet (DB) ─┐ │
│  │ ECR / Secrets /    │   │      RDS MariaDB         │ │
│  │ CloudWatch Logs    │   └──────────────────────────┘ │
│  └────────────────────┘                                │
└────────────────────────────────────────────────────────┘
                              │
          ⑧ 코드 변경 시 CodePipeline이 ①②를 자동 수행
                              │
          ⑨ 새 이미지 태그로 롤링 업데이트 (무중단)
```

**번호 순서가 곧 실습 순서입니다.**

---

## 2. 핵심 개념 9가지 (꼭 읽고 넘어가기)

> 이 9가지를 모르면 에러가 났을 때 **어디가 문제인지 찾을 수 없습니다.**

---

### 개념 1 — Docker Desktop과 WSL2 (Windows에만 있는 개념)

**Docker는 원래 리눅스 기술입니다.** Windows에는 리눅스 커널이 없으므로 그대로 실행할 수 없습니다.

Windows 11은 **WSL2(Windows Subsystem for Linux 2)** 라는 경량 가상머신 안에 진짜 리눅스 커널을 돌립니다. Docker Desktop은 이 WSL2 안에서 Docker 엔진을 실행하고, PowerShell의 `docker` 명령이 그 엔진에게 명령을 전달합니다.

```
[ PowerShell ]  docker build ...
       │
       │ (명령 전달)
       ▼
[ Docker Desktop ]  ── 관리 ──▶  [ WSL2 안의 리눅스 커널 ]
                                        │
                                        ▼
                                  실제 컨테이너 실행
```

**그래서 이런 일이 벌어집니다:**

| 현상 | 이유 |
|------|------|
| Docker Desktop을 끄면 `docker` 명령이 안 됨 | 엔진이 죽었기 때문 |
| 컴퓨터를 재부팅하면 매번 Docker Desktop을 켜야 함 | 자동 시작이 꺼져 있으면 |
| 메모리를 많이 먹음 | WSL2 VM이 1.2~2GB를 상시 점유 |

Docker Desktop VM은 약 1.2–2GB RAM을 사용하며, 엔진 프로세스(200–400MB)를 더하면 컨테이너 실행 전에 이미 1.5–2.5GB를 씁니다. 그래서 8GB가 실용적인 최소치입니다.

- **이 프로젝트에서 쓰는 실제 이름**: Docker Desktop (WSL2 백엔드)

---

### 개념 2 — 컨테이너 이미지 (밀키트)

애플리케이션 + 실행에 필요한 모든 것(JRE, 라이브러리)을 하나로 묶어 **어디서 실행해도 똑같이 동작하게** 만든 파일 묶음입니다.

- **이 프로젝트에서 쓰는 실제 이름**: `petclinic:v1`, `petclinic:v2`
- **만드는 곳**: Windows PC (Docker Desktop이 WSL2에서 빌드)

---

### 개념 3 — ECR (밀키트 창고)

AWS가 제공하는 컨테이너 이미지 저장소입니다.

- **이 프로젝트에서 쓰는 실제 이름**: `petclinic`
- **콘솔 위치**: **AWS 콘솔 → ECR (Elastic Container Registry) → 리포지토리**
- **주소 형식**: `<계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic`

---

### 개념 4 — 태스크 정의 (주방 매뉴얼)

컨테이너를 **어떻게 실행할지 적어 둔 설계도**입니다.

**중요:** 태스크 정의는 **한 번 만들면 수정할 수 없습니다.** 바꾸려면 새 리비전(revision)을 만듭니다. `petclinic-task:1`, `petclinic-task:2` 처럼 번호가 붙습니다.

- **이 프로젝트에서 쓰는 실제 이름**: `petclinic-task`
- **콘솔 위치**: **AWS 콘솔 → ECS → 태스크 정의**

---

### 개념 5 — 태스크와 서비스 (조리 중인 그릇 / 점장 지시)

**태스크(Task)** 는 실제로 실행 중인 컨테이너 한 벌입니다. 죽으면 그냥 사라집니다.

**서비스(Service)** 는 "태스크를 항상 N개 유지하라"는 규칙입니다.

```
서비스 (desired count = 2)
   ├─ 태스크 A  (실행 중)
   └─ 태스크 B  (실행 중)

태스크 B가 죽으면
   ├─ 태스크 A  (실행 중)
   └─ 태스크 C  ← 서비스가 자동 생성
```

- **이 프로젝트에서 쓰는 실제 이름**: 서비스 `petclinic-service`, 클러스터 `petclinic-cluster`

---

### 개념 6 — Fargate (통째로 빌린 주방)

EC2 인스턴스를 만들지 않고 컨테이너를 실행하는 방식입니다.

| 구분 | EC2 시작 유형 | **Fargate** |
|------|--------------|-------------|
| EC2 인스턴스 관리 | 내가 함 | **안 함** |
| OS 패치 | 내가 함 | **안 함** |
| 과금 단위 | 인스턴스 시간 | **태스크의 vCPU·메모리 사용 시간** |
| 이 실습 | ❌ | ✅ **이걸 씁니다** |

---

### 개념 7 — 태스크 실행 역할 vs 태스크 역할 (열쇠 두 개)

**이 둘의 혼동이 이 실습에서 가장 많이 터지는 오류입니다.**

```
┌─────────────────────────────────────────────────┐
│  태스크 실행 역할 (Task Execution Role)           │
│  = "ECS 에이전트"가 쓰는 열쇠                      │
│                                                 │
│  언제 쓰나: 태스크를 시작하기 "전"                  │
│  무엇을 하나:                                     │
│    · ECR에서 이미지를 pull                        │
│    · Secrets Manager에서 값을 읽어 환경변수로 주입  │
│    · CloudWatch Logs에 로그 그룹 생성              │
│                                                 │
│  이 프로젝트 이름: ecsTaskExecutionRole-petclinic │
└─────────────────────────────────────────────────┘
              vs
┌─────────────────────────────────────────────────┐
│  태스크 역할 (Task Role)                          │
│  = "애플리케이션 코드"가 쓰는 열쇠                  │
│                                                 │
│  언제 쓰나: 태스크가 실행되는 "동안"                │
│  무엇을 하나:                                     │
│    · 앱이 S3에 파일 업로드                         │
│    · 앱이 DynamoDB를 조회                         │
│                                                 │
│  이 실습에서는: 필요 없음 (앱이 AWS API를 안 씀)    │
└─────────────────────────────────────────────────┘
```

**기억법:** 시크릿을 못 읽어서 태스크가 **아예 안 뜨면** → 태스크 **실행** 역할 문제.
태스크는 떴는데 **앱 안에서** AWS 호출이 실패하면 → 태스크 역할 문제.

---

### 개념 8 — VPC 엔드포인트 (창고로 가는 전용 통로)

Fargate 태스크는 프라이빗 서브넷에 있습니다. 그런데 ECR에서 이미지를 받아야 하고, Secrets Manager를 읽어야 합니다.

| 방법 | 동작 | 비용 | 이 실습 |
|------|------|------|---------|
| NAT Gateway | 인터넷으로 나갔다가 돌아옴 | 시간당 + 데이터 처리량 | ❌ |
| **VPC 엔드포인트** | **AWS 내부망으로 직접 감** | 시간당 (NAT보다 저렴) | ✅ |

**이 실습에서 만들 엔드포인트 (총 5개):**

| 엔드포인트 | 유형 | 왜 필요한가 |
|-----------|------|------------|
| `com.amazonaws.us-west-2.ecr.api` | Interface | ECR 인증 |
| `com.amazonaws.us-west-2.ecr.dkr` | Interface | 이미지 레이어 pull |
| `com.amazonaws.us-west-2.secretsmanager` | Interface | 시크릿 조회 |
| `com.amazonaws.us-west-2.logs` | Interface | CloudWatch 로그 전송 |
| `com.amazonaws.us-west-2.s3` | **Gateway** | **ECR 이미지 레이어가 실제로는 S3에 저장됨** |

> ⚠️ **S3 Gateway 엔드포인트를 빼먹으면 이미지 pull이 실패합니다.**
> "ECR인데 왜 S3가 필요하지?"라고 생각하기 쉽지만, ECR은 메타데이터만 관리하고 실제 이미지 레이어는 S3에 저장합니다.

---

### 개념 9 — 오토스케일링

Fargate에서는 **태스크 개수만** 늘리면 됩니다.

```
CPU 사용률 50% 초과가 계속됨
         ↓
CloudWatch 경보 발생
         ↓
Application Auto Scaling이 desired count 증가
         ↓
서비스가 새 태스크를 띄움
         ↓
ALB가 새 태스크를 타깃 그룹에 자동 등록
```

- **이 프로젝트에서 쓰는 정책**: 대상 추적, 지표 `ECSServiceAverageCPUUtilization`, 목표값 `50`

---

### 개념 관계 한눈에 보기

```
클러스터 (petclinic-cluster)
   └─ 서비스 (petclinic-service)          ← "태스크 2개 유지해라"
        ├─ 태스크 정의 (petclinic-task:1)  ← "이 이미지를, 이 스펙으로"
        │     ├─ 이미지: ECR의 petclinic:v1
        │     ├─ 시크릿: Secrets Manager의 petclinic/db
        │     └─ 실행 역할: ecsTaskExecutionRole-petclinic
        ├─ 태스크 A (실행 중)  ──┐
        └─ 태스크 B (실행 중)  ──┼→ ALB 타깃 그룹에 자동 등록
                                │
        오토스케일링 ────────────┘  ← 부하에 따라 태스크 수 증감
```

---

## 3. 흔한 오해 정정

> **중요**: 아래 7가지 착각 중 하나라도 가지고 시작하면 **중간에 반드시 막힙니다.**
> ⓐ~ⓑ는 **Windows 전용** 함정입니다.

---

### 오해 ⓐ — "PowerShell은 리눅스 셸과 같다" (Windows 전용)

**전혀 다릅니다.** 인터넷에서 찾은 리눅스 명령을 그대로 붙여넣으면 대부분 실패합니다.

| 리눅스(bash) | **PowerShell** | 설명 |
|-------------|---------------|------|
| `export VAR=값` | `$env:VAR = "값"` | 환경변수 설정 |
| `$VAR` | `$env:VAR` | 환경변수 참조 |
| `echo $VAR` | `echo $env:VAR` | 출력 |
| `command \` (줄바꿈) | `` command ` `` (백틱) | 줄 연결 |
| `ls -l` | `ls` 또는 `Get-ChildItem` | 목록 |
| `cat file` | `cat file` 또는 `Get-Content` | 파일 읽기 |
| `grep 패턴` | `Select-String 패턴` | 검색 |
| `curl URL` | `curl.exe URL` ← **.exe 필수** | HTTP 요청 |
| `watch -n 5 명령` | (없음) → `while($true){...}` | 반복 실행 |
| `sed -i 's/a/b/' f` | (없음) → 아래 참조 | 문자열 치환 |

> ⚠️ **`curl` 은 반드시 `curl.exe` 로 쓰세요.**
> PowerShell에서 `curl` 은 `Invoke-WebRequest` 의 별칭(alias)입니다. 옵션 문법이 완전히 다릅니다.
> `curl -s -o /dev/null` 같은 리눅스 옵션이 전부 오류가 납니다.

> ⚠️ **줄 끝의 백슬래시(`\`)는 PowerShell에서 동작하지 않습니다.**
> 여러 줄 명령은 백틱(`` ` ``)을 쓰거나, **한 줄로 붙여 쓰세요.**
> 이 문서에서는 **혼란을 막기 위해 모든 AWS CLI 명령을 한 줄로 제공**합니다.

---

### 오해 ⓑ — "ECR 로그인 명령을 AWS 문서에서 복사하면 된다" (Windows 전용)

**PowerShell에서는 실패합니다.** AWS 공식 문서의 파이프 방식이 PowerShell에서 400 오류를 냅니다.

```powershell
# ❌ 이렇게 하면 실패 — 400 Bad Request
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-west-2.amazonaws.com
```

**원인:** 파이프 앞부분이 비밀번호에 개행 문자를 덧붙인 뒤 뒷부분으로 전달하기 때문입니다.

**해결:** 비밀번호를 변수에 담아 `--password` 로 넘깁니다.

```powershell
# ✅ 이렇게 하면 성공
$pw = aws ecr get-login-password --region us-west-2
docker login --username AWS --password $pw 123456789012.dkr.ecr.us-west-2.amazonaws.com
```

이 방식은 PowerShell과 Windows Terminal에서 네이티브로 동작합니다.

> 📌 `--password` 사용 시 "insecure" 경고가 뜨지만 실습에서는 무시해도 됩니다.

---

### 오해 ① — "Spring PetClinic은 Spring Boot다"

**절반만 맞습니다.** PetClinic에는 두 갈래가 있습니다.

| 저장소 | 정체 | 산출물 | 이 실습 |
|--------|------|--------|---------|
| `spring-projects/spring-petclinic` | **Spring Boot** (내장 톰캣) | 실행 가능 `.jar` | ✅ **이걸 씁니다** |
| `spring-petclinic/spring-framework-petclinic` | Spring MVC (외부 톰캣) | `.war` | ❌ |

**이 문서는 Spring Boot 버전을 기준으로 합니다.** 이미지 안에 톰캣을 따로 설치하지 않습니다.

---

### 오해 ② — "환경변수는 아무 이름이나 써도 된다"

Spring Boot는 정해진 이름의 환경변수만 인식합니다.

| 사용 가능 ✅ | 사용 불가 ❌ |
|-------------|-------------|
| `SPRING_DATASOURCE_URL` | `DB_URL` |
| `SPRING_DATASOURCE_USERNAME` | `DB_USER` |
| `SPRING_DATASOURCE_PASSWORD` | `DB_PASS` |
| `SPRING_PROFILES_ACTIVE` | `PROFILE` |

**근거:** Spring Boot는 `application.properties`의 키 `spring.datasource.url`을 환경변수 `SPRING_DATASOURCE_URL`로 자동 매핑합니다. 점(`.`)이 밑줄(`_`)로, 소문자가 대문자로 바뀌는 규칙입니다.

---

### 오해 ③ — "프라이빗 서브넷에 태스크를 두면 인터넷이 필요 없다"

**태스크 자체는 인터넷이 필요 없지만, 태스크를 "시작하는" ECS 에이전트는 ECR·Secrets Manager에 접근해야 합니다.**

```
CannotPullContainerError: ... i/o timeout
```

이 실습은 NAT 대신 **VPC 엔드포인트 5개**로 해결합니다. (개념 8 참조)

---

### 오해 ④ — "보안 그룹 이름에 sg- 를 붙인다"

`sg-`, `vpc-`, `subnet-` 은 **AWS가 예약한 접두어**입니다.

| 사용 가능 ✅ | 사용 불가 ❌ |
|-------------|-------------|
| `petclinic-alb-sg` | `sg-petclinic-alb` |
| `petclinic-app-sg` | `sg-app` |

**접미어(`-sg`)를 쓰세요.**

---

### 오해 ⑤ — "보안 그룹 소스에 IP를 적으면 된다"

Fargate 태스크는 죽고 살아날 때마다 **IP가 바뀝니다.**

```
[ 나쁨 ]  RDS SG 인바운드 3306 ← 소스: 10.0.2.15/32   (태스크 IP)
                                       ↑ 태스크 재시작하면 무효

[ 좋음 ]  RDS SG 인바운드 3306 ← 소스: petclinic-app-sg  (보안 그룹 참조)
                                       ↑ IP가 바뀌어도 항상 유효
```
---

## 4. 사전 준비 체크리스트

> ⏱ 예상 소요시간: 약 60분 (재부팅 2회 포함)
> 📍 작업 위치: **Windows 11 PC**

**지금 `docker` 명령이 안 되는 상태입니다.** 아래를 순서대로 진행하세요.

- [ ] 하드웨어 가상화 활성화 확인
- [ ] WSL2 설치
- [ ] Docker Desktop 설치
- [ ] AWS CLI v2 설치
- [ ] Git for Windows 설치
- [ ] JDK 17 설치
- [ ] AWS 자격증명 설정

---

### 4-0. PowerShell을 관리자로 여는 법 (모든 설치의 전제)

**Windows 키 → `PowerShell` 입력 → 우클릭 → "관리자 권한으로 실행"**

또는 **Windows 키 + X → "터미널(관리자)"**

**기대 결과:** 프롬프트가 아래처럼 시작합니다.
```
PS C:\WINDOWS\system32>
```

> 📌 경로가 `C:\WINDOWS\system32` 이면 관리자 권한입니다.
> `C:\Users\zion3` 로 시작하면 일반 권한이므로 설치가 실패합니다.

---

### 4-1. 하드웨어 가상화 확인 (가장 먼저)

BIOS 수준의 가상화가 꺼져 있는 것이 첫 설치자 대부분이 걸리는 함정입니다.

**작업 관리자(`Ctrl + Shift + Esc`) → 성능 탭 → CPU** 를 선택합니다.

우측 하단을 보세요.

**기대 결과:**
```
가상화:  사용
```

**"사용 안 함"이면** 재부팅해서 BIOS/UEFI에 들어가 켜야 합니다.

| 제조사 | 진입 키 | 설정 이름 |
|--------|--------|----------|
| Lenovo / Dell / HP | `F1` 또는 `F2` | Intel VT-x / AMD-V (SVM) |
| ASUS | `F2` 또는 `Del` | Intel Virtualization Technology |
| 삼성 | `F2` | Virtualization Technology |

> ⚠️ **이것이 꺼져 있으면 Docker Desktop이 절대 시작되지 않습니다.** 여기서 반드시 확인하고 넘어가세요.

PowerShell로도 확인할 수 있습니다.

```powershell
Get-ComputerInfo -Property "HyperVRequirementVirtualizationFirmwareEnabled"
```

**기대 결과:**
```
HyperVRequirementVirtualizationFirmwareEnabled : True
```

---

### 4-2. Windows 버전 확인

📍 **관리자 PowerShell**

```powershell
winver
```

**기대 결과:** 창이 뜨고 아래처럼 표시됩니다.
```
Windows 11 버전 23H2 (OS 빌드 22631.xxxx)
```

Docker Desktop 최신 릴리스와 호환되려면 Windows 11 버전 23H2(빌드 22631) 이상이 필요합니다.

Windows 11은 Home, Pro, Enterprise, Education 모든 에디션을 지원합니다. WSL2가 활성화되고 BIOS에서 하드웨어 가상화가 켜져 있으면 됩니다. Windows 11 Home은 과거 Hyper-V가 Pro를 요구해 차단됐지만, WSL2 백엔드가 그 제약을 없앴습니다.

빌드가 낮으면 **설정 → Windows 업데이트** 에서 먼저 업데이트하세요.

---

### 4-3. WSL2 설치

📍 **관리자 PowerShell**

```powershell
wsl --install
```

**기대 결과:**
```
설치 중: 가상 머신 플랫폼
가상 머신 플랫폼이(가) 설치되었습니다.
설치 중: Linux용 Windows 하위 시스템
Linux용 Windows 하위 시스템이(가) 설치되었습니다.
설치 중: Ubuntu
요청한 작업이 성공했습니다. 변경 내용을 적용하려면 시스템을 다시 시작하세요.
```

> ⚠️ **여기서 반드시 재부팅하세요.** 재부팅하지 않으면 Docker Desktop이 시작되지 않습니다.

```powershell
Restart-Computer
```

---

### 4-4. WSL2 확인 (재부팅 후)

재부팅 후 Ubuntu 창이 자동으로 뜨면 사용자 이름과 비밀번호를 만듭니다. (아무거나 기억하기 쉬운 것)

📍 **일반 PowerShell** (관리자 아니어도 됨)

```powershell
wsl --status
```

**기대 결과:**
```
기본 배포: Ubuntu
기본 버전: 2
```

**`기본 버전: 2` 여야 합니다.** `1` 이면 아래로 바꾸세요.

```powershell
wsl --set-default-version 2
```

버전을 확인합니다.

```powershell
wsl --version
```

**기대 결과:**
```
WSL 버전: 2.3.26.0
커널 버전: 5.15.167.4-1
```

WSL 버전 2.1.5 이상이 필요합니다. 낮으면 업데이트하세요.

```powershell
wsl --update
```

---

### 4-5. WSL2 메모리 제한 설정 (권장)

WSL2는 기본적으로 Windows 메모리를 계속 먹습니다. 상한을 정하세요.

📍 **일반 PowerShell**

```powershell
notepad $env:USERPROFILE\.wslconfig
```

파일이 없다고 나오면 **"예"** 를 눌러 새로 만듭니다. 아래 내용을 붙여넣고 저장하세요.

```ini
[wsl2]
memory=8GB
processors=4
autoMemoryReclaim=gradual
```

> 📌 `autoMemoryReclaim` 은 무거운 이미지 빌드가 끝난 뒤 RAM을 Windows에 동적으로 반환하므로 강력히 권장됩니다.
> PC 메모리가 8GB뿐이면 `memory=4GB` 로 낮추세요.

적용하려면 WSL을 재시작합니다.

```powershell
wsl --shutdown
```

---

### 4-6. Docker Desktop 설치

**브라우저에서 아래 주소로 이동해 설치 파일을 받습니다.**

```
https://www.docker.com/products/docker-desktop/
```

**"Download for Windows - AMD64"** 를 클릭합니다. (ARM 노트북이면 ARM64)

받은 `Docker Desktop Installer.exe` 를 **더블클릭**합니다.

| 설치 화면 항목 | 선택 |
|---------------|------|
| Use WSL 2 instead of Hyper-V | ☑ **체크** |
| Add shortcut to desktop | ☑ 체크 |

**Ok** → 설치가 진행됩니다 (약 5분) → **Close and restart** 클릭.

> ⚠️ **여기서 두 번째 재부팅이 일어납니다.**

---

### 4-7. Docker Desktop 첫 실행

재부팅 후 바탕화면의 **Docker Desktop** 아이콘을 더블클릭합니다.

**약관 화면:** "Accept" 클릭.
**계정 로그인:** "Skip" 또는 "Continue without signing in" 클릭. (실습에 계정 불필요)

**작업 표시줄 오른쪽 아래의 고래 아이콘**을 확인하세요.

| 아이콘 상태 | 의미 |
|------------|------|
| 고래가 움직임 (애니메이션) | 시작 중 — 기다리세요 |
| 고래가 멈춤 + 초록 점 | **준비 완료** |
| 빨간 점 / 느낌표 | 오류 — 아래 표 참조 |

**Docker Desktop 창 왼쪽 아래**에도 상태가 표시됩니다.
```
● Engine running
```

---

### 4-8. docker 명령 확인 (드디어)

📍 **새 PowerShell 창** (기존 창은 PATH를 모릅니다)

> ⚠️ **반드시 PowerShell을 새로 여세요.** 설치 전에 열어둔 창에서는 `docker` 를 찾지 못합니다.
> 이것이 지금 겪으신 `The term 'docker' is not recognized` 오류의 가장 흔한 원인 중 하나입니다.

```powershell
docker --version
```

**기대 결과:**
```
Docker version 27.4.0, build bde2b89
```

```powershell
docker ps
```

**기대 결과:** 헤더만 나오고 오류가 없으면 정상입니다.
```
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

실제 컨테이너를 하나 돌려 검증합니다.

```powershell
docker run --rm hello-world
```

**기대 결과:**
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

> 🎉 **여기까지 오면 Docker 환경 구축이 끝난 것입니다.**

---

### 4-9. Docker 오류 대응표

| PowerShell에 보이는 메시지 | 원인 | 해결 |
|--------------------------|------|------|
| `The term 'docker' is not recognized` | ① Docker Desktop 미설치 | **4-6** 으로 |
| 같은 메시지 (설치는 했는데) | ② PowerShell을 새로 안 염 | **PowerShell 창을 닫고 새로 열기** |
| 같은 메시지 (새 창인데도) | ③ PATH 미등록 | 아래 **4-10** 참조 |
| `error during connect: ... The system cannot find the file` | Docker Desktop이 꺼져 있음 | 바탕화면 아이콘으로 시작 |
| `Docker Desktop - Unexpected WSL error` | WSL2 미설치·구버전 | **4-3, 4-4** 재확인 |
| `WSL 2 is not supported with your current machine configuration` | 가상화 꺼짐 | **4-1** BIOS 설정 |
| `docker: permission denied` | Windows에서는 거의 없음 | Docker Desktop 재시작 |
| 고래 아이콘이 계속 회전 | WSL2 백엔드 시작 실패 | `wsl --shutdown` 후 Docker Desktop 재시작 |

---

### 4-10. PATH가 등록되지 않았을 때

```powershell
$env:Path -split ';' | Select-String "Docker"
```

**기대 결과:**
```
C:\Program Files\Docker\Docker\resources\bin
```

아무것도 안 나오면 수동으로 등록합니다.

**Windows 키 → `환경 변수` 검색 → "시스템 환경 변수 편집" → 환경 변수 → 시스템 변수의 `Path` 선택 → 편집 → 새로 만들기**

```
C:\Program Files\Docker\Docker\resources\bin
```

**확인 → 확인 → PowerShell 새로 열기 → `docker --version` 재시도**

---

### 4-11. AWS CLI v2 설치

📍 **관리자 PowerShell**

```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

설치 마법사가 뜨면 **Next → Next → Install → Finish**.

**새 PowerShell 창**에서 확인합니다.

```powershell
aws --version
```

**기대 결과:**
```
aws-cli/2.24.10 Python/3.12.6 Windows/11 exe/AMD64
```

`aws-cli/1.x` 가 나오면 구버전입니다. 제거 후 재설치하세요.

---

### 4-12. Git for Windows 설치

📍 **관리자 PowerShell**

```powershell
winget install --id Git.Git -e --source winget
```

**새 PowerShell 창**에서 확인합니다.

```powershell
git --version
```

**기대 결과:**
```
git version 2.47.1.windows.1
```

**줄바꿈 문자 설정 (중요):**

Windows는 줄바꿈이 `CRLF`, 리눅스는 `LF` 입니다. Dockerfile이 `CRLF` 로 저장되면 컨테이너 안에서 오류가 납니다.

```powershell
git config --global core.autocrlf input
```

> ⚠️ **이 설정을 하지 않으면** 나중에 `exec /bin/sh: no such file or directory` 같은 알 수 없는 오류가 발생합니다.

---

### 4-13. JDK 17 설치

📍 **관리자 PowerShell**

```powershell
winget install --id EclipseAdoptium.Temurin.17.JDK -e
```

**새 PowerShell 창**에서 확인합니다.

```powershell
java -version
```

**기대 결과:**
```
openjdk version "17.0.13" 2024-10-15
OpenJDK Runtime Environment Temurin-17.0.13+11
```

`17` 이 아니면 PetClinic 빌드가 실패합니다.

`JAVA_HOME` 도 확인합니다.

```powershell
echo $env:JAVA_HOME
```

**기대 결과:**
```
C:\Program Files\Eclipse Adoptium\jdk-17.0.13.11-hotspot\
```

비어 있으면 등록하세요.

```powershell
[Environment]::SetEnvironmentVariable("JAVA_HOME", "C:\Program Files\Eclipse Adoptium\jdk-17.0.13.11-hotspot", "Machine")
```

> 📌 경로의 버전 번호는 실제 설치된 것으로 바꾸세요. `dir "C:\Program Files\Eclipse Adoptium"` 로 확인합니다.

---

### 4-14. AWS 자격증명 설정

📍 **일반 PowerShell**

```powershell
aws configure
```

네 가지를 차례로 묻습니다.

```
AWS Access Key ID [None]: AKIAXXXXXXXXXXXXXXXX
AWS Secret Access Key [None]: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Default region name [None]: us-west-2
Default output format [None]: json
```

> ⚠️ **리전은 반드시 `us-west-2`** 입니다.

확인합니다.

```powershell
aws sts get-caller-identity
```

**기대 결과:**
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/student01"
}
```

> 📌 **`Account` 값(12자리 숫자)을 메모하세요.** 이후 ECR 주소에 계속 씁니다.
> 이 문서에서는 `<계정ID>` 로 표기합니다.

**오류가 나면:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `Unable to locate credentials` | 자격증명 미설정 | `aws configure` 재실행 |
| `The security token ... is invalid` | 키 만료·오타 | IAM에서 키 재발급 |
| `Could not connect to the endpoint URL` | 리전 오타 | `aws configure` 로 `us-west-2` 재입력 |

---

### 4-15. 환경변수 프로필 설정 (매우 권장)

이 실습에서 계속 쓰는 값들입니다. **PowerShell 프로필에 넣어 두면 창을 새로 열어도 유지됩니다.**

```powershell
notepad $PROFILE
```

"파일을 만들까요?"가 뜨면 **예** 를 누릅니다. 아래를 붙여넣고 저장하세요.

```powershell
$env:AWS_REGION = "us-west-2"
$env:ACCOUNT_ID = (aws sts get-caller-identity --query Account --output text)
$env:ECR_URI = "$($env:ACCOUNT_ID).dkr.ecr.us-west-2.amazonaws.com/petclinic"
```

**PowerShell을 새로 열어** 확인합니다.

```powershell
echo $env:ECR_URI
```

**기대 결과:**
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic
```

**실행 정책 오류가 나면:**
```
$PROFILE ... cannot be loaded because running scripts is disabled
```

📍 **관리자 PowerShell** 에서 한 번만 실행하세요.

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

`Y` 를 입력해 승인합니다.

---

## 5. Step 1 — 소스 코드 내려받기

> ⏱ 예상 소요시간: 약 10분
> 📍 작업 위치: **PowerShell** (`C:\Users\zion3\Desktop\3tier`)

---

### 5-1. 저장소 복제

지금 계신 폴더에서 진행합니다.

```powershell
cd C:\Users\zion3\Desktop\3tier
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

**기대 결과:**
```
Cloning into 'spring-petclinic'...
remote: Enumerating objects: 8000, done.
Resolving deltas: 100% (4000/4000), done.
```

> ⚠️ **경로에 한글이나 공백이 없어야 합니다.** `바탕 화면` 같은 한글 경로에서는 Maven 빌드가 실패할 수 있습니다.
> `C:\Users\zion3\Desktop\3tier` 는 영문·숫자뿐이라 안전합니다.

---

### 5-2. Spring Boot 버전인지 확인

오해 ①을 실제로 확인하는 단계입니다.

```powershell
Select-String -Path pom.xml -Pattern "spring-boot-starter-parent" -Context 0,2
```

**기대 결과:**
```
> pom.xml:20:    <artifactId>spring-boot-starter-parent</artifactId>
```

이 출력이 보이면 **Spring Boot 버전이 맞습니다.** 아무것도 안 나오면 잘못된 저장소입니다. **5-1로 돌아가세요.**

---

### 5-3. 로컬 빌드

**Windows에서는 `mvnw.cmd` 를 씁니다.** (리눅스의 `./mvnw` 가 아닙니다)

```powershell
.\mvnw.cmd clean package -DskipTests
```

> ⏱ 처음 실행하면 의존성을 내려받느라 **5~10분** 걸립니다. 정상입니다.

**기대 결과 (마지막 부분):**
```
[INFO] BUILD SUCCESS
[INFO] Total time:  03:12 min
```

빌드 산출물을 확인합니다.

```powershell
dir target\*.jar
```

**기대 결과:**
```
    디렉터리: C:\Users\zion3\Desktop\3tier\spring-petclinic\target

Mode    LastWriteTime      Length Name
----    -------------      ------ ----
-a---   2026-07-10  14:22  58720256 spring-petclinic-3.4.0-SNAPSHOT.jar
```

> 📌 **jar 파일 이름을 메모하세요.** 버전 번호는 다를 수 있습니다.

**실패 시:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `'.\mvnw.cmd'을(를) 인식할 수 없습니다` | 폴더 위치 오류 | `cd spring-petclinic` 확인 |
| `Unsupported class file major version` | Java 버전 불일치 | **4-13** Java 17 설치 |
| `JAVA_HOME not found` | 환경변수 미설정 | **4-13** `JAVA_HOME` 등록 |
| `Could not resolve dependencies` | 네트워크 차단 | 방화벽·프록시 확인 |

---

## 6. Step 2 — Dockerfile 작성

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **PowerShell** (`...\3tier\spring-petclinic`)

---

### 6-1. Dockerfile 생성

프로젝트 루트(`pom.xml`이 있는 위치)에 `Dockerfile` 을 만듭니다. **확장자 없습니다.**

```powershell
notepad Dockerfile
```

"파일을 만들까요?"가 뜨면 **예** 를 클릭합니다. 아래 내용을 붙여넣고 저장하세요.

```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S app && adduser -S app -G app

COPY target/spring-petclinic-*.jar app.jar

RUN chown app:app app.jar
USER app

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

> ⚠️ **메모장 저장 시 "파일 형식"을 "모든 파일 (*.*)"로 바꾸세요.**
> 그러지 않으면 `Dockerfile.txt` 로 저장됩니다. 이것이 Windows 초보자의 대표적 함정입니다.

> ⚠️ **인코딩은 `UTF-8`** 을 선택하세요. `ANSI` 나 `UTF-8 BOM` 은 오류를 냅니다.

**각 줄이 하는 일:**

| 줄 | 의미 |
|----|------|
| `FROM eclipse-temurin:17-jre-alpine` | 밑바탕. Java 17 실행 환경만 담긴 가벼운 이미지 |
| `WORKDIR /app` | 이후 명령의 기준 폴더 |
| `addgroup / adduser` | **root가 아닌 계정** 생성 (보안) |
| `COPY target/...jar app.jar` | 빌드한 jar를 이미지 안으로 복사 |
| `USER app` | root 권한을 버리고 일반 계정으로 전환 |
| `EXPOSE 8080` | 이 컨테이너는 8080 포트를 쓴다는 표시 |
| `ENTRYPOINT` | 컨테이너가 시작될 때 실행할 명령 |

---

### 6-2. 파일 이름과 위치 검증 (Windows 필수 확인)

```powershell
dir Dockerfile, pom.xml
```

**기대 결과:** 두 파일이 **같은 폴더**에 보이고, 이름이 정확히 `Dockerfile` 이어야 합니다.
```
Mode    LastWriteTime      Length Name
----    -------------      ------ ----
-a---   2026-07-10  14:30      287 Dockerfile
-a---   2026-07-10  14:10     8421 pom.xml
```

**`Dockerfile.txt` 로 저장됐다면** 이름을 바꾸세요.

```powershell
Rename-Item Dockerfile.txt Dockerfile
```

**숨겨진 확장자를 보이게 하려면:** 탐색기 → 보기 → 표시 → **파일 확장명** 체크.

---

### 6-3. 줄바꿈 문자 확인 (Windows 필수 확인)

```powershell
$content = Get-Content Dockerfile -Raw
if ($content -match "`r`n") { "CRLF (문제 있음)" } else { "LF (정상)" }
```

**기대 결과:**
```
LF (정상)
```

**`CRLF (문제 있음)` 이 나오면** 아래로 변환하세요.

```powershell
$content = (Get-Content Dockerfile -Raw) -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\Dockerfile", $content)
```

> ⚠️ **CRLF 상태로 두면** 나중에 컨테이너가 `exec /bin/sh: no such file or directory` 로 죽습니다.
> 원인을 찾기 매우 어려운 오류입니다. 지금 반드시 확인하세요.

---

### 6-4. .dockerignore 생성

```powershell
notepad .dockerignore
```

아래를 붙여넣고 **모든 파일 (*.*)** 형식으로 저장하세요.

```
.git
.mvn
src
*.md
target/*
!target/*.jar
```

마지막 두 줄이 핵심입니다. `target/` 전체를 제외하되, `.jar` 파일만 예외로 포함시킵니다.

확인합니다.

```powershell
dir .dockerignore
```

---

### 6-5. 로컬에서 이미지 빌드

**Docker Desktop이 실행 중인지 먼저 확인하세요.** (작업 표시줄 고래 아이콘)

```powershell
docker build -t petclinic:v1 .
```

> 📌 명령 끝의 점(`.`)은 "현재 폴더를 빌드 컨텍스트로 삼는다"는 뜻입니다. 빠뜨리면 실패합니다.

**기대 결과 (마지막 부분):**
```
 => exporting to image
 => => writing image sha256:a1b2c3...
 => => naming to docker.io/library/petclinic:v1
```

---

### 6-6. 이미지 확인

```powershell
docker images petclinic
```

**기대 결과:**
```
REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
petclinic    v1    a1b2c3d4e5f6   10 seconds ago   215MB
```

크기가 200MB 내외면 정상입니다. 800MB를 넘으면 `.dockerignore`가 적용되지 않은 것입니다. **6-4로 돌아가세요.**

---

### 6-7. 로컬에서 실행해 보기 (중요)

**AWS에 올리기 전에 여기서 반드시 확인하세요.** 여기서 안 되면 클라우드에서도 안 됩니다.

```powershell
docker run --rm -p 8080:8080 petclinic:v1
```

**기대 결과 (약 20~30초 후):**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
...
Started PetClinicApplication in 12.324 seconds
```

**새 PowerShell 창**을 열어 접속을 확인합니다.

```powershell
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8080
```

> ⚠️ **`curl` 이 아니라 `curl.exe`** 입니다. (오해 ⓐ 참조)
> 리눅스의 `/dev/null` 대신 Windows는 `NUL` 을 씁니다.

**기대 결과:**
```
200
```

브라우저에서 `http://localhost:8080` 을 열어도 됩니다. PetClinic 홈 화면이 보입니다.

원래 창에서 `Ctrl + C` 로 종료하세요.

> 📌 지금은 DB 설정 없이 내장 H2 데이터베이스로 뜹니다. RDS 연결은 Step 8에서 붙입니다.

**실패 시:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `exec /bin/sh: no such file or directory` | **Dockerfile이 CRLF** | **6-3** 으로 |
| `no main manifest attribute` | jar가 실행 가능 형태 아님 | **5-3** 재빌드 |
| `COPY failed: no source files` | `target\*.jar` 없음 | **5-3** 빌드 |
| `port is already allocated` | 8080 사용 중 | `-p 8081:8080` 으로 변경 |
| `error during connect` | Docker Desktop 꺼짐 | 고래 아이콘 확인 |
| `failed to solve: ... Dockerfile: not found` | 파일명이 `Dockerfile.txt` | **6-2** 로 |
---

## 7. Step 3 — VPC 신규 구축

> ⏱ 예상 소요시간: 약 25분
> 📍 작업 위치: **AWS 콘솔** (VPC)

> ⚠️ **리전이 `us-west-2` (오레곤)인지 콘솔 우측 상단에서 먼저 확인하세요.**

---

### 7-1. VPC 생성

**AWS 콘솔 → VPC → VPC 생성** 을 클릭합니다.

**"VPC 등"(VPC and more)** 을 선택하세요.

| 입력 항목 | 값 |
|----------|-----|
| 이름 태그 자동 생성 | `petclinic` |
| IPv4 CIDR 블록 | `10.0.0.0/16` |
| IPv6 CIDR 블록 | IPv6 CIDR 블록 없음 |
| 가용 영역(AZ) 수 | **2** |
| 퍼블릭 서브넷 수 | **2** |
| 프라이빗 서브넷 수 | **2** |
| NAT 게이트웨이 | **없음** ← 중요 |
| VPC 엔드포인트 | **없음** ← 나중에 직접 만듭니다 |
| DNS 호스트 이름 활성화 | ☑ 체크 |
| DNS 확인 활성화 | ☑ 체크 |

> ⚠️ **NAT 게이트웨이를 "없음"으로 두는 것이 이 실습의 핵심입니다.**

**VPC 생성** 클릭.

---

### 7-2. 서브넷 확인

📍 **PowerShell**

```powershell
aws ec2 describe-subnets --filters "Name=tag:Name,Values=petclinic-*" --query "Subnets[*].[Tags[?Key=='Name']|[0].Value,CidrBlock,AvailabilityZone]" --output table
```

**기대 결과:**
```
--------------------------------------------------------------------------
|                             DescribeSubnets                            |
+---------------------------------------+---------------+----------------+
|  petclinic-subnet-public1-us-west-2a  |  10.0.0.0/20  |  us-west-2a    |
|  petclinic-subnet-public2-us-west-2b  |  10.0.16.0/20 |  us-west-2b    |
|  petclinic-subnet-private1-us-west-2a |  10.0.128.0/20|  us-west-2a    |
|  petclinic-subnet-private2-us-west-2b |  10.0.144.0/20|  us-west-2b    |
+---------------------------------------+---------------+----------------+
```

4개가 모두 보이고 AZ가 둘로 나뉘어야 합니다.

> 📌 **PowerShell에서 `--query` 의 작은따옴표 주의**
> 위 명령처럼 전체를 큰따옴표로 감싸고 내부에 작은따옴표를 씁니다. 순서를 바꾸면 오류가 납니다.

---

### 7-3. 프라이빗 서브넷에 인터넷 경로가 없는지 확인

**VPC → 서브넷 → `petclinic-subnet-private1-us-west-2a` 선택 → 라우팅 테이블 탭**

**기대 결과:**

| 대상 | 대상(Target) |
|------|-------------|
| `10.0.0.0/16` | local |

**`0.0.0.0/0` 행이 없어야 정상입니다.** 있으면 NAT나 IGW가 붙은 것이므로 VPC를 삭제하고 **7-1부터 다시** 하세요.

---

### 7-4. 퍼블릭 서브넷에는 인터넷 경로가 있는지 확인

**VPC → 서브넷 → `petclinic-subnet-public1-us-west-2a` 선택 → 라우팅 테이블 탭**

**기대 결과:**

| 대상 | 대상(Target) |
|------|-------------|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | `igw-xxxxxxxx` |

---

### 7-5. DNS 설정 확인

📍 **PowerShell**

```powershell
$vpcId = aws ec2 describe-vpcs --filters "Name=tag:Name,Values=petclinic-vpc" --query "Vpcs[0].VpcId" --output text
aws ec2 describe-vpc-attribute --vpc-id $vpcId --attribute enableDnsHostnames --query "EnableDnsHostnames.Value"
aws ec2 describe-vpc-attribute --vpc-id $vpcId --attribute enableDnsSupport --query "EnableDnsSupport.Value"
```

**기대 결과:**
```
true
true
```

> ⚠️ 하나라도 `false` 면 VPC 엔드포인트가 동작하지 않습니다.
> **VPC → 작업 → VPC 설정 편집** 에서 둘 다 체크하세요.

---

## 8. Step 4 — 보안 그룹 4개 생성

> ⏱ 예상 소요시간: 약 20분
> 📍 작업 위치: **AWS 콘솔** (VPC → 보안 그룹)

**만들 순서가 중요합니다.** 뒤의 그룹이 앞의 그룹을 참조합니다.

```
① petclinic-alb-sg       (참조 없음)
② petclinic-app-sg       ← ①을 참조
③ petclinic-db-sg        ← ②를 참조
④ petclinic-endpoint-sg  ← ②를 참조
```

---

### 8-1. ALB용 보안 그룹

**VPC → 보안 그룹 → 보안 그룹 생성**

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-alb-sg` |
| 설명 | `Allow HTTP from internet to ALB` |
| VPC | `petclinic-vpc` |

**인바운드 규칙:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| HTTP | TCP | 80 | `0.0.0.0/0` |

---

### 8-2. 태스크용 보안 그룹

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-app-sg` |
| 설명 | `Allow 8080 from ALB only` |
| VPC | `petclinic-vpc` |

**인바운드 규칙:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| 사용자 지정 TCP | TCP | **8080** | **`petclinic-alb-sg`** ← 보안 그룹 선택 |

> 📌 소스 칸에 IP를 적지 마세요. **드롭다운에서 `petclinic-alb-sg` 를 선택**합니다. (오해 ⑤)

---

### 8-3. RDS용 보안 그룹

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-db-sg` |
| 설명 | `Allow 3306 from app tasks only` |
| VPC | `petclinic-vpc` |

**인바운드 규칙:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| MYSQL/Aurora | TCP | 3306 | **`petclinic-app-sg`** |

---

### 8-4. VPC 엔드포인트용 보안 그룹

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-endpoint-sg` |
| 설명 | `Allow 443 from app tasks to VPC endpoints` |
| VPC | `petclinic-vpc` |

**인바운드 규칙:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| HTTPS | TCP | **443** | **`petclinic-app-sg`** |

> ⚠️ **443입니다. 80이 아닙니다.**

---

### 8-5. 4개 모두 만들어졌는지 확인

📍 **PowerShell**

```powershell
aws ec2 describe-security-groups --filters "Name=group-name,Values=petclinic-*" --query "SecurityGroups[*].[GroupName,GroupId]" --output table
```

**기대 결과:**
```
--------------------------------------------------
|            DescribeSecurityGroups              |
+-------------------------+----------------------+
|  petclinic-alb-sg       |  sg-0aaa111...       |
|  petclinic-app-sg       |  sg-0bbb222...       |
|  petclinic-db-sg        |  sg-0ccc333...       |
|  petclinic-endpoint-sg  |  sg-0ddd444...       |
+-------------------------+----------------------+
```

---

### 8-6. 보안 그룹 관계 한눈에 보기

```
        인터넷
          │ 80
          ▼
  ┌───────────────────┐
  │ petclinic-alb-sg  │
  └─────────┬─────────┘
            │ 8080 (SG 참조)
            ▼
  ┌───────────────────┐
  │ petclinic-app-sg  │ ◄─── 태스크가 여기 붙음
  └───┬───────────┬───┘
      │ 3306      │ 443
      ▼           ▼
┌──────────┐ ┌──────────────────────┐
│ db-sg    │ │ endpoint-sg          │
│ (RDS)    │ │ (ECR/Secrets/Logs)   │
└──────────┘ └──────────────────────┘
```

---

## 9. Step 5 — VPC 엔드포인트 5개 생성

> ⏱ 예상 소요시간: 약 25분
> 📍 작업 위치: **AWS 콘솔** (VPC → 엔드포인트)

**개념 8을 다시 읽고 오세요.**

---

### 9-1. 인터페이스 엔드포인트 4개

**VPC → 엔드포인트 → 엔드포인트 생성**

아래 4개를 **똑같은 절차로 하나씩** 만듭니다.

| # | 이름 태그 | 서비스 이름 |
|---|----------|------------|
| 1 | `petclinic-ecr-api-ep` | `com.amazonaws.us-west-2.ecr.api` |
| 2 | `petclinic-ecr-dkr-ep` | `com.amazonaws.us-west-2.ecr.dkr` |
| 3 | `petclinic-secrets-ep` | `com.amazonaws.us-west-2.secretsmanager` |
| 4 | `petclinic-logs-ep` | `com.amazonaws.us-west-2.logs` |

**각각에 대해 아래를 동일하게 입력합니다:**

| 입력 항목 | 값 |
|----------|-----|
| 서비스 범주 | AWS 서비스 |
| 서비스 | 위 표의 서비스 이름을 검색창에 붙여넣기 |
| VPC | `petclinic-vpc` |
| **서브넷** | **프라이빗 서브넷 2개 모두 선택** |
| IP 주소 유형 | IPv4 |
| **보안 그룹** | **`petclinic-endpoint-sg`** ← 기본값(default) 해제 필수 |
| 정책 | 전체 액세스 |

> ⚠️ **보안 그룹에서 `default` 를 반드시 해제하세요.**
> ⚠️ **서브넷은 프라이빗 2개**입니다.

각 엔드포인트가 `사용 가능(Available)` 상태가 될 때까지 **2~3분** 기다립니다.

---

### 9-2. 게이트웨이 엔드포인트 1개 (S3)

**이것만 절차가 다릅니다.** 서브넷·보안 그룹 대신 **라우팅 테이블**을 고릅니다.

| 입력 항목 | 값 |
|----------|-----|
| 이름 태그 | `petclinic-s3-ep` |
| 서비스 | `com.amazonaws.us-west-2.s3` |
| **유형** | **Gateway** ← Interface 아님 |
| VPC | `petclinic-vpc` |
| **라우팅 테이블** | **프라이빗 서브넷의 라우팅 테이블 선택** |
| 정책 | 전체 액세스 |

> ⚠️ **이 단계를 빠뜨리면 Step 12에서 `CannotPullContainerError`가 발생합니다.**

---

### 9-3. 5개 모두 확인

📍 **PowerShell**

```powershell
aws ec2 describe-vpc-endpoints --query "VpcEndpoints[*].[ServiceName,VpcEndpointType,State]" --output table
```

**기대 결과:**
```
-----------------------------------------------------------------------
|                        DescribeVpcEndpoints                         |
+------------------------------------------+-----------+--------------+
|  com.amazonaws.us-west-2.ecr.api          |  Interface|  available   |
|  com.amazonaws.us-west-2.ecr.dkr          |  Interface|  available   |
|  com.amazonaws.us-west-2.secretsmanager   |  Interface|  available   |
|  com.amazonaws.us-west-2.logs             |  Interface|  available   |
|  com.amazonaws.us-west-2.s3               |  Gateway  |  available   |
+------------------------------------------+-----------+--------------+
```

**5개가 모두 `available` 이어야 다음으로 넘어갑니다.**

| 상태 | 해결 |
|------|------|
| `pending` | 2~3분 더 기다림 |
| `failed` | 삭제 후 **9-1부터 재시도** |
| 목록에 없음 | **9-1 또는 9-2 재수행** |

---

## 10. Step 6 — ECR 리포지토리 생성 및 이미지 푸시

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **AWS 콘솔** → **PowerShell**

---

### 10-1. 리포지토리 생성 (콘솔)

**AWS 콘솔 → ECR → 리포지토리 → 리포지토리 생성**

| 입력 항목 | 값 |
|----------|-----|
| 표시 여부 설정 | **프라이빗** |
| 리포지토리 이름 | `petclinic` |
| 태그 변경 불가능 | 비활성화 |
| 이미지 스캔 | 활성화 |
| 암호화 | AES-256 |

**기대 결과:** URI 열에 아래 형식의 주소가 보입니다.
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic
```

---

### 10-2. 환경변수 확인

📍 **PowerShell**

**4-15**에서 프로필을 설정했다면 이미 준비되어 있습니다.

```powershell
echo $env:ECR_URI
```

**기대 결과:**
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic
```

비어 있으면 지금 설정합니다.

```powershell
$env:ACCOUNT_ID = (aws sts get-caller-identity --query Account --output text)
$env:ECR_URI = "$($env:ACCOUNT_ID).dkr.ecr.us-west-2.amazonaws.com/petclinic"
echo $env:ECR_URI
```

> ⚠️ **PowerShell 변수는 `$env:` 접두어가 필요합니다.** 리눅스의 `export VAR=값` 이 아닙니다. (오해 ⓐ)

---

### 10-3. ECR 로그인 (PowerShell 전용 방식)

> ⚠️ **오해 ⓑ를 반드시 읽고 오세요.** AWS 문서의 파이프 방식은 PowerShell에서 실패합니다.

```powershell
$pw = aws ecr get-login-password --region us-west-2
docker login --username AWS --password $pw "$($env:ACCOUNT_ID).dkr.ecr.us-west-2.amazonaws.com"
```

**기대 결과:**
```
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Login Succeeded
```

> 📌 **`WARNING` 은 무시해도 됩니다.** `Login Succeeded` 만 나오면 성공입니다.

**실패 시:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `failed with status: 400 Bad Request` | 파이프(`\|`) 방식 사용 | 위 두 줄 방식으로 |
| `no basic auth credentials` | 로그인 안 됨 | 이 명령 재실행 |
| `AccessDeniedException` | IAM 권한 부족 | `AmazonEC2ContainerRegistryFullAccess` 확인 |
| `error during connect` | Docker Desktop 꺼짐 | 고래 아이콘 확인 |

---

### 10-4. 이미지에 ECR 주소 태그 붙이기

```powershell
docker tag petclinic:v1 "$($env:ECR_URI):v1"
```

> 📌 **PowerShell에서 변수 뒤에 콜론(`:`)이 오면** `$env:ECR_URI:v1` 이 하나의 변수명으로 해석됩니다.
> 반드시 `"$($env:ECR_URI):v1"` 처럼 `$()` 로 감싸세요.

확인합니다.

```powershell
docker images | Select-String petclinic
```

**기대 결과:** 두 줄이 보입니다. IMAGE ID가 같습니다.
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic   v1   a1b2c3d4e5f6   ...
petclinic                                                v1   a1b2c3d4e5f6   ...
```

---

### 10-5. 푸시

```powershell
docker push "$($env:ECR_URI):v1"
```

**기대 결과 (마지막 줄):**
```
v1: digest: sha256:xxxx... size: 1573
```

---

### 10-6. ECR에 올라갔는지 확인

```powershell
aws ecr describe-images --repository-name petclinic --query "imageDetails[*].[imageTags[0],imageSizeInBytes]" --output table
```

**기대 결과:**
```
------------------------------
|       DescribeImages       |
+------+---------------------+
|  v1  |  215000000          |
+------+---------------------+
```

---

## 11. Step 7 — RDS MariaDB 생성

> ⏱ 예상 소요시간: 약 20분 (생성 대기 10분 포함)
> 📍 작업 위치: **AWS 콘솔** (RDS)

---

### 11-1. DB 서브넷 그룹 생성

**AWS 콘솔 → RDS → 서브넷 그룹 → DB 서브넷 그룹 생성**

| 입력 항목 | 값 |
|----------|-----|
| 이름 | `petclinic-db-subnet-group` |
| 설명 | `Private subnets for PetClinic RDS` |
| VPC | `petclinic-vpc` |
| 가용 영역 | `us-west-2a`, `us-west-2b` **둘 다** |
| 서브넷 | **프라이빗 서브넷 2개 선택** |

> ⚠️ **퍼블릭 서브넷을 고르지 마세요.**

---

### 11-2. RDS 인스턴스 생성

**RDS → 데이터베이스 → 데이터베이스 생성**

| 입력 항목 | 값 |
|----------|-----|
| 생성 방식 | 표준 생성 |
| 엔진 | **MariaDB** |
| 템플릿 | **프리 티어** |
| DB 인스턴스 식별자 | `petclinic-db` |
| 마스터 사용자 이름 | `petclinic` |
| 자격 증명 관리 | **자체 관리** |
| 마스터 암호 | `ChangeMe2026!` |
| 인스턴스 구성 | `db.t3.micro` |
| 스토리지 | gp3, 20GB |
| 스토리지 자동 조정 | 비활성화 |
| **컴퓨팅 리소스** | **EC2 컴퓨팅 리소스에 연결 안 함** |
| VPC | `petclinic-vpc` |
| DB 서브넷 그룹 | `petclinic-db-subnet-group` |
| **퍼블릭 액세스** | **아니요** ← 중요 |
| VPC 보안 그룹 | **기존 항목 선택 → `petclinic-db-sg`** (default 해제) |
| 가용 영역 | `us-west-2a` |
| 데이터베이스 포트 | 3306 |
| **초기 데이터베이스 이름** | **`petclinic`** ← 추가 구성에서 입력 |
| 자동 백업 | 비활성화 |
| 삭제 방지 | 비활성화 |

> ⚠️ **"초기 데이터베이스 이름"을 비워 두면 DB 스키마가 만들어지지 않습니다.**
> 추가 구성(Additional configuration) 섹션을 펼쳐서 반드시 `petclinic` 을 입력하세요.

**데이터베이스 생성** 클릭. **약 10분 걸립니다.**

---

### 11-3. 엔드포인트 주소 확인

📍 **PowerShell**

```powershell
aws rds describe-db-instances --db-instance-identifier petclinic-db --query "DBInstances[0].[DBInstanceStatus,Endpoint.Address]" --output text
```

**기대 결과:**
```
available    petclinic-db.abcdefghij.us-west-2.rds.amazonaws.com
```

`creating` 이 나오면 아직 생성 중입니다. 몇 분 더 기다리세요.

> 📌 **이 엔드포인트 주소 전체를 메모하세요.** Step 8 시크릿에 넣습니다.

변수에 담아 두면 편합니다.

```powershell
$env:RDS_ENDPOINT = (aws rds describe-db-instances --db-instance-identifier petclinic-db --query "DBInstances[0].Endpoint.Address" --output text)
echo $env:RDS_ENDPOINT
```

---

## 12. Step 8 — Secrets Manager에 DB 자격증명 저장

> ⏱ 예상 소요시간: 약 10분
> 📍 작업 위치: **AWS 콘솔** (Secrets Manager)

---

### ✅ 왜 시크릿을 쓰는가?

```
[ 나쁨 ]  태스크 정의에 평문으로 박아 넣기
   environment:
     - SPRING_DATASOURCE_PASSWORD = ChangeMe2026!
                                    ↑ 콘솔에서 누구나 볼 수 있음

[ 좋음 ]  Secrets Manager에서 런타임에 주입
   secrets:
     - SPRING_DATASOURCE_PASSWORD ← arn:aws:secretsmanager:...:password
                                    ↑ 태스크 정의에는 "주소"만 있음
```

---

### 12-1. 시크릿 생성

**AWS 콘솔 → Secrets Manager → 새 보안 암호 저장**

**1단계 — 보안 암호 유형:**

| 입력 항목 | 값 |
|----------|-----|
| 유형 | **다른 유형의 보안 암호** |

> 📌 "Amazon RDS 데이터베이스에 대한 자격 증명"을 고르지 마세요. 키 이름이 자동으로 정해져 Spring Boot가 인식하지 못합니다.

**키/값 쌍**에 아래 3개를 입력합니다.

| 키 | 값 |
|----|-----|
| `url` | `jdbc:mysql://<RDS엔드포인트>:3306/petclinic` |
| `username` | `petclinic` |
| `password` | `ChangeMe2026!` |

`<RDS엔드포인트>` 자리에 11-3에서 확인한 주소를 넣으세요. 예:
```
jdbc:mysql://petclinic-db.abcdefghij.us-west-2.rds.amazonaws.com:3306/petclinic
```

> ⚠️ **MariaDB인데 왜 `jdbc:mysql://` 인가요?**
> MariaDB는 MySQL 프로토콜과 호환됩니다. PetClinic은 MySQL 드라이버를 포함하고 있어 그대로 접속됩니다. `jdbc:mariadb://` 로 쓰면 드라이버가 없어 실패합니다.

**2단계 — 보안 암호 구성:**

| 입력 항목 | 값 |
|----------|-----|
| 보안 암호 이름 | `petclinic/db` |
| 설명 | `PetClinic RDS credentials` |

**3단계 — 교체 구성:** 자동 교체 **비활성화**

**저장** 클릭.

---

### 12-2. 시크릿 ARN 확인

📍 **PowerShell**

```powershell
aws secretsmanager describe-secret --secret-id petclinic/db --query "ARN" --output text
```

**기대 결과:**
```
arn:aws:secretsmanager:us-west-2:123456789012:secret:petclinic/db-AbCdEf
```

> 📌 **끝의 6자리 임의 문자(`-AbCdEf`)까지 포함한 전체 ARN을 메모하세요.**
> 이 문자는 시크릿마다 다릅니다. 빼먹으면 태스크 정의에서 시크릿을 못 찾습니다.

변수에 담아 두세요.

```powershell
$env:SECRET_ARN = (aws secretsmanager describe-secret --secret-id petclinic/db --query "ARN" --output text)
echo $env:SECRET_ARN
```

---

### 12-3. 값이 제대로 들어갔는지 확인

```powershell
aws secretsmanager get-secret-value --secret-id petclinic/db --query "SecretString" --output text
```

**기대 결과:**
```json
{"url":"jdbc:mysql://petclinic-db.abcdefghij.us-west-2.rds.amazonaws.com:3306/petclinic","username":"petclinic","password":"ChangeMe2026!"}
```

세 개 키가 모두 보이면 정상입니다. 하나라도 없으면 **12-1로 돌아가** 편집하세요.

---

## 13. Step 9 — IAM 태스크 실행 역할 생성

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **AWS 콘솔** (IAM)

**개념 7을 다시 읽고 오세요.** 여기서 만드는 것은 **태스크 실행 역할**입니다.

---

### 13-1. 역할 생성

**AWS 콘솔 → IAM → 역할 → 역할 생성**

**1단계 — 신뢰할 수 있는 엔터티:**

| 입력 항목 | 값 |
|----------|-----|
| 유형 | **AWS 서비스** |
| 서비스 또는 사용 사례 | **Elastic Container Service** |
| 사용 사례 | **Elastic Container Service Task** |

> ⚠️ **"Elastic Container Service Task"** 를 고르세요. 그냥 "Elastic Container Service"가 아닙니다.
> 잘못 고르면 신뢰 정책 주체가 `ecs.amazonaws.com` 이 되어 태스크가 역할을 맡지 못합니다.

**2단계 — 권한 추가:**

검색창에 `AmazonECSTaskExecutionRolePolicy` 를 입력해 체크합니다.

**3단계 — 이름 지정:**

| 입력 항목 | 값 |
|----------|-----|
| 역할 이름 | `ecsTaskExecutionRole-petclinic` |

---

### 13-2. 시크릿 읽기 권한 추가 (인라인 정책)

**`AmazonECSTaskExecutionRolePolicy` 에는 Secrets Manager 권한이 없습니다.**

> ⚠️ **이 단계를 빠뜨리는 것이 이 실습에서 가장 많이 발생하는 오류입니다.**

**IAM → 역할 → `ecsTaskExecutionRole-petclinic` → 권한 탭 → 권한 추가 → 인라인 정책 생성**

**JSON 탭**을 클릭하고 아래를 붙여넣습니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-west-2:<계정ID>:secret:petclinic/db-*"
      ]
    }
  ]
}
```

> 📌 `<계정ID>` 를 실제 12자리 숫자로 바꾸세요. 끝의 `-*` 는 그대로 두세요.

**정책 이름**: `PetClinicSecretsAccess`

---

### 13-3. 역할에 정책 2개가 붙었는지 확인

📍 **PowerShell**

```powershell
aws iam list-attached-role-policies --role-name ecsTaskExecutionRole-petclinic --query "AttachedPolicies[*].PolicyName" --output text
aws iam list-role-policies --role-name ecsTaskExecutionRole-petclinic --query "PolicyNames" --output text
```

**기대 결과:**
```
AmazonECSTaskExecutionRolePolicy
PetClinicSecretsAccess
```

---

### 13-4. 신뢰 정책 확인 (중요)

```powershell
aws iam get-role --role-name ecsTaskExecutionRole-petclinic --query "Role.AssumeRolePolicyDocument.Statement[0].Principal.Service" --output text
```

**기대 결과:**
```
ecs-tasks.amazonaws.com
```

> ⚠️ **`ecs.amazonaws.com` 이 나오면 잘못된 것입니다.** 역할을 삭제하고 **13-1부터 다시** 하세요.
---

## 14. Step 10 — ECS 클러스터와 태스크 정의

> ⏱ 예상 소요시간: 약 20분
> 📍 작업 위치: **AWS 콘솔** (ECS)

---

### 14-1. 클러스터 생성

**AWS 콘솔 → ECS → 클러스터 → 클러스터 생성**

| 입력 항목 | 값 |
|----------|-----|
| 클러스터 이름 | `petclinic-cluster` |
| 인프라 | **AWS Fargate(서버리스)** ☑ |
| Amazon EC2 인스턴스 | 체크 해제 |
| 모니터링 | Container Insights 비활성화 |

**생성** 클릭. 약 1분 걸립니다.

---

### 14-2. CloudWatch 로그 그룹 미리 만들기

📍 **PowerShell**

```powershell
aws logs create-log-group --log-group-name /ecs/petclinic --region us-west-2
```

**기대 결과:** 아무 출력도 없으면 성공입니다.

이미 있으면 아래가 나옵니다. 무시하고 진행하세요.
```
An error occurred (ResourceAlreadyExistsException) ...
```

확인합니다.

```powershell
aws logs describe-log-groups --log-group-name-prefix /ecs/petclinic --query "logGroups[0].logGroupName" --output text
```

**기대 결과:**
```
/ecs/petclinic
```

---

### 14-3. 태스크 정의 생성

**ECS → 태스크 정의 → 새 태스크 정의 생성**

**1단계 — 태스크 정의 구성:**

| 입력 항목 | 값 |
|----------|-----|
| 태스크 정의 패밀리 | `petclinic-task` |
| 시작 유형 | **AWS Fargate** |
| 운영 체제/아키텍처 | **Linux/X86_64** |
| 네트워크 모드 | `awsvpc` |
| CPU | **1 vCPU** |
| 메모리 | **2GB** |
| **태스크 실행 역할** | **`ecsTaskExecutionRole-petclinic`** |
| 태스크 역할 | **없음** ← 개념 7 참조 |

> ⚠️ **운영 체제/아키텍처가 `Linux/X86_64` 인지 확인하세요.**
> Windows PC에서 빌드했어도 컨테이너는 리눅스입니다. Docker Desktop이 WSL2에서 리눅스 이미지를 만들기 때문입니다. (개념 1)
>
> **ARM 노트북(Snapdragon)** 을 쓰신다면 이미지가 ARM64로 빌드됩니다. 이 경우 아키텍처를 `Linux/ARM64` 로 맞추거나, 빌드 시 아래처럼 플랫폼을 지정하세요.
> ```powershell
> docker build --platform linux/amd64 -t petclinic:v1 .
> ```

> ⚠️ 메모리를 1GB로 낮추지 마세요. Spring Boot + JVM은 1GB에서 OOM으로 죽습니다.

**2단계 — 컨테이너 정의:**

| 입력 항목 | 값 |
|----------|-----|
| 이름 | `petclinic` |
| 이미지 URI | `<계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic:v1` |
| 필수 컨테이너 | 예 |
| 포트 매핑 | 컨테이너 포트 **8080**, 프로토콜 TCP, 앱 프로토콜 HTTP |

**환경 변수 (일반):**

| 키 | 값 유형 | 값 |
|----|---------|-----|
| `SPRING_PROFILES_ACTIVE` | 값 | `mysql` |

> 📌 PetClinic은 `mysql` 프로파일이 활성화되어야 외부 DB를 씁니다. 없으면 내장 H2로 뜹니다.

**환경 변수 (시크릿에서 주입):**

"환경 변수 추가" 아래의 **"보안 암호에서 값 가져오기"** 옵션을 사용합니다.

| 키 | 값 유형 | 값 |
|----|---------|-----|
| `SPRING_DATASOURCE_URL` | **ValueFrom** | `<시크릿ARN>:url::` |
| `SPRING_DATASOURCE_USERNAME` | **ValueFrom** | `<시크릿ARN>:username::` |
| `SPRING_DATASOURCE_PASSWORD` | **ValueFrom** | `<시크릿ARN>:password::` |

> ⚠️ **ARN 뒤의 `:키이름::` 형식이 핵심입니다.** 콜론 2개로 끝납니다.
>
> 예시:
> ```
> arn:aws:secretsmanager:us-west-2:123456789012:secret:petclinic/db-AbCdEf:url::
> ```
>
> `::` 를 빼면 JSON 전체가 문자열로 들어가 접속이 실패합니다.

**로깅:**

| 입력 항목 | 값 |
|----------|-----|
| 로그 수집 사용 | ☑ 체크 |
| awslogs-group | `/ecs/petclinic` |
| awslogs-region | `us-west-2` |
| awslogs-stream-prefix | `ecs` |

**상태 검사 (Health check) — 권장:**

| 입력 항목 | 값 |
|----------|-----|
| 명령 | `CMD-SHELL,curl -f http://localhost:8080/ \|\| exit 1` |
| 간격 | 30 |
| 제한 시간 | 5 |
| 시작 기간 | **90** |
| 재시도 | 3 |

> ⚠️ **시작 기간(startPeriod)을 90초로 두세요.** Spring Boot는 뜨는 데 30~60초 걸립니다.

**생성** 클릭.

---

### 14-4. 태스크 정의 확인

📍 **PowerShell**

```powershell
aws ecs describe-task-definition --task-definition petclinic-task --query "taskDefinition.[family,revision,cpu,memory]" --output table
```

**기대 결과:**
```
------------------------------------------
|        DescribeTaskDefinition          |
+------------------+---+--------+--------+
|  petclinic-task  | 1 |  1024  |  2048  |
+------------------+---+--------+--------+
```

시크릿 연결도 확인합니다.

```powershell
aws ecs describe-task-definition --task-definition petclinic-task --query "taskDefinition.containerDefinitions[0].secrets[*].name" --output text
```

**기대 결과:**
```
SPRING_DATASOURCE_URL   SPRING_DATASOURCE_USERNAME   SPRING_DATASOURCE_PASSWORD
```

3개가 모두 보여야 합니다.

아키텍처도 확인합니다.

```powershell
aws ecs describe-task-definition --task-definition petclinic-task --query "taskDefinition.runtimePlatform" --output json
```

**기대 결과:**
```json
{
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
}
```

---

## 15. Step 11 — ALB 생성

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **AWS 콘솔** (EC2 → 로드 밸런서)

---

### 15-1. 대상 그룹 먼저 생성

**EC2 → 대상 그룹 → 대상 그룹 생성**

| 입력 항목 | 값 |
|----------|-----|
| 대상 유형 | **IP 주소** ← 중요 |
| 대상 그룹 이름 | `petclinic-tg` |
| 프로토콜/포트 | HTTP / **8080** |
| VPC | `petclinic-vpc` |
| 프로토콜 버전 | HTTP1 |

> ⚠️ **대상 유형은 반드시 "IP 주소"입니다.**
> Fargate 태스크는 EC2 인스턴스가 아니라 ENI(IP)를 가집니다.

**상태 검사:**

| 입력 항목 | 값 |
|----------|-----|
| 프로토콜 | HTTP |
| 경로 | `/` |
| 정상 임계값 | 2 |
| 비정상 임계값 | 3 |
| 제한 시간 | 5초 |
| 간격 | 30초 |
| 성공 코드 | 200 |

**다음** → 대상 등록 화면에서는 **아무것도 등록하지 않고** **대상 그룹 생성** 클릭.

> 📌 태스크는 ECS 서비스가 자동으로 등록합니다.

---

### 15-2. ALB 생성

**EC2 → 로드 밸런서 → 로드 밸런서 생성 → Application Load Balancer**

| 입력 항목 | 값 |
|----------|-----|
| 이름 | `petclinic-alb` |
| 체계 | **인터넷 경계(Internet-facing)** |
| IP 주소 유형 | IPv4 |
| VPC | `petclinic-vpc` |
| **매핑** | **퍼블릭 서브넷 2개 모두 선택** |
| 보안 그룹 | **`petclinic-alb-sg`** (default 해제) |

**리스너 및 라우팅:**

| 프로토콜 | 포트 | 기본 작업 |
|---------|------|----------|
| HTTP | 80 | **`petclinic-tg`** 로 전달 |

**로드 밸런서 생성** 클릭. **프로비저닝에 약 3분** 걸립니다.

---

### 15-3. DNS 이름 확인

📍 **PowerShell**

```powershell
aws elbv2 describe-load-balancers --names petclinic-alb --query "LoadBalancers[0].[State.Code,DNSName]" --output text
```

**기대 결과:**
```
active   petclinic-alb-1234567890.us-west-2.elb.amazonaws.com
```

`provisioning` 이면 2~3분 더 기다리세요.

변수에 담아 둡니다.

```powershell
$env:ALB_DNS = (aws elbv2 describe-load-balancers --names petclinic-alb --query "LoadBalancers[0].DNSName" --output text)
echo $env:ALB_DNS
```

---

### 15-4. 지금 접속하면 어떻게 되나?

```powershell
curl.exe -s -o NUL -w "%{http_code}`n" "http://$($env:ALB_DNS)"
```

**기대 결과:**
```
503
```

**503이 정상입니다.** 아직 대상 그룹에 등록된 태스크가 하나도 없기 때문입니다.

---

## 16. Step 12 — ECS 서비스 생성 (태스크 실행)

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **AWS 콘솔** (ECS)

**드디어 태스크가 뜨는 단계입니다.** 앞의 모든 준비가 여기서 검증됩니다.

---

### 16-1. 서비스 생성

**ECS → 클러스터 → `petclinic-cluster` → 서비스 탭 → 생성**

**환경:**

| 입력 항목 | 값 |
|----------|-----|
| 컴퓨팅 옵션 | **시작 유형** |
| 시작 유형 | **FARGATE** |
| 플랫폼 버전 | LATEST |

**배포 구성:**

| 입력 항목 | 값 |
|----------|-----|
| 애플리케이션 유형 | **서비스** |
| 패밀리 | `petclinic-task` |
| 개정 | `1(최신)` |
| 서비스 이름 | `petclinic-service` |
| 서비스 유형 | 복제본(Replica) |
| **원하는 태스크** | **2** |

**배포 옵션:**

| 입력 항목 | 값 |
|----------|-----|
| 배포 유형 | 롤링 업데이트 |
| **최소 실행 태스크 비율** | **100** |
| **최대 실행 태스크 비율** | **200** |

> 📌 이 두 값의 의미는 Step 15에서 자세히 다룹니다.

**네트워킹:**

| 입력 항목 | 값 |
|----------|-----|
| VPC | `petclinic-vpc` |
| **서브넷** | **프라이빗 서브넷 2개 모두** |
| 보안 그룹 | **기존 항목 선택 → `petclinic-app-sg`** |
| **퍼블릭 IP** | **끄기(Turned off)** ← 중요 |

> ⚠️ **퍼블릭 IP를 반드시 끄세요.**
> 프라이빗 서브넷에서 퍼블릭 IP를 켜면 라우팅이 없어 오히려 통신이 안 됩니다.

**로드 밸런싱:**

| 입력 항목 | 값 |
|----------|-----|
| 로드 밸런서 유형 | Application Load Balancer |
| 기존 로드 밸런서 사용 | **`petclinic-alb`** |
| 리스너 | 기존 리스너 사용 → `80:HTTP` |
| 대상 그룹 | 기존 대상 그룹 사용 → **`petclinic-tg`** |
| 상태 검사 유예 기간 | **120** |

> ⚠️ **상태 검사 유예 기간을 120초로 두세요.**
> Spring Boot가 뜨기 전에 ALB가 헬스체크를 시작하면, 준비 안 된 태스크를 비정상으로 판정해 죽입니다. 무한 재시작 루프에 빠집니다.

**서비스 오토스케일링:** 지금은 **사용 안 함**. Step 13에서 추가합니다.

**생성** 클릭.

---

### 16-2. 태스크가 뜨는지 지켜보기

**약 2~3분간 상태가 이렇게 바뀝니다:**

```
PROVISIONING  →  PENDING  →  ACTIVATING  →  RUNNING
```

📍 **PowerShell** — 실시간 감시 (리눅스 `watch` 대체)

```powershell
while ($true) {
  $r = aws ecs describe-services --cluster petclinic-cluster --services petclinic-service --query "services[0].[desiredCount,runningCount,pendingCount]" --output text
  Write-Host "$(Get-Date -Format 'HH:mm:ss')  desired/running/pending = $r"
  Start-Sleep -Seconds 10
}
```

> 📌 **PowerShell에는 `watch` 명령이 없습니다.** 위 `while` 루프가 그 역할을 합니다. (오해 ⓐ)

**기대 결과 (최종):**
```
06:12:30  desired/running/pending = 2       2       0
```

`desired=2, running=2, pending=0` 이면 성공입니다. `Ctrl + C` 로 종료하세요.

---

### 16-3. 태스크가 안 뜰 때 — 오류 진단표

**ECS → 서비스 → `petclinic-service` → 이벤트 탭** 에서 메시지를 확인하세요.

📍 **PowerShell** 로도 확인 가능합니다.

```powershell
$stopped = aws ecs list-tasks --cluster petclinic-cluster --desired-status STOPPED --query "taskArns[0]" --output text
aws ecs describe-tasks --cluster petclinic-cluster --tasks $stopped --query "tasks[0].stoppedReason" --output text
```

| 실제 오류 메시지 | 원인 | 돌아갈 곳 |
|----------------|------|----------|
| `CannotPullContainerError: ... i/o timeout` | VPC 엔드포인트 누락 (특히 **S3 Gateway**) | **Step 5 (9-2)** |
| `CannotPullContainerError: ... 403 Forbidden` | 태스크 실행 역할에 ECR 권한 없음 | **Step 9 (13-1)** |
| `CannotPullContainerError: ... no match for platform` | **ARM PC에서 빌드한 이미지** | **Step 10 (14-3)** 플랫폼 지정 |
| `ResourceInitializationError: unable to pull secrets` | 실행 역할에 `secretsmanager:GetSecretValue` 없음 | **Step 9 (13-2)** |
| `ResourceInitializationError: ... secretsmanager: RequestError` | Secrets Manager 엔드포인트 누락 | **Step 5 (9-1)** |
| `Task failed ELB health checks` | 유예 기간이 짧음 / 8080 포트 불일치 | **Step 12 (16-1)** |
| `exec /bin/sh: no such file or directory` | **Dockerfile이 CRLF** | **Step 2 (6-3)** |
| `OutOfMemoryError` (로그) | 메모리 1GB로 설정 | **Step 10 (14-3)** 2GB로 |

> 📌 **`exec /bin/sh: no such file or directory` 는 Windows 사용자 전용 오류입니다.**
> Dockerfile이 CRLF 줄바꿈으로 저장되면 이 오류가 납니다. **6-3** 으로 돌아가세요.

---

### 16-4. 컨테이너 로그 확인

📍 **PowerShell**

```powershell
aws logs tail /ecs/petclinic --follow --region us-west-2
```

**기대 결과 (정상):**
```
2026-07-10T05:12:33 ecs/petclinic/abc123  Started PetClinicApplication in 24.5 seconds
2026-07-10T05:12:33 ecs/petclinic/abc123  Tomcat started on port 8080
```

`Ctrl + C` 로 종료합니다.

**비정상 로그와 해결:**

| 로그에 보이는 메시지 | 원인 | 해결 |
|--------------------|------|------|
| `Communications link failure` | RDS 보안 그룹이 태스크를 막음 | **Step 4 (8-3)** |
| `Access denied for user 'petclinic'` | 시크릿의 비밀번호 불일치 | **Step 8 (12-1)** |
| `Unknown database 'petclinic'` | RDS 초기 DB 이름 미지정 | **Step 7 (11-2)** |
| `No suitable driver found for jdbc:mariadb` | URL 스킴 오류 | **Step 8 (12-1)** — `jdbc:mysql://` |
| `Table 'petclinic.owners' doesn't exist` | 프로파일 미적용 | **Step 10 (14-3)** — `SPRING_PROFILES_ACTIVE=mysql` |
| 로그 자체가 안 나옴 | CloudWatch Logs 엔드포인트 누락 | **Step 5 (9-1)** |

---

### 16-5. 접속 확인 (첫 성공 지점)

📍 **PowerShell**

```powershell
curl.exe -s -o NUL -w "%{http_code}`n" "http://$($env:ALB_DNS)"
```

**기대 결과:**
```
200
```

브라우저에서도 열어 보세요.

```powershell
Start-Process "http://$($env:ALB_DNS)"
```

**기대 결과:** PetClinic 홈 화면(강아지 그림과 "Welcome" 문구)이 보입니다.

> 🎉 **여기까지 왔으면 절반은 끝난 것입니다.**
> 컨테이너 이미지 → ECR → Fargate → ALB → RDS(시크릿 주입)까지 전 경로가 동작합니다.

**여전히 503이면:**

```powershell
$tgArn = aws elbv2 describe-target-groups --names petclinic-tg --query "TargetGroups[0].TargetGroupArn" --output text
aws elbv2 describe-target-health --target-group-arn $tgArn --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" --output table
```

**기대 결과:**
```
----------------------------------
|      DescribeTargetHealth      |
+---------------+----------------+
|  10.0.130.45  |  healthy       |
|  10.0.145.22  |  healthy       |
+---------------+----------------+
```

| 상태 | 의미 | 해결 |
|------|------|------|
| `healthy` | 정상 | ✅ |
| `initial` | 헬스체크 진행 중 | 1~2분 대기 |
| `unhealthy` | 헬스체크 실패 | **16-4** 로그 확인 |
| 목록이 비어 있음 | 태스크 미등록 | **Step 12 (16-1)** 로드밸런싱 설정 재확인 |

---

### 16-6. DB 연결 확인 (앱 기능 테스트)

브라우저에서 아래를 수행합니다.

1. `http://<ALB DNS>/owners/find` 접속
2. 검색창을 비운 채 **Find Owner** 클릭
3. 소유자 목록이 표시되면 **RDS 연결 성공**

**기대 결과:** George Franklin, Betty Davis 등 10명의 목록이 나옵니다.

PowerShell로 확인하려면:

```powershell
curl.exe -s "http://$($env:ALB_DNS)/owners?lastName=" | Select-String "Franklin"
```

**기대 결과:**
```
<td><a href="/owners/1">George Franklin</a></td>
```

> ⚠️ **목록이 비어 있거나 500 오류가 나면** DB 연결에 문제가 있습니다. **16-4** 로 돌아가세요.
---

## 17. Step 13 — 오토스케일링 설정

> ⏱ 예상 소요시간: 약 10분
> 📍 작업 위치: **AWS 콘솔** (ECS)

---

### 17-1. 오토스케일링 추가

**ECS → 클러스터 → `petclinic-cluster` → 서비스 → `petclinic-service` → 업데이트**

화면 아래쪽 **서비스 Auto Scaling** 섹션을 펼칩니다.

| 입력 항목 | 값 |
|----------|-----|
| 서비스 Auto Scaling 사용 | ☑ 체크 |
| **최소 태스크 수** | **2** |
| **최대 태스크 수** | **6** |

**조정 정책 추가** 클릭.

| 입력 항목 | 값 |
|----------|-----|
| 정책 유형 | **대상 추적(Target tracking)** |
| 정책 이름 | `petclinic-cpu-scaling` |
| ECS 서비스 지표 | **ECSServiceAverageCPUUtilization** |
| **목표 값** | **50** |
| 규모 축소 휴지 기간 | 300초 |
| 규모 확장 휴지 기간 | 60초 |
| 조정 축소 비활성화 | 체크 해제 |

> 📌 **목표값 50인 이유:** 실습에서 스케일 아웃을 빨리 보기 위해서입니다. 실무에서는 보통 70입니다.

> 📌 **확장 60초 / 축소 300초인 이유:** 늘릴 때는 빠르게(장애 방지), 줄일 때는 천천히(플래핑 방지) 하는 것이 원칙입니다.

**업데이트** 클릭.

---

### 17-2. 오토스케일링 등록 확인

📍 **PowerShell**

```powershell
aws application-autoscaling describe-scalable-targets --service-namespace ecs --query "ScalableTargets[*].[ResourceId,MinCapacity,MaxCapacity]" --output table
```

**기대 결과:**
```
------------------------------------------------------------------
|                    DescribeScalableTargets                     |
+----------------------------------------------+-----+-----------+
|  service/petclinic-cluster/petclinic-service |  2  |     6     |
+----------------------------------------------+-----+-----------+
```

정책도 확인합니다.

```powershell
aws application-autoscaling describe-scaling-policies --service-namespace ecs --query "ScalingPolicies[*].[PolicyName,TargetTrackingScalingPolicyConfiguration.TargetValue]" --output table
```

**기대 결과:**
```
------------------------------------------
|      DescribeScalingPolicies           |
+--------------------------+-------------+
|  petclinic-cpu-scaling   |  50.0       |
+--------------------------+-------------+
```

---

## 18. Step 14 — 부하 테스트로 스케일 아웃 관찰

> ⏱ 예상 소요시간: 약 30분 (관찰 시간 포함)
> 📍 작업 위치: **PowerShell**

---

### 18-1. 부하 생성 도구 설치

Windows에는 리눅스의 `hey` 나 `ab` 가 없습니다. 세 가지 방법이 있습니다.

**방법 ① — `hey` 바이너리 직접 다운로드 (권장)**

```powershell
cd $env:USERPROFILE
Invoke-WebRequest -Uri "https://hey-release.s3.us-east-2.amazonaws.com/hey_windows_amd64" -OutFile "hey.exe"
.\hey.exe -h
```

**기대 결과:**
```
Usage: hey [options...] <url>

Options:
  -n  Number of requests to run.
  -c  Number of workers to run concurrently.
```

**방법 ② — winget으로 설치**

```powershell
winget install --id=k6.k6 -e
k6 version
```

**방법 ③ — PowerShell 자체 부하 생성 (설치 불필요)**

`hey` 를 못 쓰는 경우 아래 스크립트를 씁니다. 효율은 낮지만 동작합니다.

```powershell
$url = "http://$($env:ALB_DNS)"
$jobs = 1..30 | ForEach-Object {
  Start-Job -ScriptBlock {
    param($u)
    $end = (Get-Date).AddMinutes(10)
    while ((Get-Date) -lt $end) {
      try { Invoke-WebRequest -Uri $u -UseBasicParsing -TimeoutSec 5 | Out-Null } catch {}
    }
  } -ArgumentList $url
}
Write-Host "부하 시작. 30개 작업 실행 중. 중지하려면: Get-Job | Stop-Job"
```

> ⚠️ **방법 ③은 CPU를 많이 씁니다.** 학생 노트북이 느려질 수 있습니다. 방법 ①을 우선 시도하세요.

---

### 18-2. 모니터링 창 먼저 열기 (중요)

**부하를 주기 전에** 관찰 창을 열어야 변화를 볼 수 있습니다.

**PowerShell 창 1** — 태스크 개수 감시:

```powershell
while ($true) {
  $r = aws ecs describe-services --cluster petclinic-cluster --services petclinic-service --query "services[0].[desiredCount,runningCount]" --output text
  Write-Host "$(Get-Date -Format 'HH:mm:ss')  desired/running = $r"
  Start-Sleep -Seconds 15
}
```

**브라우저** — CPU 사용률:

**ECS → 클러스터 → `petclinic-cluster` → 서비스 → `petclinic-service` → 지표 탭**

---

### 18-3. 부하 인가

**PowerShell 창 2** 에서 실행합니다.

```powershell
cd $env:USERPROFILE
.\hey.exe -z 10m -c 100 "http://$($env:ALB_DNS)/"
```

옵션 의미:

| 옵션 | 의미 |
|------|------|
| `-z 10m` | 10분간 계속 요청 |
| `-c 100` | 동시 연결 100개 |

> ⚠️ **`-c` 를 500 이상으로 올리지 마세요.** Windows의 동시 연결 제한과 로컬 CPU 한계에 걸립니다.

---

### 18-4. 관찰 — 무엇을 봐야 하는가

**시간 순서대로 이렇게 진행됩니다.**

```
0분    부하 시작.  태스크 2개.  CPU 20% → 상승
       │
1~3분  CPU 50% 초과.  CloudWatch 경보가 ALARM 상태로 전환
       │
3~5분  desiredCount 2 → 3 또는 4 로 증가
       │  ┌─ 창 1에서 "2  2" → "4  2" 로 바뀜
       │  └─ 새 태스크가 PROVISIONING → RUNNING
       │
5~7분  runningCount 도 따라 증가.  "4  4"
       │  ALB가 새 태스크를 대상 그룹에 자동 등록
       │
7~10분 태스크가 늘어 CPU가 분산.  50% 아래로 하강
       │
10분   부하 종료
       │
15~20분 규모 축소 휴지 300초 경과 후 desiredCount 감소
       │  "4  4" → "2  2"
```

> 📌 **스케일 아웃까지 3~5분이 걸립니다.** 즉시 반응하지 않습니다.
> CloudWatch 지표는 1분 단위로 수집되고, 경보는 여러 데이터 포인트를 보고 판단하기 때문입니다.

---

### 18-5. 스케일링 이력 확인

📍 **PowerShell**

```powershell
aws application-autoscaling describe-scaling-activities --service-namespace ecs --resource-id service/petclinic-cluster/petclinic-service --query "ScalingActivities[*].[StartTime,Description,StatusCode]" --output table
```

**기대 결과:**
```
------------------------------------------------------------------------
|                     DescribeScalingActivities                        |
+---------------------+---------------------------------+--------------+
|  2026-07-10T06:15:2 |  Setting desired count to 4.     |  Successful  |
|  2026-07-10T06:32:1 |  Setting desired count to 2.     |  Successful  |
+---------------------+---------------------------------+--------------+
```

두 줄(확장, 축소)이 모두 보이면 오토스케일링이 정상 동작한 것입니다.

**아무 줄도 없으면:**

| 원인 | 해결 |
|------|------|
| 부하가 부족해 CPU가 50%를 못 넘음 | `-c` 값을 200으로 올려 재시도 |
| 오토스케일링이 등록 안 됨 | **Step 13 (17-2)** 확인 |
| 최대값이 2로 설정됨 | **Step 13 (17-1)** 최대 6 확인 |

**부하 종료 (방법 ③을 썼다면):**

```powershell
Get-Job | Stop-Job
Get-Job | Remove-Job
```

---

## 19. Step 15 — 롤링 업데이트 (무중단 배포)

> ⏱ 예상 소요시간: 약 25분
> 📍 작업 위치: **PowerShell** → **AWS 콘솔**

---

### 19-1. 배포 파라미터의 의미 (먼저 이해하기)

Step 12에서 설정한 두 값이 여기서 작동합니다.

```
원하는 태스크 수 = 2

┌─ 최소 실행 태스크 비율 = 100%  → 배포 중 최소 2개는 살아 있어야 함
└─ 최대 실행 태스크 비율 = 200%  → 배포 중 최대 4개까지 띄울 수 있음

결과: 새 태스크 2개를 먼저 띄우고 → 헬스체크 통과 → 구 태스크 2개 제거
      순간 총 4개가 됨.  서비스 중단 0초.
```

**만약 50/100 이었다면:**

```
결과: 구 태스크 1개를 먼저 내리고 → 새 태스크 1개 띄움 → 반복
      순간 용량이 절반으로 떨어짐.  트래픽이 많으면 지연 발생.
```

| 설정 | 배포 중 최대 태스크 | 배포 중 최소 태스크 | 용량 손실 | 추가 비용 |
|------|-------------------|-------------------|----------|----------|
| **100 / 200** | 4 | 2 | **없음** | 배포 중 2배 |
| 50 / 100 | 2 | 1 | **50%** | 없음 |

**이 실습은 100/200 을 씁니다.**

---

### 19-2. v2 이미지 만들기

📍 **PowerShell** (`...\3tier\spring-petclinic`)

변경을 눈으로 확인할 수 있도록 화면 문구를 바꿉니다.

```powershell
cd C:\Users\zion3\Desktop\3tier\spring-petclinic
Select-String -Path src\main\resources\messages\messages.properties -Pattern "^welcome="
```

**기대 결과:**
```
messages.properties:1:welcome=Welcome
```

**PowerShell에는 `sed` 가 없습니다.** 아래로 치환합니다.

```powershell
$f = "src\main\resources\messages\messages.properties"
(Get-Content $f) -replace '^welcome=Welcome$', 'welcome=Welcome - Version 2' | Set-Content $f -Encoding UTF8
Select-String -Path $f -Pattern "^welcome="
```

**기대 결과:**
```
messages.properties:1:welcome=Welcome - Version 2
```

> ⚠️ **`-Encoding UTF8` 을 빠뜨리면** PowerShell이 `UTF-16` 으로 저장해 Maven 빌드가 깨집니다. (오해 ⓐ)

---

### 19-3. 빌드 및 푸시

```powershell
.\mvnw.cmd clean package -DskipTests
docker build -t petclinic:v2 .
docker tag petclinic:v2 "$($env:ECR_URI):v2"
docker push "$($env:ECR_URI):v2"
```

> ⚠️ **`$env:ECR_URI` 가 비어 있으면** PowerShell을 새로 연 것입니다. **10-2** 를 다시 실행하거나, **4-15** 프로필 설정을 확인하세요.

**기대 결과 (마지막 줄):**
```
v2: digest: sha256:yyyy... size: 1573
```

ECR에 두 태그가 모두 있는지 확인합니다.

```powershell
aws ecr describe-images --repository-name petclinic --query "imageDetails[*].imageTags[0]" --output text
```

**기대 결과:**
```
v2   v1
```

---

### 19-4. 태스크 정의 새 리비전 생성

**개념 4를 기억하세요.** 태스크 정의는 수정할 수 없습니다.

**ECS → 태스크 정의 → `petclinic-task` → 최신 리비전 선택 → 새 리비전 생성**

**컨테이너 정의의 이미지 URI만** 바꿉니다.

```
변경 전: <계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic:v1
변경 후: <계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic:v2
                                                          ↑ 여기만
```

**나머지는 전부 그대로 두세요.**

**생성** 클릭.

📍 **PowerShell** 확인:

```powershell
aws ecs describe-task-definition --task-definition petclinic-task --query "taskDefinition.[revision,containerDefinitions[0].image]" --output text
```

**기대 결과:**
```
2    123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic:v2
```

---

### 19-5. 무중단 확인용 감시 시작 (배포 전에 먼저)

**PowerShell 창 1** — 1초마다 접속해 응답 코드를 기록합니다.

```powershell
$url = "http://$($env:ALB_DNS)"
while ($true) {
  $code = curl.exe -s -o NUL -w "%{http_code}" $url
  $ts = Get-Date -Format 'HH:mm:ss'
  if ($code -eq "200") { Write-Host "$ts  $code" -ForegroundColor Green }
  else { Write-Host "$ts  $code  <<< 실패!" -ForegroundColor Red }
  Start-Sleep -Seconds 1
}
```

**기대 결과 (배포 전):** 초록색으로 계속 200이 찍힙니다.
```
06:40:01  200
06:40:02  200
06:40:03  200
```

**이 창을 계속 열어 둔 채로** 다음 단계를 진행합니다.

---

### 19-6. 서비스 업데이트 (배포 실행)

**PowerShell 창 2** 또는 콘솔에서 수행합니다.

**콘솔:** ECS → 서비스 → `petclinic-service` → **업데이트** → 개정을 `2(최신)` 로 변경 → **업데이트**

**PowerShell:**

```powershell
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --task-definition petclinic-task:2 --query "service.[serviceName,taskDefinition]" --output text
```

**기대 결과:**
```
petclinic-service   arn:aws:ecs:us-west-2:...:task-definition/petclinic-task:2
```

---

### 19-7. 배포 진행 관찰

**PowerShell 창 3:**

```powershell
while ($true) {
  Clear-Host
  aws ecs describe-services --cluster petclinic-cluster --services petclinic-service --query "services[0].deployments[*].[status,taskDefinition,desiredCount,runningCount]" --output table
  Start-Sleep -Seconds 10
}
```

**기대 결과 (배포 중):**
```
--------------------------------------------------------
|                  DescribeServices                    |
+----------+----------------------+-------+------------+
|  PRIMARY |  petclinic-task:2    |   2   |     1      |  ← 새 버전 뜨는 중
|  ACTIVE  |  petclinic-task:1    |   2   |     2      |  ← 구 버전 아직 살아 있음
+----------+----------------------+-------+------------+
```

**PRIMARY(새 버전)와 ACTIVE(구 버전)가 동시에 존재하는 순간**이 핵심입니다. 총 4개가 떠 있습니다.

**기대 결과 (배포 완료, 약 3~5분 후):**
```
+----------+----------------------+-------+------------+
|  PRIMARY |  petclinic-task:2    |   2   |     2      |
+----------+----------------------+-------+------------+
```

`ACTIVE` 줄이 사라지고 `PRIMARY` 하나만 남으면 완료입니다. `Ctrl + C` 로 종료하세요.

---

### 19-8. 무중단이었는지 검증

**PowerShell 창 1** 을 다시 봅니다.

**기대 결과:** 배포 중에도 초록색 200만 찍혀야 합니다.
```
06:42:01  200
06:45:33  200      ← 배포 중에도 계속 200
06:45:34  200
```

**빨간색 줄이 단 하나도 없어야 합니다.**

> 🎉 **한 줄도 실패하지 않았다면 무중단 배포에 성공한 것입니다.**

`Ctrl + C` 로 종료하세요.

**만약 빨간 줄이 나왔다면:**

| 원인 | 해결 |
|------|------|
| 최소 비율이 100 미만 | **Step 12 (16-1)** — 100으로 수정 |
| 헬스체크 유예 기간 부족 | **Step 12 (16-1)** — 120초로 수정 |
| 대상 그룹 등록 해제 지연 시간 짧음 | 대상 그룹 속성 → `deregistration_delay` 30초 이상 |

---

### 19-9. 브라우저에서 v2 확인

```powershell
Start-Process "http://$($env:ALB_DNS)"
```

**기대 결과:** 화면에 **"Welcome - Version 2"** 가 보입니다.

여전히 "Welcome"만 보이면 브라우저 캐시입니다. `Ctrl + Shift + R` 로 강제 새로고침하세요.

---

### 19-10. 롤백 실습

배포가 잘못됐을 때 되돌리는 방법입니다. **이전 리비전을 다시 지정**하면 끝입니다.

```powershell
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --task-definition petclinic-task:1
```

3~5분 뒤 브라우저를 새로고침하면 "Welcome"으로 돌아옵니다.

> 📌 **이것이 태스크 정의를 수정 불가로 만든 이유입니다.**
> 모든 리비전이 영구 보존되므로, 언제든 특정 시점으로 되돌릴 수 있습니다.

다시 v2로 올려 두고 다음 단계로 갑니다.

```powershell
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --task-definition petclinic-task:2
```

---

## 20. Step 16 — CodePipeline으로 CI/CD 자동화

> ⏱ 예상 소요시간: 약 40분
> 📍 작업 위치: **PowerShell** → **AWS 콘솔**

**지금까지는 이미지를 손으로 빌드해서 손으로 배포했습니다.** 이제 Git에 푸시하면 자동으로 되게 만듭니다.

---

### 20-1. 자동화될 흐름

```
Windows PC에서 git push
         │
         ▼
   ① CodePipeline이 감지 (Source 단계)
         │
         ▼
   ② CodeBuild가 실행 (Build 단계)  ← 리눅스 환경에서 돎
      ├─ mvn package
      ├─ docker build
      └─ ECR에 push
         │
         ▼
   ③ ECS가 새 이미지로 롤링 업데이트 (Deploy 단계)
```

> 📌 **CodeBuild는 AWS의 리눅스 컨테이너에서 실행됩니다.**
> 여러분의 Windows PC와 무관합니다. 그래서 `buildspec.yml` 은 리눅스 문법(bash)으로 씁니다.

---

### 20-2. GitHub 저장소 준비

📍 **PowerShell**

본인 GitHub 계정에 저장소를 만들고 코드를 올립니다.

```powershell
cd C:\Users\zion3\Desktop\3tier\spring-petclinic
git remote remove origin
git remote add origin https://github.com/<본인계정>/spring-petclinic.git
git add .
git commit -m "Add Dockerfile for ECS deployment"
git push -u origin main
```

**기대 결과:**
```
To https://github.com/yourname/spring-petclinic.git
 * [new branch]      main -> main
```

**인증 창이 뜨면** GitHub 계정으로 로그인합니다. (Git Credential Manager가 자동 처리)

---

### 20-3. buildspec.yml 작성

프로젝트 루트에 `buildspec.yml` 로 저장하세요.

```powershell
notepad buildspec.yml
```

아래를 붙여넣고 **모든 파일 (*.*)**, **UTF-8** 로 저장합니다.

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - echo Image tag is $IMAGE_TAG

  build:
    commands:
      - echo Building the application...
      - ./mvnw clean package -DskipTests
      - echo Building the Docker image...
      - docker build -t $ECR_URI:$IMAGE_TAG .
      - docker tag $ECR_URI:$IMAGE_TAG $ECR_URI:latest

  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $ECR_URI:$IMAGE_TAG
      - docker push $ECR_URI:latest
      - echo Writing image definitions file...
      - printf '[{"name":"petclinic","imageUri":"%s"}]' $ECR_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

> 📌 **여기서는 파이프(`|`) 방식이 정상 동작합니다.**
> CodeBuild는 리눅스에서 실행되므로 오해 ⓑ의 PowerShell 문제가 없습니다.
> 로컬 PowerShell에서만 `$pw` 변수 방식을 씁니다.

> 📌 **`./mvnw` 입니다. `mvnw.cmd` 가 아닙니다.** CodeBuild는 리눅스이기 때문입니다.

**각 단계가 하는 일:**

| 단계 | 하는 일 |
|------|--------|
| `pre_build` | ECR 로그인, 커밋 해시로 이미지 태그 생성 |
| `build` | Maven 빌드 → Docker 이미지 빌드 |
| `post_build` | ECR 푸시, `imagedefinitions.json` 생성 |

> 📌 **`imagedefinitions.json` 이 핵심입니다.**
> Deploy 단계가 이 파일을 읽어 "어느 컨테이너를 어느 이미지로 바꿀지" 판단합니다.
> `"name"` 값 `petclinic` 은 **태스크 정의의 컨테이너 이름과 정확히 일치**해야 합니다.

**줄바꿈을 LF로 변환합니다 (Windows 필수):**

```powershell
$c = (Get-Content buildspec.yml -Raw) -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\buildspec.yml", $c)
```

커밋하고 푸시합니다.

```powershell
git add buildspec.yml
git commit -m "Add buildspec for CodeBuild"
git push
```

---

### 20-4. CodeBuild 프로젝트 생성

**AWS 콘솔 → CodeBuild → 빌드 프로젝트 → 프로젝트 생성**

| 입력 항목 | 값 |
|----------|-----|
| 프로젝트 이름 | `petclinic-build` |
| 소스 공급자 | **GitHub** |
| 리포지토리 | 본인 저장소 연결 (OAuth 인증) |
| 환경 이미지 | 관리형 이미지 |
| 운영 체제 | Amazon Linux |
| 런타임 | Standard |
| 이미지 | `aws/codebuild/amazonlinux2-x86_64-standard:5.0` |
| **권한 있음(Privileged)** | **☑ 체크** ← 중요 |
| 서비스 역할 | 새 서비스 역할 (`codebuild-petclinic-build-service-role`) |
| Buildspec | buildspec 파일 사용 |

> ⚠️ **"권한 있음(Privileged)" 체크를 빠뜨리면 Docker 빌드가 실패합니다.**
> 오류: `Cannot connect to the Docker daemon`

**환경 변수 추가:**

| 이름 | 값 | 유형 |
|------|-----|------|
| `ECR_URI` | `<계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic` | 일반 텍스트 |

**프로젝트 생성** 클릭.

---

### 20-5. CodeBuild 역할에 ECR 권한 추가

**IAM → 역할 → `codebuild-petclinic-build-service-role` → 권한 추가 → 정책 연결**

`AmazonEC2ContainerRegistryPowerUser` 를 검색해 연결합니다.

📍 **PowerShell** 확인:

```powershell
aws iam list-attached-role-policies --role-name codebuild-petclinic-build-service-role --query "AttachedPolicies[*].PolicyName" --output text
```

**기대 결과:** 목록에 `AmazonEC2ContainerRegistryPowerUser` 가 포함됩니다.

---

### 20-6. 빌드 단독 테스트 (파이프라인 만들기 전)

**CodeBuild → `petclinic-build` → 빌드 시작**

**기대 결과 (약 5~8분 후):**
```
빌드 상태: 성공(Succeeded)
```

**단계별 로그를 확인하세요.**

| 단계 | 성공 표시 |
|------|----------|
| PRE_BUILD | `Login Succeeded` |
| BUILD | `BUILD SUCCESS` (Maven) → `naming to ...` (Docker) |
| POST_BUILD | `latest: digest: sha256:...` |

**실패 시:**

| 오류 메시지 | 원인 | 해결 |
|-----------|------|------|
| `Cannot connect to the Docker daemon` | 특권 모드 미체크 | **20-4** 프로젝트 편집 |
| `denied: ... not authorized to perform: ecr:*` | ECR 권한 없음 | **20-5** |
| `Permission denied: ./mvnw` | 실행 권한 없음 | 아래 명령 실행 |
| `buildspec.yml not found` | 파일이 저장소 루트에 없음 | **20-3** 커밋·푸시 확인 |
| `/bin/sh: bad interpreter` | **buildspec이 CRLF** | **20-3** LF 변환 |

**`Permission denied: ./mvnw` 해결 (Windows 사용자 필수):**

Windows에서 커밋하면 `mvnw` 의 실행 권한이 사라집니다.

```powershell
git update-index --chmod=+x mvnw
git commit -m "Make mvnw executable"
git push
```

---

### 20-7. CodePipeline 생성

**AWS 콘솔 → CodePipeline → 파이프라인 생성**

**1단계 — 설정:**

| 입력 항목 | 값 |
|----------|-----|
| 파이프라인 이름 | `petclinic-pipeline` |
| 실행 모드 | 대기 중(Queued) |
| 서비스 역할 | 새 서비스 역할 |

**2단계 — 소스:**

| 입력 항목 | 값 |
|----------|-----|
| 소스 공급자 | **GitHub(버전 2)** |
| 연결 | 새 연결 생성 → GitHub 앱 설치 승인 |
| 리포지토리 | `<본인계정>/spring-petclinic` |
| 브랜치 | `main` |
| 트리거 | 푸시 시 시작 |

**3단계 — 빌드:**

| 입력 항목 | 값 |
|----------|-----|
| 빌드 공급자 | AWS CodeBuild |
| 프로젝트 이름 | `petclinic-build` |

**4단계 — 배포:**

| 입력 항목 | 값 |
|----------|-----|
| 배포 공급자 | **Amazon ECS** |
| 클러스터 이름 | `petclinic-cluster` |
| 서비스 이름 | `petclinic-service` |
| 이미지 정의 파일 | `imagedefinitions.json` |

**파이프라인 생성** 클릭. 생성 즉시 첫 실행이 시작됩니다.

---

### 20-8. 전체 파이프라인 동작 확인

**CodePipeline → `petclinic-pipeline`**

**기대 결과 (약 10분 후):**
```
Source   ✓ 성공
Build    ✓ 성공
Deploy   ✓ 성공
```

---

### 20-9. 실제 자동 배포 시연

📍 **PowerShell**

코드를 바꾸고 푸시하기만 하면 됩니다.

```powershell
cd C:\Users\zion3\Desktop\3tier\spring-petclinic
$f = "src\main\resources\messages\messages.properties"
(Get-Content $f) -replace '^welcome=.*', 'welcome=Welcome - Version 3 (Auto Deployed)' | Set-Content $f -Encoding UTF8

git add .
git commit -m "Update welcome message to v3"
git push
```

**다른 창에서 무중단을 감시하면서:**

```powershell
$url = "http://$($env:ALB_DNS)"
while ($true) {
  $code = curl.exe -s -o NUL -w "%{http_code}" $url
  $ts = Get-Date -Format 'HH:mm:ss'
  if ($code -eq "200") { Write-Host "$ts  $code" -ForegroundColor Green }
  else { Write-Host "$ts  $code  <<< 실패!" -ForegroundColor Red }
  Start-Sleep -Seconds 2
}
```

**기대 결과:**

```
1분 후   CodePipeline의 Source 단계가 In Progress
3분 후   Build 단계 진행 (mvn + docker)
9분 후   Deploy 단계 시작 → ECS 롤링 업데이트
14분 후  브라우저에 "Welcome - Version 3 (Auto Deployed)" 표시
```

**그동안 화면은 계속 초록색 200이어야 합니다.**

> 🎉 **여기까지 성공하면 실습 전 과정이 완료된 것입니다.**
> `git push` 한 번으로 빌드·이미지 생성·무중단 배포가 자동 수행되었습니다.

**Deploy 단계가 실패하면:**

| 오류 메시지 | 원인 | 해결 |
|-----------|------|------|
| `The image definitions file cannot be found` | `imagedefinitions.json` 미생성 | **20-3** artifacts 확인 |
| `Invalid action configuration: container name` | JSON의 `name` 과 태스크 정의 컨테이너명 불일치 | **20-3** — `petclinic` 으로 통일 |
| `Deployment failed: tasks failed to start` | 새 이미지가 기동 실패 | **16-4** 로그 확인 |
---

## 21. 정상 동작 최종 확인

📍 **PowerShell**

---

### 21-1. 서비스 상태

```powershell
aws ecs describe-services --cluster petclinic-cluster --services petclinic-service --query "services[0].[status,desiredCount,runningCount,pendingCount]" --output text
```

**기대 결과:**
```
ACTIVE   2   2   0
```

---

### 21-2. 대상 그룹 상태

```powershell
$tgArn = aws elbv2 describe-target-groups --names petclinic-tg --query "TargetGroups[0].TargetGroupArn" --output text
aws elbv2 describe-target-health --target-group-arn $tgArn --query "TargetHealthDescriptions[*].TargetHealth.State" --output text
```

**기대 결과:**
```
healthy   healthy
```

---

### 21-3. 애플리케이션 응답

```powershell
curl.exe -s "http://$($env:ALB_DNS)/" | Select-String -Pattern "Welcome[^<]*"
```

**기대 결과:**
```
Welcome - Version 3 (Auto Deployed)
```

---

### 21-4. 오류 로그 일괄 확인

```powershell
aws logs tail /ecs/petclinic --since 30m --region us-west-2 | Select-String -Pattern "error|exception|fail" -CaseSensitive:$false
```

**기대 결과:** 아무것도 출력되지 않으면 정상입니다.

| 로그에 보이는 메시지 | 원인 | 해결 |
|--------------------|------|------|
| `Communications link failure` | RDS SG가 태스크 차단 | **Step 4 (8-3)** |
| `Access denied for user` | 시크릿 비밀번호 불일치 | **Step 8 (12-1)** |
| `Unknown database 'petclinic'` | RDS 초기 DB 미생성 | **Step 7 (11-2)** |
| `No suitable driver` | JDBC URL 스킴 오류 | **Step 8 (12-1)** |
| `Table ... doesn't exist` | 프로파일 미적용 | **Step 10 (14-3)** |
| `exec /bin/sh: no such file` | Dockerfile CRLF | **Step 2 (6-3)** |
| `OutOfMemoryError` | 메모리 부족 | **Step 10 (14-3)** |

---

## 22. 변경·롤백 절차

---

### 22-1. DB 비밀번호를 바꿀 때

```
① RDS에서 마스터 암호 변경          (아직 태스크는 구 암호를 갖고 있음)
       ↓
② Secrets Manager의 password 값 수정
       ↓
③ 서비스 강제 재배포                 (새 태스크가 새 암호를 읽어 감)
       ↓
④ 롤링 업데이트로 태스크 교체         (무중단)
```

> ⚠️ **①과 ② 사이에 재배포하면 안 됩니다.** 새 태스크가 구 암호로 접속을 시도해 실패합니다.

```powershell
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --force-new-deployment
```

> 📌 **시크릿 값만 바꾸면 자동으로 반영되지 않습니다.**
> 시크릿은 태스크가 **시작될 때 한 번만** 읽힙니다.

---

### 22-2. 이미지를 바꿀 때

```
① 새 태그로 빌드·푸시  (v3)
       ↓
② 태스크 정의 새 리비전 생성  (이미지 URI만 변경)
       ↓
③ 서비스 업데이트  (--task-definition petclinic-task:3)
```

CodePipeline을 쓰면 ①②③이 자동입니다.

---

### 22-3. 롤백

리비전 목록 확인:

```powershell
aws ecs list-task-definitions --family-prefix petclinic-task --query "taskDefinitionArns" --output text
```

원하는 리비전으로 되돌리기:

```powershell
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --task-definition petclinic-task:1
```

---

## 23. 보안 주의사항 (반드시 지킬 것)

| 번호 | 규칙 | 이유 |
|------|------|------|
| 1 | **DB 비밀번호를 태스크 정의의 `environment` 에 넣지 않는다** | 콘솔·CLI로 누구나 평문 조회 가능. 반드시 `secrets` (ValueFrom) 사용 |
| 2 | **Dockerfile에 자격증명을 넣지 않는다** | 이미지 레이어에 영구 기록됨. `docker history` 로 노출 |
| 3 | **buildspec.yml에 시크릿을 직접 쓰지 않는다** | Git에 커밋되어 영구 노출 |
| 4 | **컨테이너를 root로 실행하지 않는다** | 컨테이너 탈출 시 피해 확대. `USER app` 필수 |
| 5 | **보안 그룹 소스에 IP 대신 SG를 참조한다** | Fargate는 IP가 매번 바뀜 |
| 6 | **RDS 퍼블릭 액세스를 "아니요"로 둔다** | 인터넷에서 DB에 직접 접근 가능해짐 |
| 7 | **ECR 이미지 스캔을 켠다** | 알려진 CVE를 푸시 시점에 탐지 |
| 8 | **`latest` 태그만으로 배포하지 않는다** | 어느 커밋이 배포됐는지 추적 불가 |
| 9 | **태스크 실행 역할의 시크릿 권한을 특정 ARN으로 제한한다** | `Resource: "*"` 는 계정의 모든 시크릿을 읽음 |
| 10 | **VPC 엔드포인트 SG를 443만 열고 소스를 app-sg로 제한한다** | 0.0.0.0/0 은 VPC 내 모든 리소스가 API 호출 가능 |
| 11 | **AWS 액세스 키를 PowerShell 히스토리에 남기지 않는다** | `Get-Content (Get-PSReadlineOption).HistorySavePath` 로 조회 가능 |

**PowerShell 히스토리 확인 및 삭제 (Windows 전용):**

```powershell
# 히스토리 파일 위치 확인
(Get-PSReadlineOption).HistorySavePath

# 액세스 키가 남아 있는지 확인
Get-Content (Get-PSReadlineOption).HistorySavePath | Select-String "AKIA"

# 발견되면 히스토리 삭제
Remove-Item (Get-PSReadlineOption).HistorySavePath
```

---

## 24. 리소스 정리 (실습 종료 시 필수)

> ⚠️ **정리하지 않으면 계속 과금됩니다.**

**삭제는 생성의 역순입니다.**

```
① CodePipeline 삭제
② CodeBuild 프로젝트 삭제
③ ECS 서비스 삭제  (desiredCount를 0으로 먼저 내림)
④ ECS 클러스터 삭제
⑤ ALB 삭제 → 대상 그룹 삭제
⑥ RDS 삭제 (최종 스냅샷 없음)
⑦ VPC 엔드포인트 5개 삭제   ← 잊기 쉬움
⑧ Secrets Manager 시크릿 삭제 (강제 삭제)
⑨ ECR 리포지토리 삭제 (이미지 포함)
⑩ CloudWatch 로그 그룹 삭제
⑪ 보안 그룹 4개 삭제
⑫ VPC 삭제
⑬ IAM 역할 삭제
```

---

### 24-1. PowerShell 일괄 정리 스크립트

📍 **PowerShell**

```powershell
# ③ 서비스 축소 후 삭제
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --desired-count 0
Start-Sleep -Seconds 30
aws ecs delete-service --cluster petclinic-cluster --service petclinic-service --force

# ④ 클러스터 삭제
aws ecs delete-cluster --cluster petclinic-cluster

# ⑤ ALB 삭제
$albArn = aws elbv2 describe-load-balancers --names petclinic-alb --query "LoadBalancers[0].LoadBalancerArn" --output text
aws elbv2 delete-load-balancer --load-balancer-arn $albArn
Start-Sleep -Seconds 60

# 대상 그룹 삭제
$tgArn = aws elbv2 describe-target-groups --names petclinic-tg --query "TargetGroups[0].TargetGroupArn" --output text
aws elbv2 delete-target-group --target-group-arn $tgArn

# ⑥ RDS 삭제
aws rds delete-db-instance --db-instance-identifier petclinic-db --skip-final-snapshot --delete-automated-backups

# ⑧ 시크릿 강제 삭제
aws secretsmanager delete-secret --secret-id petclinic/db --force-delete-without-recovery

# ⑨ ECR 삭제
aws ecr delete-repository --repository-name petclinic --force

# ⑩ 로그 그룹 삭제
aws logs delete-log-group --log-group-name /ecs/petclinic
```

> 📌 **⑦ VPC 엔드포인트와 ⑫ VPC는 콘솔에서 지우는 편이 안전합니다.**
> RDS 삭제(약 5분)가 끝난 뒤에 진행하세요. 먼저 지우면 종속성 오류가 납니다.

---

### 24-2. 삭제 확인

```powershell
aws ecs list-clusters
aws elbv2 describe-load-balancers --query "LoadBalancers[*].LoadBalancerName"
aws rds describe-db-instances --query "DBInstances[*].DBInstanceIdentifier"
aws ec2 describe-vpc-endpoints --query "VpcEndpoints[*].ServiceName"
```

**기대 결과:** 모두 빈 배열 `[]` 이면 정리 완료입니다.

**다음 날 Cost Explorer에서 잔여 과금이 없는지 교차 확인하세요.**

---

### 24-3. 로컬 Docker 정리 (Windows 디스크 회수)

Docker Desktop은 이미지를 가상 디스크에 쌓아 둡니다. 실습이 끝나면 정리하세요.

```powershell
# 사용하지 않는 이미지·컨테이너·볼륨 일괄 삭제
docker system prune -a --volumes
```

`y` 를 입력해 확인합니다.

**기대 결과:**
```
Total reclaimed space: 3.2GB
```

WSL2 가상 디스크 자체를 줄이려면:

```powershell
wsl --shutdown
```

그 후 **설정 → 앱 → Docker Desktop → 고급 옵션** 에서 정리하거나, Docker Desktop을 완전히 제거합니다.

---

## 25. 완료 체크리스트 (인수인계용)

### Windows PC 환경 구축

- [ ] BIOS 가상화 활성화 확인
- [ ] WSL2 설치 및 버전 2 확인
- [ ] `.wslconfig` 메모리 제한 설정
- [ ] Docker Desktop 설치 (WSL2 백엔드)
- [ ] `docker run hello-world` 성공
- [ ] AWS CLI v2 설치
- [ ] Git for Windows 설치 + `core.autocrlf input`
- [ ] JDK 17 설치 + `JAVA_HOME` 등록
- [ ] `aws configure` (리전 `us-west-2`)
- [ ] `$PROFILE` 에 환경변수 등록

### PowerShell에서 한 일

- [ ] `spring-petclinic` 저장소 복제
- [ ] `Dockerfile` 작성 (`USER app`, **LF 줄바꿈**, 확장자 없음)
- [ ] `.dockerignore` 작성
- [ ] `.\mvnw.cmd clean package` 빌드 성공
- [ ] `docker run` 으로 로컬 200 확인
- [ ] ECR 로그인 (`$pw` 변수 방식)
- [ ] `buildspec.yml` 작성 (**LF 변환** 후 커밋)
- [ ] `git update-index --chmod=+x mvnw`
- [ ] `hey.exe` 로 부하 테스트

### AWS 콘솔에서 한 일

- [ ] VPC `petclinic-vpc` (NAT 없음, DNS 활성화)
- [ ] 퍼블릭·프라이빗 서브넷 각 2개
- [ ] 보안 그룹 4개 (`-sg` 접미어, SG 체이닝)
- [ ] VPC 엔드포인트 5개 (Interface 4 + Gateway 1)
- [ ] ECR 리포지토리 `petclinic`
- [ ] RDS `petclinic-db` (프라이빗, 퍼블릭 액세스 아니요)
- [ ] Secrets Manager `petclinic/db`
- [ ] IAM `ecsTaskExecutionRole-petclinic` + 인라인 정책
- [ ] ECS 클러스터 `petclinic-cluster`
- [ ] 태스크 정의 `petclinic-task` (1 vCPU / 2GB / **Linux X86_64**)
- [ ] ALB `petclinic-alb` + 대상 그룹 `petclinic-tg` (IP 유형)
- [ ] ECS 서비스 `petclinic-service` (100/200, 유예 120초)
- [ ] 오토스케일링 (min 2, max 6, CPU 50%)
- [ ] CodeBuild `petclinic-build` (특권 모드)
- [ ] CodePipeline `petclinic-pipeline`

### 검증 완료 항목

- [ ] ALB DNS로 200 응답
- [ ] `/owners/find` 에서 소유자 목록 조회
- [ ] 부하 테스트로 태스크 2 → 4 증가
- [ ] 부하 종료 후 4 → 2 감소
- [ ] 롤링 업데이트 중 503 발생 0건
- [ ] `git push` 로 자동 배포 성공

---

## 26. 자주 묻는 질문 (FAQ)

---

### Q1. `docker: The term 'docker' is not recognized` 오류가 납니다.

**A**: 원인이 세 가지입니다. 순서대로 확인하세요.

**① Docker Desktop이 설치되어 있는가?**

```powershell
Test-Path "C:\Program Files\Docker\Docker\Docker Desktop.exe"
```

`False` 면 미설치입니다. **4-6** 으로 가세요.

**② PowerShell을 새로 열었는가?**

설치 전에 열어둔 창은 PATH를 모릅니다. **PowerShell 창을 완전히 닫고 새로 여세요.** 이것이 가장 흔한 원인입니다.

**③ PATH에 등록됐는가?**

```powershell
$env:Path -split ';' | Select-String "Docker"
```

아무것도 안 나오면 **4-10** 으로 가세요.

---

### Q2. Docker Desktop이 시작되지 않고 고래가 계속 돕니다.

**A**: WSL2 문제입니다.

```powershell
wsl --status
```

`기본 버전: 2` 가 아니면:

```powershell
wsl --set-default-version 2
wsl --shutdown
```

그 후 Docker Desktop을 재시작합니다.

**여전히 안 되면 가상화가 꺼진 것입니다.** **4-1** 로 가서 BIOS를 확인하세요.

```powershell
Get-ComputerInfo -Property "HyperVRequirementVirtualizationFirmwareEnabled"
```

---

### Q3. ECR 로그인이 `400 Bad Request` 로 실패합니다.

**A**: **PowerShell 전용 문제입니다.** AWS 문서의 파이프 방식을 쓰셨을 것입니다.

```powershell
# ❌ 실패
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin ...

# ✅ 성공
$pw = aws ecr get-login-password --region us-west-2
docker login --username AWS --password $pw "$($env:ACCOUNT_ID).dkr.ecr.us-west-2.amazonaws.com"
```

파이프 앞부분이 비밀번호에 개행 문자를 덧붙여 전달하기 때문입니다.

**오해 ⓑ** 를 다시 읽으세요.

---

### Q4. 태스크가 `exec /bin/sh: no such file or directory` 로 죽습니다.

**A**: **Windows 사용자 전용 오류입니다.** Dockerfile이 CRLF 줄바꿈으로 저장됐습니다.

```powershell
$content = Get-Content Dockerfile -Raw
if ($content -match "`r`n") { "CRLF (문제)" } else { "LF (정상)" }
```

`CRLF` 면 변환하세요.

```powershell
$content = (Get-Content Dockerfile -Raw) -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\Dockerfile", $content)
docker build -t petclinic:v1 .
```

재발 방지:

```powershell
git config --global core.autocrlf input
```

---

### Q5. `docker build` 시 `Dockerfile: not found` 가 나옵니다.

**A**: 메모장이 `Dockerfile.txt` 로 저장했습니다.

```powershell
dir Dockerfile*
```

**기대 결과:** `Dockerfile` 만 보여야 합니다. `Dockerfile.txt` 가 보이면:

```powershell
Rename-Item Dockerfile.txt Dockerfile
```

**앞으로 방지하려면** 탐색기 → 보기 → 표시 → **파일 확장명** 을 체크해 두세요.

---

### Q6. 태스크가 계속 PENDING에서 멈춥니다.

**A**: 대부분 이미지를 못 받는 경우입니다.

```powershell
$stopped = aws ecs list-tasks --cluster petclinic-cluster --desired-status STOPPED --query "taskArns[0]" --output text
aws ecs describe-tasks --cluster petclinic-cluster --tasks $stopped --query "tasks[0].stoppedReason" --output text
```

| 메시지 | 해결 |
|--------|------|
| `CannotPullContainerError: ... i/o timeout` | **S3 Gateway 엔드포인트 누락** → **Step 5 (9-2)** |
| `CannotPullContainerError: ... no match for platform` | **ARM PC 문제** → 아래 **Q7** |
| `ResourceInitializationError` | **Step 9 (13-2)** 또는 **Step 5 (9-1)** |

---

### Q7. `no match for platform in manifest` 오류가 납니다. (ARM 노트북)

**A**: Snapdragon 등 ARM 기반 Windows PC에서 빌드하면 ARM64 이미지가 만들어집니다. Fargate는 기본이 X86_64입니다.

**해결 ① — 빌드 시 플랫폼 지정 (권장)**

```powershell
docker build --platform linux/amd64 -t petclinic:v1 .
docker tag petclinic:v1 "$($env:ECR_URI):v1"
docker push "$($env:ECR_URI):v1"
```

**해결 ② — 태스크 정의를 ARM64로**

**ECS → 태스크 정의 → 새 리비전 → 운영 체제/아키텍처 → `Linux/ARM64`**

현재 이미지 아키텍처를 확인하려면:

```powershell
docker inspect petclinic:v1 --format "{{.Architecture}}"
```

**기대 결과:** `amd64` (Fargate 기본과 일치)

---

### Q8. 시크릿을 만들었는데 `ResourceInitializationError` 로 죽습니다.

**A**: 두 가지 중 하나입니다.

**① 태스크 실행 역할에 권한이 없다**

```powershell
aws iam list-role-policies --role-name ecsTaskExecutionRole-petclinic --query "PolicyNames" --output text
```

`PetClinicSecretsAccess` 가 없으면 **Step 9 (13-2)** 로.

**② Secrets Manager VPC 엔드포인트가 없다**

```powershell
aws ec2 describe-vpc-endpoints --query "VpcEndpoints[?contains(ServiceName,'secretsmanager')].State" --output text
```

`available` 이 안 나오면 **Step 5 (9-1)** 로.

---

### Q9. 시크릿 값을 바꿨는데 앱이 여전히 옛날 비밀번호를 씁니다.

**A**: **시크릿은 태스크가 시작될 때 한 번만 읽힙니다.**

```powershell
aws ecs update-service --cluster petclinic-cluster --service petclinic-service --force-new-deployment
```

**22-1 절차**를 참고하세요.

---

### Q10. ALB에 접속하면 계속 503이 나옵니다.

**A**: 3단계로 좁히세요.

```powershell
# ① 태스크가 실행 중인가?
aws ecs describe-services --cluster petclinic-cluster --services petclinic-service --query "services[0].runningCount"
```

`0` 이면 **Q6** 으로.

```powershell
# ② 타깃이 등록됐는가?
$tgArn = aws elbv2 describe-target-groups --names petclinic-tg --query "TargetGroups[0].TargetGroupArn" --output text
aws elbv2 describe-target-health --target-group-arn $tgArn --query "TargetHealthDescriptions[*].TargetHealth.State" --output text
```

목록이 비었으면 **Step 12 (16-1)** 로드밸런싱 설정 누락.

`unhealthy` 면 **③** 으로.

**③ 헬스체크 실패 원인:**

| 확인 | 정상값 |
|------|--------|
| 대상 그룹 포트 | 8080 |
| 대상 그룹 유형 | IP 주소 |
| `petclinic-app-sg` 인바운드 | 8080 ← `petclinic-alb-sg` |
| 상태 검사 유예 기간 | 120초 |

---

### Q11. `curl` 명령이 이상하게 동작합니다.

**A**: PowerShell의 `curl` 은 `Invoke-WebRequest` 의 별칭입니다. **반드시 `curl.exe`** 를 쓰세요.

```powershell
# ❌ PowerShell alias — 옵션 문법이 다름
curl -s -o /dev/null http://example.com

# ✅ 실제 curl 실행 파일
curl.exe -s -o NUL -w "%{http_code}`n" http://example.com
```

리눅스의 `/dev/null` 은 Windows에서 `NUL` 입니다.

---

### Q12. `$env:ECR_URI` 가 비어 있다고 나옵니다.

**A**: PowerShell을 새로 열면 세션 변수가 사라집니다.

**임시 해결:**

```powershell
$env:ACCOUNT_ID = (aws sts get-caller-identity --query Account --output text)
$env:ECR_URI = "$($env:ACCOUNT_ID).dkr.ecr.us-west-2.amazonaws.com/petclinic"
$env:ALB_DNS = (aws elbv2 describe-load-balancers --names petclinic-alb --query "LoadBalancers[0].DNSName" --output text)
```

**영구 해결:** **4-15** 의 `$PROFILE` 설정을 하세요.

---

### Q13. CodeBuild에서 `Permission denied: ./mvnw` 가 나옵니다.

**A**: **Windows에서 커밋하면 실행 권한이 사라집니다.**

```powershell
git update-index --chmod=+x mvnw
git commit -m "Make mvnw executable"
git push
```

---

### Q14. CodeBuild에서 `/bin/sh: bad interpreter` 가 나옵니다.

**A**: `buildspec.yml` 이 CRLF로 저장됐습니다.

```powershell
$c = (Get-Content buildspec.yml -Raw) -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\buildspec.yml", $c)
git add buildspec.yml
git commit -m "Fix line endings"
git push
```

---

### Q15. 부하를 줬는데 태스크가 안 늘어납니다.

**A**: 세 가지를 확인하세요.

**① 시간을 충분히 기다렸는가?** 스케일 아웃까지 **3~5분**이 걸립니다.

**② CPU가 실제로 50%를 넘었는가?** ECS 콘솔 지표 탭에서 확인하세요.

```powershell
.\hey.exe -z 10m -c 200 "http://$($env:ALB_DNS)/"
```

**③ 오토스케일링이 등록됐는가?**

```powershell
aws application-autoscaling describe-scalable-targets --service-namespace ecs --query "ScalableTargets[0].MaxCapacity"
```

`6` 이 나와야 합니다.

---

### Q16. Docker Desktop이 메모리를 너무 많이 씁니다.

**A**: Docker Desktop VM은 1.2–2GB를 상시 점유합니다. 상한을 정하세요.

```powershell
notepad $env:USERPROFILE\.wslconfig
```

```ini
[wsl2]
memory=4GB
processors=2
autoMemoryReclaim=gradual
```

적용:

```powershell
wsl --shutdown
```

그 후 Docker Desktop을 재시작합니다.

---

## 27. 관련 리소스 이름표

### Windows 로컬 환경

| 구분 | 값 |
|------|-----|
| 작업 폴더 | `C:\Users\zion3\Desktop\3tier\spring-petclinic` |
| 터미널 | PowerShell |
| Docker 백엔드 | WSL2 |
| WSL 설정 파일 | `%USERPROFILE%\.wslconfig` |
| PowerShell 프로필 | `$PROFILE` |
| Maven 래퍼 | `.\mvnw.cmd` (리눅스는 `./mvnw`) |
| HTTP 도구 | `curl.exe` (`curl` 아님) |
| 부하 도구 | `hey.exe` |

### 네트워크

| 구분 | 이름 |
|------|------|
| VPC | `petclinic-vpc` |
| VPC CIDR | `10.0.0.0/16` |
| 퍼블릭 서브넷 1 | `petclinic-subnet-public1-us-west-2a` |
| 퍼블릭 서브넷 2 | `petclinic-subnet-public2-us-west-2b` |
| 프라이빗 서브넷 1 | `petclinic-subnet-private1-us-west-2a` |
| 프라이빗 서브넷 2 | `petclinic-subnet-private2-us-west-2b` |

### 보안 그룹

| 구분 | 이름 | 인바운드 |
|------|------|---------|
| ALB | `petclinic-alb-sg` | 80 ← `0.0.0.0/0` |
| 태스크 | `petclinic-app-sg` | 8080 ← `petclinic-alb-sg` |
| RDS | `petclinic-db-sg` | 3306 ← `petclinic-app-sg` |
| 엔드포인트 | `petclinic-endpoint-sg` | 443 ← `petclinic-app-sg` |

### VPC 엔드포인트

| 구분 | 서비스 이름 | 유형 |
|------|------------|------|
| ECR API | `com.amazonaws.us-west-2.ecr.api` | Interface |
| ECR Docker | `com.amazonaws.us-west-2.ecr.dkr` | Interface |
| Secrets Manager | `com.amazonaws.us-west-2.secretsmanager` | Interface |
| CloudWatch Logs | `com.amazonaws.us-west-2.logs` | Interface |
| S3 | `com.amazonaws.us-west-2.s3` | **Gateway** |

### 컨테이너

| 구분 | 이름 |
|------|------|
| ECR 리포지토리 | `petclinic` |
| ECR URI | `<계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic` |
| 이미지 태그 | `v1`, `v2`, `<커밋해시>` |
| 클러스터 | `petclinic-cluster` |
| 태스크 정의 패밀리 | `petclinic-task` |
| 컨테이너 이름 | `petclinic` |
| 서비스 | `petclinic-service` |
| 아키텍처 | `Linux / X86_64` |

### 데이터

| 구분 | 이름 |
|------|------|
| RDS 식별자 | `petclinic-db` |
| DB 서브넷 그룹 | `petclinic-db-subnet-group` |
| 초기 데이터베이스명 | `petclinic` |
| 마스터 사용자 | `petclinic` |
| 시크릿 이름 | `petclinic/db` |
| 시크릿 키 | `url`, `username`, `password` |

### IAM

| 구분 | 이름 |
|------|------|
| 태스크 실행 역할 | `ecsTaskExecutionRole-petclinic` |
| 인라인 정책 | `PetClinicSecretsAccess` |
| CodeBuild 역할 | `codebuild-petclinic-build-service-role` |

### 로드밸런서 · CI/CD

| 구분 | 이름 |
|------|------|
| ALB | `petclinic-alb` |
| 대상 그룹 | `petclinic-tg` (유형: **IP 주소**, 포트 8080) |
| 로그 그룹 | `/ecs/petclinic` |
| CodeBuild 프로젝트 | `petclinic-build` |
| CodePipeline | `petclinic-pipeline` |
| 이미지 정의 파일 | `imagedefinitions.json` |

### 환경변수

| 이름 | 출처 |
|------|------|
| `SPRING_PROFILES_ACTIVE` | 일반 환경변수 (`mysql`) |
| `SPRING_DATASOURCE_URL` | 시크릿 (`...:url::`) |
| `SPRING_DATASOURCE_USERNAME` | 시크릿 (`...:username::`) |
| `SPRING_DATASOURCE_PASSWORD` | 시크릿 (`...:password::`) |

---

## 28. PowerShell ↔ Bash 명령 대조표

인터넷에서 리눅스 명령을 찾았을 때 참고하세요.

| 목적 | Bash (리눅스) | **PowerShell (Windows)** |
|------|--------------|-------------------------|
| 환경변수 설정 | `export V=값` | `$env:V = "값"` |
| 환경변수 참조 | `$V` | `$env:V` |
| 변수 뒤 콜론 | `$V:v1` | `"$($env:V):v1"` |
| 줄 연결 | `\` | `` ` `` (백틱) |
| 파일 목록 | `ls -l` | `dir` / `Get-ChildItem` |
| 파일 읽기 | `cat f` | `Get-Content f` |
| 문자열 검색 | `grep 패턴` | `Select-String 패턴` |
| 문자열 치환 | `sed -i 's/a/b/' f` | `(Get-Content f) -replace 'a','b' \| Set-Content f` |
| HTTP 요청 | `curl URL` | `curl.exe URL` |
| null 장치 | `/dev/null` | `NUL` |
| 반복 감시 | `watch -n 5 cmd` | `while($true){cmd; Start-Sleep 5}` |
| 대기 | `sleep 30` | `Start-Sleep -Seconds 30` |
| Maven 래퍼 | `./mvnw` | `.\mvnw.cmd` |
| 화면 지우기 | `clear` | `Clear-Host` |
| 파일 존재 확인 | `test -f f` | `Test-Path f` |
| 이름 변경 | `mv a b` | `Rename-Item a b` |

---

## 29. 한눈에 보는 작업 순서

### Windows 환경 구축 (최초 1회)

```
1.  BIOS 가상화 확인 (작업 관리자 → 성능 → CPU)
2.  wsl --install → 재부팅
3.  Docker Desktop 설치 (WSL2 백엔드) → 재부팅
4.  docker run hello-world 성공 확인
5.  AWS CLI v2 / Git / JDK 17 설치
6.  git config --global core.autocrlf input
7.  aws configure (리전 us-west-2)
8.  $PROFILE 에 환경변수 등록
```

### 이미지 만들기

```
9.  git clone spring-petclinic
10. .\mvnw.cmd clean package -DskipTests
11. Dockerfile 작성  (확장자 없음, UTF-8, LF)
12. 줄바꿈 LF 확인      ← Windows 필수 검증
13. .dockerignore 작성
14. docker build -t petclinic:v1 .
15. docker run 으로 로컬 200 확인      ← 여기서 안 되면 클라우드도 안 됨
```

### AWS 인프라 준비

```
16. VPC 생성  (2AZ, NAT 없음, DNS 켜기)
17. 보안 그룹 4개  (①alb → ②app → ③db → ④endpoint 순서)
18. VPC 엔드포인트 5개  (Interface 4 + S3 Gateway 1)
19. ECR 리포지토리 petclinic
20. RDS petclinic-db  (프라이빗, 초기 DB명 petclinic)
21. Secrets Manager petclinic/db
22. IAM ecsTaskExecutionRole-petclinic  (+ 시크릿 인라인 정책)
```

### 이미지 배포

```
23. $pw = aws ecr get-login-password ...    ← PowerShell 전용 방식
24. docker login --password $pw ...
25. docker tag / docker push
```

### 서비스 기동

```
26. ECS 클러스터 petclinic-cluster
27. CloudWatch 로그 그룹 /ecs/petclinic
28. 태스크 정의 petclinic-task  (1vCPU/2GB, Linux X86_64, 시크릿 3개)
29. 대상 그룹 petclinic-tg  (IP 유형, 8080)
30. ALB petclinic-alb  (퍼블릭 서브넷 2개)
31. ECS 서비스 petclinic-service  (프라이빗, 퍼블릭IP 끄기, 유예 120초)
32. ALB DNS로 200 확인            ← 첫 번째 성공 지점
```

### 오토스케일링 · 무중단 배포

```
33. 오토스케일링  (min2 / max6 / CPU 50%)
34. hey.exe 로 부하 → 태스크 2→4 증가 관찰  (3~5분 소요)
35. 부하 종료 → 4→2 감소 관찰  (5분 후)
36. v2 이미지 빌드·푸시
37. 태스크 정의 리비전 2 생성
38. 서비스 업데이트 → 200 감시하며 503 0건 확인
39. 리비전 1로 롤백 실습 → 다시 2로
```

### CI/CD 자동화

```
40. GitHub에 코드 푸시
41. buildspec.yml 작성 → LF 변환 → 커밋
42. git update-index --chmod=+x mvnw    ← Windows 필수
43. CodeBuild petclinic-build  (특권 모드 필수)
44. CodeBuild 역할에 ECR 권한 추가
45. 빌드 단독 테스트 → 성공 확인
46. CodePipeline petclinic-pipeline
47. 코드 수정 → git push → 자동 배포 확인   ← 최종 성공 지점
```

### 정리

```
48. 서비스 desiredCount 0 → 서비스 삭제
49. 클러스터 · ALB · 대상그룹 · RDS 삭제
50. VPC 엔드포인트 5개 삭제      ← 잊기 쉬움
51. 시크릿 · ECR · 로그그룹 삭제
52. 보안그룹 · VPC · IAM 역할 삭제
53. docker system prune -a --volumes    ← 로컬 디스크 회수
54. 다음 날 Cost Explorer 확인
```

---

## 30. 참고 문서

- [Docker Desktop Windows 설치](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker Desktop WSL2 백엔드](https://docs.docker.com/desktop/features/wsl/)
- [Amazon ECS 개발자 안내서](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/Welcome.html)
- [AWS Fargate 시작하기](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/getting-started-fargate.html)
- [태스크 정의에서 민감한 데이터 지정](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/specifying-sensitive-data.html)
- [Amazon ECR 프라이빗 레지스트리 인증](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html)
- [Amazon ECS 인터페이스 VPC 엔드포인트](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/vpc-endpoints.html)
- [서비스 Auto Scaling](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/service-auto-scaling.html)
- [CodePipeline으로 ECS 배포 자동화](https://docs.aws.amazon.com/ko_kr/codepipeline/latest/userguide/ecs-cd-pipeline.html)
- [Spring PetClinic 저장소](https://github.com/spring-projects/spring-petclinic)

---

> **문서 버전**: 2.0 (Windows 11 + PowerShell)
> **작성 기준**: `us-west-2` 리전, Docker Desktop WSL2 백엔드, ECS Fargate LATEST
> **⚠️ 확인 필요**: AWS 콘솔 UI와 Docker Desktop UI는 수시로 변경됩니다. 메뉴 명칭이 다르면 각 단계의 CLI 명령으로 대체하거나 공식 문서를 확인하세요.
