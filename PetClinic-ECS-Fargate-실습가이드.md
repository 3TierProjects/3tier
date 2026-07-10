# Spring PetClinic — ECS Fargate 컨테이너 배포 가이드

> **이 문서는 AWS 실습 경험 3개월 미만인 학생이 그대로 따라 하면 됩니다.**
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
| **총 소요시간** | 약 4~5시간 (Step 1 ~ Step 11) |
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
```

이 표를 계속 곁에 두고 읽으세요. 용어가 나올 때마다 대응되는 사물을 떠올리면 훨씬 쉽습니다.

---

## 1. 전체 흐름 (그림)

```
                        [ 개발자 PC ]
                              │
          ① Dockerfile 작성 · 이미지 빌드
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

**번호 순서가 곧 실습 순서입니다.** 지금 몇 번을 하고 있는지 항상 확인하세요.

---

## 2. 핵심 개념 8가지 (꼭 읽고 넘어가기)

> 이 8가지를 모르면 에러가 났을 때 **어디가 문제인지 찾을 수 없습니다.**
> 각 개념마다 "이 프로젝트에서 쓰는 실제 이름"을 적어 두었습니다. 이 이름은 문서 끝 리소스 이름표와 반드시 일치합니다.

---

### 개념 1 — 컨테이너 이미지 (밀키트)

애플리케이션 + 실행에 필요한 모든 것(JRE, 톰캣, 라이브러리)을 하나로 묶어 **어디서 실행해도 똑같이 동작하게** 만든 파일 묶음입니다.

- **이 프로젝트에서 쓰는 실제 이름**: `petclinic:v1`, `petclinic:v2`
- **만드는 곳**: 로컬 PC (Dockerfile로 빌드)

---

### 개념 2 — ECR (밀키트 창고)

AWS가 제공하는 컨테이너 이미지 저장소입니다. Docker Hub의 AWS 전용 사설 버전이라고 보면 됩니다.

- **이 프로젝트에서 쓰는 실제 이름**: `petclinic`
- **콘솔 위치**: **AWS 콘솔 → ECR (Elastic Container Registry) → 리포지토리**
- **주소 형식**: `<계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic`

---

### 개념 3 — 태스크 정의 (주방 매뉴얼)

컨테이너를 **어떻게 실행할지 적어 둔 설계도**입니다. 어떤 이미지를 쓸지, CPU/메모리는 얼마나 줄지, 환경변수는 무엇인지를 담습니다.

**중요:** 태스크 정의는 **한 번 만들면 수정할 수 없습니다.** 바꾸려면 새 리비전(revision)을 만듭니다. `petclinic-task:1`, `petclinic-task:2` 처럼 번호가 붙습니다.

- **이 프로젝트에서 쓰는 실제 이름**: `petclinic-task`
- **콘솔 위치**: **AWS 콘솔 → ECS → 태스크 정의**

---

### 개념 4 — 태스크와 서비스 (조리 중인 그릇 / 점장 지시)

**태스크(Task)** 는 실제로 실행 중인 컨테이너 한 벌입니다. 죽으면 그냥 사라집니다.

**서비스(Service)** 는 "태스크를 항상 N개 유지하라"는 규칙입니다. 태스크가 죽으면 서비스가 자동으로 새로 띄웁니다.

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

### 개념 5 — Fargate (통째로 빌린 주방)

EC2 인스턴스를 만들지 않고 컨테이너를 실행하는 방식입니다. CPU와 메모리만 지정하면 AWS가 알아서 실행 공간을 마련합니다.

| 구분 | EC2 시작 유형 | **Fargate** |
|------|--------------|-------------|
| EC2 인스턴스 관리 | 내가 함 | **안 함** |
| OS 패치 | 내가 함 | **안 함** |
| 과금 단위 | 인스턴스 시간 | **태스크의 vCPU·메모리 사용 시간** |
| 이 실습 | ❌ | ✅ **이걸 씁니다** |

---

### 개념 6 — 태스크 실행 역할 vs 태스크 역할 (열쇠 두 개)

**이 둘의 혼동이 이 실습에서 가장 많이 터지는 오류입니다.** 반드시 구분하세요.

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

### 개념 7 — VPC 엔드포인트 (창고로 가는 전용 통로)

Fargate 태스크는 프라이빗 서브넷에 있습니다. 그런데 ECR에서 이미지를 받아야 하고, Secrets Manager를 읽어야 합니다. 이 둘은 **VPC 바깥의 AWS 서비스**입니다.

밖으로 나가는 방법은 두 가지입니다.

| 방법 | 동작 | 비용 | 이 실습 |
|------|------|------|---------|
| NAT Gateway | 인터넷으로 나갔다가 돌아옴 | 시간당 + 데이터 처리량 | ❌ |
| **VPC 엔드포인트** | **AWS 내부망으로 직접 감** | 시간당 (NAT보다 저렴) | ✅ |

```
[ NAT 방식 ]
태스크 → NAT Gateway → 인터넷 게이트웨이 → 인터넷 → ECR
                                          ↑ 실제로는 AWS 안인데 밖으로 한 바퀴

[ VPC 엔드포인트 방식 ]
태스크 → VPC 엔드포인트 → ECR
         ↑ AWS 내부망으로 바로 감. 인터넷 안 나감.
```

**이 실습에서 만들 엔드포인트 (총 5개):**

| 엔드포인트 | 유형 | 왜 필요한가 |
|-----------|------|------------|
| `com.amazonaws.us-west-2.ecr.api` | Interface | ECR 인증 |
| `com.amazonaws.us-west-2.ecr.dkr` | Interface | 이미지 레이어 pull |
| `com.amazonaws.us-west-2.secretsmanager` | Interface | 시크릿 조회 |
| `com.amazonaws.us-west-2.logs` | Interface | CloudWatch 로그 전송 |
| `com.amazonaws.us-west-2.s3` | **Gateway** | **ECR 이미지 레이어가 실제로는 S3에 저장됨** |

> ⚠️ **S3 Gateway 엔드포인트를 빼먹으면 이미지 pull이 실패합니다.**
> "ECR인데 왜 S3가 필요하지?"라고 생각하기 쉽지만, ECR은 메타데이터만 관리하고 실제 이미지 레이어는 S3에 저장합니다. 이것이 초보자가 가장 많이 걸리는 함정입니다.

---

### 개념 8 — 오토스케일링 두 단계

Fargate에서는 **태스크 개수만** 늘리면 됩니다. EC2 인스턴스를 늘릴 필요가 없습니다.

```
CPU 사용률 70% 초과가 계속됨
         ↓
CloudWatch 경보 발생
         ↓
Application Auto Scaling이 desired count 증가
         ↓
서비스가 새 태스크를 띄움
         ↓
ALB가 새 태스크를 타깃 그룹에 자동 등록
```

- **이 프로젝트에서 쓰는 정책**: 대상 추적(Target Tracking), 지표 `ECSServiceAverageCPUUtilization`, 목표값 `70`

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

> **중요**: 아래 5가지 착각 중 하나라도 가지고 시작하면 **중간에 반드시 막힙니다.** 먼저 읽으세요.

---

### 오해 ① "Spring PetClinic은 Spring Boot다"

**절반만 맞습니다.** PetClinic에는 두 갈래가 있습니다.

| 저장소 | 정체 | 산출물 | 이 실습 |
|--------|------|--------|---------|
| `spring-projects/spring-petclinic` | **Spring Boot** (내장 톰캣) | 실행 가능 `.jar` | ✅ **이걸 씁니다** |
| `spring-petclinic/spring-framework-petclinic` | Spring MVC (외부 톰캣) | `.war` | ❌ |

**이 문서는 Spring Boot 버전을 기준으로 합니다.** 이미지 안에 톰캣을 따로 설치하지 않습니다. `.jar` 하나를 `java -jar`로 실행하면 끝입니다.

이전에 EC2 + 외부 톰캣으로 실습했다면 **그 방식은 여기서 쓰지 않습니다.** WAR를 `webapps/`에 넣는 절차가 없습니다.

---

### 오해 ② "환경변수는 아무 이름이나 써도 된다"

Spring Boot는 정해진 이름의 환경변수만 인식합니다. 아래 표를 벗어나면 무시됩니다.

| 사용 가능 ✅ | 사용 불가 ❌ |
|-------------|-------------|
| `SPRING_DATASOURCE_URL` | `DB_URL` |
| `SPRING_DATASOURCE_USERNAME` | `DB_USER` |
| `SPRING_DATASOURCE_PASSWORD` | `DB_PASS` |
| `SPRING_PROFILES_ACTIVE` | `PROFILE` |

**근거:** Spring Boot는 `application.properties`의 키 `spring.datasource.url`을 환경변수 `SPRING_DATASOURCE_URL`로 자동 매핑합니다. 점(`.`)이 밑줄(`_`)로, 소문자가 대문자로 바뀌는 규칙입니다. 임의의 이름은 이 규칙에 걸리지 않아 그냥 무시됩니다.

---

### 오해 ③ "프라이빗 서브넷에 태스크를 두면 인터넷이 필요 없다"

**태스크 자체는 인터넷이 필요 없지만, 태스크를 "시작하는" ECS 에이전트는 ECR·Secrets Manager에 접근해야 합니다.**

NAT도 없고 VPC 엔드포인트도 없으면 태스크가 다음 오류로 실패합니다.

```
CannotPullContainerError: ... i/o timeout
```

이 실습은 NAT 대신 **VPC 엔드포인트 5개**로 해결합니다. (개념 7 참조)

---

### 오해 ④ "보안 그룹 이름에 sg- 를 붙인다"

`sg-`, `vpc-`, `subnet-` 은 **AWS가 예약한 접두어**입니다. 리소스 이름에 쓸 수 없습니다.

| 사용 가능 ✅ | 사용 불가 ❌ |
|-------------|-------------|
| `petclinic-alb-sg` | `sg-petclinic-alb` |
| `petclinic-app-sg` | `sg-app` |

**접미어(`-sg`)를 쓰세요.**

---

### 오해 ⑤ "보안 그룹 소스에 IP를 적으면 된다"

Fargate 태스크는 죽고 살아날 때마다 **IP가 바뀝니다.** IP로 규칙을 만들면 다음 배포에서 깨집니다.

**보안 그룹은 다른 보안 그룹을 소스로 참조할 수 있습니다(SG 체이닝).** 이 실습에서는 반드시 이 방식을 씁니다.

```
[ 나쁨 ]  RDS SG 인바운드 3306 ← 소스: 10.0.2.15/32   (태스크 IP)
                                       ↑ 태스크 재시작하면 무효

[ 좋음 ]  RDS SG 인바운드 3306 ← 소스: petclinic-app-sg  (보안 그룹 참조)
                                       ↑ IP가 바뀌어도 항상 유효
```

---

## 4. 사전 준비 체크리스트

작업 전에 아래를 모두 확인하세요. (⏱ 약 20분)

- [ ] AWS 계정에 로그인 가능
- [ ] IAM 사용자에 관리자 권한 (실습용)
- [ ] AWS CLI v2 설치됨
- [ ] Docker 설치되어 실행 중
- [ ] Git 설치됨
- [ ] Java 17 설치됨 (로컬 빌드용)
- [ ] 리전이 `us-west-2` 인지 확인

---

### 4-1. AWS CLI 확인

📍 작업 위치: **로컬 PC** (터미널)

```bash
aws --version
```

**기대 결과:**
```
aws-cli/2.15.30 Python/3.11.8 Linux/6.5.0 exe/x86_64.ubuntu.22
```

`aws-cli/1.x` 가 나오면 v1입니다. v2로 재설치하세요.
`command not found` 가 나오면 CLI가 없습니다. 아래로 설치하세요.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

### 4-2. 인증 정보 확인

```bash
aws sts get-caller-identity --region us-west-2
```

**기대 결과:**
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/student01"
}
```

> 📌 **여기 나오는 `Account` 값(12자리 숫자)을 메모하세요.** 이후 ECR 주소에 계속 씁니다.
> 이 문서에서는 `<계정ID>` 로 표기합니다. 여러분의 실제 숫자로 바꿔 넣으세요.

**오류가 나면:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `Unable to locate credentials` | 자격증명 미설정 | `aws configure` 실행 |
| `The security token included in the request is invalid` | 액세스 키 만료·오타 | 키 재발급 후 `aws configure` |

---

### 4-3. Docker 확인

```bash
docker --version
docker ps
```

**기대 결과:**
```
Docker version 25.0.3, build 4debf41
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

두 번째 명령이 **헤더만 출력되고 에러가 없으면 정상**입니다. (실행 중인 컨테이너가 없는 상태)

**오류가 나면:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `Cannot connect to the Docker daemon` | Docker 데몬 미실행 | `sudo systemctl start docker` |
| `permission denied while trying to connect` | 현재 사용자가 docker 그룹에 없음 | `sudo usermod -aG docker $USER` 후 **재로그인** |

---

### 4-4. Java 확인

```bash
java -version
```

**기대 결과:**
```
openjdk version "17.0.10" 2024-01-16
OpenJDK Runtime Environment (build 17.0.10+7-Ubuntu-1ubuntu122.04)
```

`17` 이 아니면 PetClinic 빌드가 실패합니다. Java 17을 설치하세요.

```bash
sudo apt update && sudo apt install -y openjdk-17-jdk   # Ubuntu
```

---

## 5. Step 1 — 소스 코드 내려받기

> ⏱ 예상 소요시간: 약 5분
> 📍 작업 위치: **로컬 PC** (홈 디렉터리)

---

### 5-1. 저장소 복제

```bash
cd ~
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

**기대 결과:**
```
Cloning into 'spring-petclinic'...
remote: Enumerating objects: 8000, done.
...
Resolving deltas: 100% (4000/4000), done.
```

---

### 5-2. 이것이 Spring Boot 버전인지 확인

오해 ①을 실제로 확인하는 단계입니다.

```bash
grep -m1 "spring-boot-starter-parent" pom.xml -A 2
```

**기대 결과:**
```xml
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
```

이 출력이 보이면 **Spring Boot 버전이 맞습니다.** 아무것도 안 나오면 잘못된 저장소를 받은 것입니다. Step 5-1로 돌아가세요.

---

### 5-3. 로컬 빌드

```bash
./mvnw clean package -DskipTests
```

> ⏱ 처음 실행하면 의존성을 내려받느라 **5~10분** 걸립니다. 정상입니다.

**기대 결과 (마지막 부분):**
```
[INFO] BUILD SUCCESS
[INFO] Total time:  02:31 min
```

빌드 산출물을 확인합니다.

```bash
ls -lh target/*.jar
```

**기대 결과:**
```
-rw-rw-r-- 1 user user 56M Jul 10 14:22 target/spring-petclinic-3.2.0-SNAPSHOT.jar
```

> 📌 **이 jar 파일 이름을 메모하세요.** Dockerfile에서 씁니다. 버전 번호는 다를 수 있습니다.

**실패 시:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `Unsupported class file major version` | Java 버전 불일치 | **4-4로 돌아가** Java 17 설치 |
| `Could not resolve dependencies` | 네트워크 차단 | 프록시·방화벽 확인 후 재시도 |

---

## 6. Step 2 — Dockerfile 작성

> ⏱ 예상 소요시간: 약 10분
> 📍 작업 위치: **로컬 PC** (`~/spring-petclinic` 폴더)

---

### 6-1. Dockerfile 생성

프로젝트 루트(`pom.xml`이 있는 위치)에 `Dockerfile` 이라는 이름의 파일을 만듭니다. **확장자 없습니다.**

```bash
cd ~/spring-petclinic
```

아래 내용을 `Dockerfile` 로 저장하세요.

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

> ⚠️ **`USER app` 줄을 빼지 마세요.** 컨테이너를 root로 실행하는 것은 보안 사고의 흔한 원인입니다. Step 12 보안 섹션에서 다시 다룹니다.

파일이 올바른 위치에 저장됐는지 확인합니다.

```bash
ls -l Dockerfile pom.xml
```

**기대 결과:** 두 파일이 **같은 폴더**에 보여야 합니다.
```
-rw-rw-r-- 1 user user  287 Jul 10 14:30 Dockerfile
-rw-rw-r-- 1 user user 8421 Jul 10 14:10 pom.xml
```

`Dockerfile: No such file or directory` 가 나오면 다른 폴더에 저장한 것입니다. `~/spring-petclinic` 안에 만드세요.

---

### 6-2. .dockerignore 생성

불필요한 파일이 이미지에 들어가지 않게 막습니다. 같은 폴더에 `.dockerignore` 로 저장하세요.

```
.git
.mvn
src
*.md
target/*
!target/*.jar
```

마지막 두 줄이 핵심입니다. `target/` 전체를 제외하되, `.jar` 파일만은 예외로 포함시킵니다.

---

### 6-3. 로컬에서 이미지 빌드

```bash
docker build -t petclinic:v1 .
```

> 명령 끝의 점(`.`)은 "현재 폴더를 빌드 컨텍스트로 삼는다"는 뜻입니다. 빠뜨리면 실패합니다.

**기대 결과 (마지막 부분):**
```
 => exporting to image
 => => writing image sha256:a1b2c3...
 => => naming to docker.io/library/petclinic:v1
```

---

### 6-4. 이미지 확인

```bash
docker images petclinic
```

**기대 결과:**
```
REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
petclinic    v1    a1b2c3d4e5f6   10 seconds ago   215MB
```

크기가 200MB 내외면 정상입니다. 800MB를 넘으면 `.dockerignore`가 적용되지 않은 것입니다. **6-2로 돌아가세요.**

---

### 6-5. 로컬에서 실행해 보기 (중요)

**AWS에 올리기 전에 여기서 반드시 확인하세요.** 여기서 안 되면 클라우드에서도 안 됩니다.

```bash
docker run --rm -p 8080:8080 petclinic:v1
```

**기대 결과 (약 20초 후):**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
...
Started PetClinicApplication in 8.324 seconds
```

새 터미널을 열어 접속을 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

**기대 결과:**
```
200
```

`200`이 나오면 성공입니다. 원래 터미널에서 `Ctrl+C` 로 종료하세요.

> 📌 지금은 DB 설정 없이 내장 H2 데이터베이스로 뜹니다. RDS 연결은 Step 8에서 붙입니다.
> **처음부터 RDS를 붙이지 않는 이유:** 태스크가 안 뜰 때 원인이 이미지 문제인지 DB 문제인지 구분하기 위해서입니다.

**실패 시:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `no main manifest attribute` | jar가 실행 가능 형태가 아님 | **5-3으로 돌아가** `./mvnw package` 재실행 |
| `COPY failed: no source files` | `target/*.jar` 없음 | **5-3으로 돌아가** 빌드 |
| `port is already allocated` | 8080 이미 사용 중 | `-p 8081:8080` 으로 변경 |

---

## 7. Step 3 — VPC 신규 구축

> ⏱ 예상 소요시간: 약 25분
> 📍 작업 위치: **AWS 콘솔** (VPC)

> ⚠️ **리전이 `us-west-2` (오레곤)인지 콘솔 우측 상단에서 먼저 확인하세요.**
> 리전이 다르면 이후 모든 단계가 어긋납니다.

---

### ✅ 이미 실습용 VPC가 있다면? (먼저 읽기)

**Step 3 전체를 건너뛸 수 있습니다.** 단, 아래 조건을 모두 만족해야 합니다.

| 확인 항목 | 필요 조건 |
|----------|----------|
| 퍼블릭 서브넷 | 2개 이상, 서로 다른 AZ |
| 프라이빗 서브넷 | 2개 이상, 서로 다른 AZ |
| DNS 호스트 이름 | 활성화됨 |
| DNS 확인 | 활성화됨 |

| 확인 결과 | 다음 행동 |
|----------|----------|
| 모두 만족 | ✅ Step 3 통과 → **Step 4로** |
| 서브넷이 1개 AZ에만 있음 | ⚠️ ALB가 최소 2개 AZ를 요구함 → 서브넷 추가 |
| DNS 설정 꺼짐 | ⚠️ VPC 엔드포인트가 동작 안 함 → **7-5 참고** |

---

### 7-1. VPC 생성

**AWS 콘솔 → VPC → VPC 생성** 을 클릭합니다.

**"VPC 등"(VPC and more)** 을 선택하세요. 서브넷·라우팅 테이블을 한 번에 만들어 줍니다.

| 입력 항목 | 값 |
|----------|-----|
| 이름 태그 자동 생성 | `petclinic` |
| IPv4 CIDR 블록 | `10.0.0.0/16` |
| IPv6 CIDR 블록 | IPv6 CIDR 블록 없음 |
| 테넌시 | 기본값 |
| 가용 영역(AZ) 수 | **2** |
| 퍼블릭 서브넷 수 | **2** |
| 프라이빗 서브넷 수 | **2** |
| NAT 게이트웨이 | **없음** ← 중요 |
| VPC 엔드포인트 | **없음** ← 나중에 직접 만듭니다 |
| DNS 호스트 이름 활성화 | ☑ 체크 |
| DNS 확인 활성화 | ☑ 체크 |

> ⚠️ **NAT 게이트웨이를 "없음"으로 두는 것이 이 실습의 핵심입니다.**
> NAT는 시간당 요금이 붙습니다. 대신 Step 5에서 VPC 엔드포인트를 만듭니다.

**VPC 생성** 을 클릭합니다.

**기대 결과:** 생성 완료 화면에 아래 리소스가 모두 초록색 체크로 표시됩니다.
```
✓ VPC
✓ 서브넷 (4개)
✓ 라우팅 테이블 (3개)
✓ 인터넷 게이트웨이
```

---

### 7-2. 서브넷 이름 확인

**VPC → 서브넷** 으로 이동해 4개가 만들어졌는지 확인합니다.

**기대 결과:**

| 이름 | CIDR | 가용 영역 | 용도 |
|------|------|----------|------|
| `petclinic-subnet-public1-us-west-2a` | 10.0.0.0/20 | us-west-2a | ALB |
| `petclinic-subnet-public2-us-west-2b` | 10.0.16.0/20 | us-west-2b | ALB |
| `petclinic-subnet-private1-us-west-2a` | 10.0.128.0/20 | us-west-2a | 태스크·RDS |
| `petclinic-subnet-private2-us-west-2b` | 10.0.144.0/20 | us-west-2b | 태스크·RDS |

> 📌 **CIDR 값은 조금 다를 수 있습니다.** 이름 패턴과 AZ 분포만 맞으면 됩니다.
> 이 4개 이름을 메모하세요. 이후 계속 선택해야 합니다.

---

### 7-3. 프라이빗 서브넷에 인터넷 경로가 없는지 확인

이것이 "NAT 없음"이 제대로 적용됐는지 검증하는 단계입니다.

**VPC → 서브넷 → `petclinic-subnet-private1-us-west-2a` 선택 → 라우팅 테이블 탭**

**기대 결과:**

| 대상 | 대상(Target) |
|------|-------------|
| `10.0.0.0/16` | local |

**`0.0.0.0/0` 행이 없어야 정상입니다.** 있으면 NAT나 IGW가 붙은 것이므로, VPC를 삭제하고 7-1부터 다시 하세요.

---

### 7-4. 퍼블릭 서브넷에는 인터넷 경로가 있는지 확인

**VPC → 서브넷 → `petclinic-subnet-public1-us-west-2a` 선택 → 라우팅 테이블 탭**

**기대 결과:**

| 대상 | 대상(Target) |
|------|-------------|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | `igw-xxxxxxxx` |

**`0.0.0.0/0` → IGW 행이 있어야 정상입니다.** 없으면 ALB가 인터넷에서 접근 불가합니다.

---

### 7-5. DNS 설정 확인

VPC 엔드포인트가 동작하려면 **반드시** 켜져 있어야 합니다.

**VPC → VPC → `petclinic-vpc` 선택 → 세부 정보 탭**

**기대 결과:**
```
DNS 호스트 이름:  활성화됨
DNS 확인:        활성화됨
```

하나라도 "비활성화됨"이면 **작업 → VPC 설정 편집** 에서 체크하세요.

> ⚠️ 이 설정이 꺼져 있으면 Step 5의 VPC 엔드포인트를 만들어도 태스크가 ECR을 찾지 못합니다. 원인을 찾기 매우 어려운 오류이므로 지금 확인하세요.
---

## 8. Step 4 — 보안 그룹 4개 생성

> ⏱ 예상 소요시간: 약 20분
> 📍 작업 위치: **AWS 콘솔** (VPC → 보안 그룹)

**만들 순서가 중요합니다.** 뒤의 그룹이 앞의 그룹을 참조하므로, 반드시 아래 순서대로 만드세요.

```
① petclinic-alb-sg       (참조 없음)
② petclinic-app-sg       ← ①을 참조
③ petclinic-db-sg        ← ②를 참조
④ petclinic-endpoint-sg  ← ②를 참조
```

> ⚠️ **오해 ④를 기억하세요.** 이름은 `-sg` **접미어**입니다. `sg-` 로 시작하면 AWS가 거부합니다.

---

### 8-1. ALB용 보안 그룹

**VPC → 보안 그룹 → 보안 그룹 생성**

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-alb-sg` |
| 설명 | `Allow HTTP from internet to ALB` |
| VPC | `petclinic-vpc` |

**인바운드 규칙 추가:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| HTTP | TCP | 80 | `0.0.0.0/0` |

**아웃바운드 규칙:** 기본값(모든 트래픽 허용) 유지.

**보안 그룹 생성** 클릭. 생성된 ID(`sg-0abc...`)를 메모하세요.

---

### 8-2. 태스크용 보안 그룹

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-app-sg` |
| 설명 | `Allow 8080 from ALB only` |
| VPC | `petclinic-vpc` |

**인바운드 규칙 추가:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| 사용자 지정 TCP | TCP | **8080** | **`petclinic-alb-sg`** ← 보안 그룹 선택 |

> 📌 소스 칸에 IP를 적지 마세요. **드롭다운에서 `petclinic-alb-sg` 를 선택**합니다.
> 검색창에 `petclinic-alb` 를 입력하면 나타납니다. (오해 ⑤ 참조)

**아웃바운드 규칙:** 기본값 유지. (엔드포인트·RDS로 나가야 하므로 열어 둡니다)

---

### 8-3. RDS용 보안 그룹

| 입력 항목 | 값 |
|----------|-----|
| 보안 그룹 이름 | `petclinic-db-sg` |
| 설명 | `Allow 3306 from app tasks only` |
| VPC | `petclinic-vpc` |

**인바운드 규칙 추가:**

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

**인바운드 규칙 추가:**

| 유형 | 프로토콜 | 포트 | 소스 |
|------|---------|------|------|
| HTTPS | TCP | **443** | **`petclinic-app-sg`** |

> ⚠️ **443입니다. 80이 아닙니다.** VPC 엔드포인트는 AWS API를 호출하므로 HTTPS만 씁니다.
> 이 규칙을 빠뜨리면 Step 9에서 태스크가 이미지를 못 받습니다.

---

### 8-5. 4개 모두 만들어졌는지 확인

📍 작업 위치: **로컬 PC** (터미널)

```bash
aws ec2 describe-security-groups \
  --region us-west-2 \
  --filters "Name=group-name,Values=petclinic-*" \
  --query 'SecurityGroups[*].[GroupName,GroupId]' \
  --output table
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

4개가 모두 보이지 않으면 빠진 것을 다시 만드세요.

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

**개념 7을 다시 읽고 오세요.** 왜 5개인지, 왜 S3가 필요한지 이해하지 못하면 오류가 났을 때 손을 못 씁니다.

---

### 9-1. 인터페이스 엔드포인트 4개 (반복 작업)

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

> ⚠️ **보안 그룹에서 `default` 를 반드시 해제하세요.** 기본으로 선택되어 있습니다.
> `petclinic-endpoint-sg` 만 남겨야 합니다.

> ⚠️ **서브넷은 프라이빗 2개**입니다. 퍼블릭을 고르면 태스크가 접근하지 못합니다.

각 엔드포인트가 `사용 가능(Available)` 상태가 될 때까지 **2~3분** 기다립니다.

---

### 9-2. 게이트웨이 엔드포인트 1개 (S3)

**이것만 절차가 다릅니다.** 서브넷·보안 그룹을 고르지 않고 **라우팅 테이블**을 고릅니다.

**VPC → 엔드포인트 → 엔드포인트 생성**

| 입력 항목 | 값 |
|----------|-----|
| 이름 태그 | `petclinic-s3-ep` |
| 서비스 | `com.amazonaws.us-west-2.s3` |
| **유형** | **Gateway** ← Interface 아님 |
| VPC | `petclinic-vpc` |
| **라우팅 테이블** | **프라이빗 서브넷의 라우팅 테이블 선택** |
| 정책 | 전체 액세스 |

> 📌 라우팅 테이블 목록에서 이름에 `private` 이 들어간 것을 고르세요.
> 어느 것인지 모르겠으면 **VPC → 서브넷 → 프라이빗 서브넷 선택 → 라우팅 테이블 탭**에서 ID를 확인하세요.

> ⚠️ **이 단계를 빠뜨리면 Step 9에서 `CannotPullContainerError`가 발생합니다.**
> ECR의 실제 이미지 데이터는 S3에 저장되기 때문입니다. (개념 7 참조)

---

### 9-3. 5개 모두 확인

📍 작업 위치: **로컬 PC**

```bash
aws ec2 describe-vpc-endpoints \
  --region us-west-2 \
  --query 'VpcEndpoints[*].[ServiceName,VpcEndpointType,State]' \
  --output table
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

| 상태 | 의미 | 해결 |
|------|------|------|
| `pending` | 생성 중 | 2~3분 더 기다림 |
| `failed` | 생성 실패 | 삭제 후 **9-1부터 재시도** |
| 목록에 없음 | 안 만들어짐 | **9-1 또는 9-2 재수행** |

---

## 10. Step 6 — ECR 리포지토리 생성 및 이미지 푸시

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **AWS 콘솔** → **로컬 PC**

---

### 10-1. 리포지토리 생성 (콘솔)

**AWS 콘솔 → ECR → 리포지토리 → 리포지토리 생성**

| 입력 항목 | 값 |
|----------|-----|
| 표시 여부 설정 | **프라이빗** |
| 리포지토리 이름 | `petclinic` |
| 태그 변경 불가능 | 비활성화 (실습에서 v1→v2 덮어쓰기 가능하게) |
| 이미지 스캔 | 활성화 (푸시 시 취약점 스캔) |
| 암호화 | AES-256 |

**리포지토리 생성** 클릭.

**기대 결과:** 목록에 `petclinic` 이 나타나고, URI 열에 아래 형식의 주소가 보입니다.
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic
```

> 📌 **이 URI 전체를 복사해 두세요.** 앞으로 계속 씁니다.

---

### 10-2. 계정 ID를 변수로 저장 (편의)

📍 작업 위치: **로컬 PC**

매번 12자리 숫자를 치지 않도록 변수에 넣습니다.

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGION=us-west-2
export ECR_URI=${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/petclinic

echo $ECR_URI
```

**기대 결과:**
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic
```

> ⚠️ **터미널을 닫으면 이 변수는 사라집니다.** 새 터미널을 열면 이 명령을 다시 실행하세요.

---

### 10-3. ECR 로그인

```bash
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com
```

**기대 결과:**
```
Login Succeeded
```

**실패 시:**

| 메시지 | 원인 | 해결 |
|--------|------|------|
| `Cannot perform an interactive login from a non TTY device` | 파이프(`|`) 누락 | 명령 전체를 그대로 복사 |
| `no basic auth credentials` | 로그인 안 됨 | 이 명령 재실행 |
| `AccessDeniedException` | IAM 권한 부족 | `AmazonEC2ContainerRegistryFullAccess` 정책 확인 |

---

### 10-4. 이미지에 ECR 주소 태그 붙이기

로컬 이미지 `petclinic:v1` 에 ECR 주소를 붙여야 푸시할 수 있습니다.

```bash
docker tag petclinic:v1 ${ECR_URI}:v1
```

확인합니다.

```bash
docker images | grep petclinic
```

**기대 결과:** 두 줄이 보입니다. IMAGE ID가 같습니다(같은 이미지에 이름만 두 개).
```
123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic   v1   a1b2c3d4e5f6   ...
petclinic                                                v1   a1b2c3d4e5f6   ...
```

---

### 10-5. 푸시

```bash
docker push ${ECR_URI}:v1
```

**기대 결과 (마지막 줄):**
```
v1: digest: sha256:xxxx... size: 1573
```

---

### 10-6. ECR에 올라갔는지 확인

```bash
aws ecr describe-images \
  --repository-name petclinic \
  --region $REGION \
  --query 'imageDetails[*].[imageTags[0],imageSizeInBytes]' \
  --output table
```

**기대 결과:**
```
------------------------------
|       DescribeImages       |
+------+---------------------+
|  v1  |  215000000          |
+------+---------------------+
```

콘솔에서도 **ECR → petclinic → 이미지** 에서 `v1` 태그를 확인할 수 있습니다.

---

## 11. Step 7 — RDS MariaDB 생성

> ⏱ 예상 소요시간: 약 20분 (생성 대기 10분 포함)
> 📍 작업 위치: **AWS 콘솔** (RDS)

---

### 11-1. DB 서브넷 그룹 생성

RDS는 어느 서브넷에 놓일지 미리 지정해야 합니다.

**AWS 콘솔 → RDS → 서브넷 그룹 → DB 서브넷 그룹 생성**

| 입력 항목 | 값 |
|----------|-----|
| 이름 | `petclinic-db-subnet-group` |
| 설명 | `Private subnets for PetClinic RDS` |
| VPC | `petclinic-vpc` |
| 가용 영역 | `us-west-2a`, `us-west-2b` **둘 다 선택** |
| 서브넷 | **프라이빗 서브넷 2개 선택** |

> ⚠️ **퍼블릭 서브넷을 고르지 마세요.** DB가 인터넷에 노출됩니다.

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
| 마스터 암호 | `ChangeMe2026!` (실습용, 실제로는 강한 암호 사용) |
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
| 자동 백업 | 비활성화 (실습이므로) |
| 삭제 방지 | 비활성화 |

> ⚠️ **"초기 데이터베이스 이름"을 비워 두면 DB 스키마가 만들어지지 않습니다.**
> 추가 구성(Additional configuration) 섹션을 펼쳐서 반드시 `petclinic` 을 입력하세요.

**데이터베이스 생성** 클릭. **생성에 약 10분 걸립니다.**

---

### 11-3. 엔드포인트 주소 확인

상태가 `사용 가능(Available)` 이 되면 확인합니다.

**RDS → 데이터베이스 → `petclinic-db` → 연결 & 보안 탭**

**기대 결과:**
```
엔드포인트: petclinic-db.abcdefghij.us-west-2.rds.amazonaws.com
포트: 3306
```

> 📌 **이 엔드포인트 주소 전체를 메모하세요.** Step 8 시크릿에 넣습니다.

CLI로도 확인할 수 있습니다.

```bash
aws rds describe-db-instances \
  --db-instance-identifier petclinic-db \
  --region us-west-2 \
  --query 'DBInstances[0].[DBInstanceStatus,Endpoint.Address]' \
  --output text
```

**기대 결과:**
```
available    petclinic-db.abcdefghij.us-west-2.rds.amazonaws.com
```

`creating` 이 나오면 아직 생성 중입니다. 몇 분 더 기다리세요.

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
                                    ↑ Git에 커밋되면 영구 노출

[ 좋음 ]  Secrets Manager에서 런타임에 주입
   secrets:
     - SPRING_DATASOURCE_PASSWORD ← arn:aws:secretsmanager:...:password
                                    ↑ 태스크 정의에는 "주소"만 있음
                                    ↑ 실제 값은 시작할 때 ECS가 가져옴
```

---

### 12-1. 시크릿 생성

**AWS 콘솔 → Secrets Manager → 새 보안 암호 저장**

**1단계 — 보안 암호 유형:**

| 입력 항목 | 값 |
|----------|-----|
| 유형 | **다른 유형의 보안 암호** |

> 📌 "Amazon RDS 데이터베이스에 대한 자격 증명"을 고르지 마세요. 그 유형은 키 이름이 자동으로 정해져 Spring Boot가 인식하지 못합니다.

**키/값 쌍**에 아래 3개를 입력합니다. (평문 탭이 아니라 키/값 탭)

| 키 | 값 |
|----|-----|
| `url` | `jdbc:mysql://<RDS엔드포인트>:3306/petclinic` |
| `username` | `petclinic` |
| `password` | `ChangeMe2026!` |

`<RDS엔드포인트>` 자리에 11-3에서 메모한 주소를 넣으세요. 예:
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

**3단계 — 교체 구성:** 자동 교체 **비활성화** (실습이므로)

**저장** 클릭.

---

### 12-2. 시크릿 ARN 확인

**Secrets Manager → `petclinic/db` → 보안 암호 세부 정보**

**기대 결과:**
```
보안 암호 ARN: arn:aws:secretsmanager:us-west-2:123456789012:secret:petclinic/db-AbCdEf
```

> 📌 **끝의 6자리 임의 문자(`-AbCdEf`)까지 포함한 전체 ARN을 메모하세요.**
> 이 문자는 시크릿마다 다릅니다. 빼먹으면 태스크 정의에서 시크릿을 못 찾습니다.

CLI로도 확인 가능합니다.

```bash
aws secretsmanager describe-secret \
  --secret-id petclinic/db \
  --region us-west-2 \
  --query 'ARN' --output text
```

**기대 결과:**
```
arn:aws:secretsmanager:us-west-2:123456789012:secret:petclinic/db-AbCdEf
```

---

### 12-3. 값이 제대로 들어갔는지 확인

```bash
aws secretsmanager get-secret-value \
  --secret-id petclinic/db \
  --region us-west-2 \
  --query 'SecretString' --output text
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

**개념 6을 다시 읽고 오세요.** 이 단계에서 만드는 것은 **태스크 실행 역할**입니다. 태스크 역할이 아닙니다.

---

### 13-1. 역할 생성

**AWS 콘솔 → IAM → 역할 → 역할 생성**

**1단계 — 신뢰할 수 있는 엔터티 선택:**

| 입력 항목 | 값 |
|----------|-----|
| 신뢰할 수 있는 엔터티 유형 | **AWS 서비스** |
| 서비스 또는 사용 사례 | **Elastic Container Service** |
| 사용 사례 | **Elastic Container Service Task** |

> ⚠️ **"Elastic Container Service Task"** 를 고르세요. 그냥 "Elastic Container Service"가 아닙니다.
> 잘못 고르면 신뢰 정책의 주체가 `ecs.amazonaws.com` 이 되어 태스크가 이 역할을 맡지 못합니다.

**2단계 — 권한 추가:**

검색창에 `AmazonECSTaskExecutionRolePolicy` 를 입력해 체크합니다.

이 정책이 주는 권한:
- ECR에서 이미지 pull
- CloudWatch Logs에 로그 그룹·스트림 생성

**3단계 — 이름 지정:**

| 입력 항목 | 값 |
|----------|-----|
| 역할 이름 | `ecsTaskExecutionRole-petclinic` |

**역할 생성** 클릭.

---

### 13-2. 시크릿 읽기 권한 추가 (인라인 정책)

**`AmazonECSTaskExecutionRolePolicy` 에는 Secrets Manager 권한이 없습니다.** 직접 추가해야 합니다.

> ⚠️ **이 단계를 빠뜨리는 것이 이 실습에서 가장 많이 발생하는 오류입니다.**
> 태스크가 `ResourceInitializationError` 로 실패합니다.

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

> 📌 `<계정ID>` 를 실제 12자리 숫자로 바꾸세요.
> 끝의 `-*` 는 그대로 두세요. 시크릿 ARN 뒤의 임의 6자리를 포함하기 위한 와일드카드입니다.

**정책 이름**: `PetClinicSecretsAccess`

**정책 생성** 클릭.

---

### 13-3. 역할에 정책 2개가 붙었는지 확인

📍 작업 위치: **로컬 PC**

```bash
# 관리형 정책 확인
aws iam list-attached-role-policies \
  --role-name ecsTaskExecutionRole-petclinic \
  --query 'AttachedPolicies[*].PolicyName' --output text

# 인라인 정책 확인
aws iam list-role-policies \
  --role-name ecsTaskExecutionRole-petclinic \
  --query 'PolicyNames' --output text
```

**기대 결과:**
```
AmazonECSTaskExecutionRolePolicy
PetClinicSecretsAccess
```

두 줄이 모두 나와야 합니다. 하나라도 없으면 **13-1 또는 13-2로 돌아가세요.**

---

### 13-4. 신뢰 정책 확인 (중요)

```bash
aws iam get-role \
  --role-name ecsTaskExecutionRole-petclinic \
  --query 'Role.AssumeRolePolicyDocument.Statement[0].Principal.Service' \
  --output text
```

**기대 결과:**
```
ecs-tasks.amazonaws.com
```

> ⚠️ **`ecs.amazonaws.com` 이 나오면 잘못된 것입니다.** (13-1에서 사용 사례를 잘못 고른 경우)
> 역할을 삭제하고 **13-1부터 다시 하세요.**

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
| 모니터링 | Container Insights 비활성화 (비용 절감) |

**생성** 클릭. 약 1분 걸립니다.

---

### 14-2. CloudWatch 로그 그룹 미리 만들기

태스크 정의에서 로그 그룹을 자동 생성하게 둘 수도 있지만, 미리 만들어 두면 오류 원인을 좁히기 쉽습니다.

📍 작업 위치: **로컬 PC**

```bash
aws logs create-log-group \
  --log-group-name /ecs/petclinic \
  --region us-west-2
```

**기대 결과:** 아무 출력도 없으면 성공입니다.

이미 있으면 아래가 나옵니다. 무시하고 진행하세요.
```
An error occurred (ResourceAlreadyExistsException) ...
```

확인합니다.

```bash
aws logs describe-log-groups \
  --log-group-name-prefix /ecs/petclinic \
  --region us-west-2 \
  --query 'logGroups[0].logGroupName' --output text
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
| 운영 체제/아키텍처 | Linux/X86_64 |
| 네트워크 모드 | `awsvpc` (Fargate는 이것만 가능) |
| CPU | **1 vCPU** |
| 메모리 | **2GB** |
| **태스크 실행 역할** | **`ecsTaskExecutionRole-petclinic`** |
| 태스크 역할 | **없음** ← 개념 6 참조 |

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
> 이 형식을 쓰면 JSON 시크릿에서 **해당 키의 값만** 꺼내 환경변수에 넣습니다.
> `::` 를 빼면 JSON 전체가 문자열로 들어가 접속이 실패합니다.

**로깅:**

| 입력 항목 | 값 |
|----------|-----|
| 로그 수집 사용 | ☑ 체크 |
| awslogs-group | `/ecs/petclinic` |
| awslogs-region | `us-west-2` |
| awslogs-stream-prefix | `ecs` |

**상태 검사 (Health check) — 선택이지만 권장:**

| 입력 항목 | 값 |
|----------|-----|
| 명령 | `CMD-SHELL,curl -f http://localhost:8080/ \|\| exit 1` |
| 간격 | 30 |
| 제한 시간 | 5 |
| 시작 기간 | **90** |
| 재시도 | 3 |

> ⚠️ **시작 기간(startPeriod)을 90초로 두세요.** Spring Boot는 뜨는 데 30~60초 걸립니다.
> 이 값이 짧으면 정상 기동 중인 컨테이너를 ECS가 죽여 버립니다.

**생성** 클릭.

---

### 14-4. 태스크 정의가 만들어졌는지 확인

📍 작업 위치: **로컬 PC**

```bash
aws ecs describe-task-definition \
  --task-definition petclinic-task \
  --region us-west-2 \
  --query 'taskDefinition.[family,revision,cpu,memory,executionRoleArn]' \
  --output table
```

**기대 결과:**
```
--------------------------------------------------------------------------
|                         DescribeTaskDefinition                          |
+------------------+---+--------+--------+--------------------------------+
|  petclinic-task  | 1 |  1024  |  2048  | arn:aws:iam::...:role/ecsTask..|
+------------------+---+--------+--------+--------------------------------+
```

`revision` 이 `1` 이면 첫 버전입니다.

시크릿이 제대로 연결됐는지도 확인합니다.

```bash
aws ecs describe-task-definition \
  --task-definition petclinic-task \
  --region us-west-2 \
  --query 'taskDefinition.containerDefinitions[0].secrets[*].name' \
  --output text
```

**기대 결과:**
```
SPRING_DATASOURCE_URL   SPRING_DATASOURCE_USERNAME   SPRING_DATASOURCE_PASSWORD
```

3개가 모두 보여야 합니다. 없으면 **14-3의 시크릿 입력을 다시 하세요.**

---

## 15. Step 11 — ALB 생성

> ⏱ 예상 소요시간: 약 15분
> 📍 작업 위치: **AWS 콘솔** (EC2 → 로드 밸런서)

---

### 15-1. 대상 그룹 먼저 생성

ALB를 만들 때 대상 그룹을 선택해야 하므로, 대상 그룹을 먼저 만듭니다.

**EC2 → 대상 그룹 → 대상 그룹 생성**

| 입력 항목 | 값 |
|----------|-----|
| 대상 유형 | **IP 주소** ← 중요 |
| 대상 그룹 이름 | `petclinic-tg` |
| 프로토콜/포트 | HTTP / **8080** |
| VPC | `petclinic-vpc` |
| 프로토콜 버전 | HTTP1 |

> ⚠️ **대상 유형은 반드시 "IP 주소"입니다.**
> Fargate 태스크는 EC2 인스턴스가 아니라 ENI(IP)를 가집니다. "인스턴스"를 고르면 태스크를 등록할 수 없습니다.

**상태 검사:**

| 입력 항목 | 값 |
|----------|-----|
| 상태 검사 프로토콜 | HTTP |
| 상태 검사 경로 | `/` |
| 정상 임계값 | 2 |
| 비정상 임계값 | 3 |
| 제한 시간 | 5초 |
| 간격 | 30초 |
| 성공 코드 | 200 |

**다음** → 대상 등록 화면에서는 **아무것도 등록하지 않고** 그냥 **대상 그룹 생성** 클릭.

> 📌 태스크는 ECS 서비스가 자동으로 등록합니다. 여기서 수동 등록하지 마세요.

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

> ⚠️ **서브넷은 퍼블릭 2개**입니다. ALB는 최소 2개 AZ를 요구합니다.
> 프라이빗을 고르면 인터넷에서 접근할 수 없습니다.

**리스너 및 라우팅:**

| 프로토콜 | 포트 | 기본 작업 |
|---------|------|----------|
| HTTP | 80 | **`petclinic-tg`** 로 전달 |

**로드 밸런서 생성** 클릭. **프로비저닝에 약 3분** 걸립니다.

---

### 15-3. DNS 이름 확인

상태가 `활성(Active)` 이 되면 확인합니다.

**EC2 → 로드 밸런서 → `petclinic-alb` → 세부 정보**

**기대 결과:**
```
DNS 이름: petclinic-alb-1234567890.us-west-2.elb.amazonaws.com
```

> 📌 **이 주소를 메모하세요.** 접속 테스트와 부하 테스트에 씁니다.

CLI로도 확인 가능합니다.

```bash
aws elbv2 describe-load-balancers \
  --names petclinic-alb \
  --region us-west-2 \
  --query 'LoadBalancers[0].[State.Code,DNSName]' \
  --output text
```

**기대 결과:**
```
active   petclinic-alb-1234567890.us-west-2.elb.amazonaws.com
```

`provisioning` 이면 아직 준비 중입니다. 2~3분 더 기다리세요.

---

### 15-4. 지금 접속하면 어떻게 되나?

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://<ALB DNS 이름>
```

**기대 결과:**
```
503
```

**503이 정상입니다.** 아직 대상 그룹에 등록된 태스크가 하나도 없기 때문입니다. Step 12에서 서비스를 만들면 200으로 바뀝니다.

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

> 📌 이 두 값의 의미는 Step 15(롤링 업데이트)에서 자세히 다룹니다.
> 지금은 100/200으로 두세요. "새 태스크를 먼저 띄우고 구 태스크를 내린다"는 뜻입니다.

**네트워킹:**

| 입력 항목 | 값 |
|----------|-----|
| VPC | `petclinic-vpc` |
| **서브넷** | **프라이빗 서브넷 2개 모두** |
| 보안 그룹 | **기존 항목 선택 → `petclinic-app-sg`** |
| **퍼블릭 IP** | **끄기(Turned off)** ← 중요 |

> ⚠️ **퍼블릭 IP를 반드시 끄세요.**
> 프라이빗 서브넷에서 퍼블릭 IP를 켜면 라우팅이 없어 오히려 통신이 안 됩니다.
> VPC 엔드포인트로 나가므로 퍼블릭 IP가 필요 없습니다.

**로드 밸런싱:**

| 입력 항목 | 값 |
|----------|-----|
| 로드 밸런서 유형 | Application Load Balancer |
| 기존 로드 밸런서 사용 | **`petclinic-alb`** |
| 리스너 | 기존 리스너 사용 → `80:HTTP` |
| 대상 그룹 | 기존 대상 그룹 사용 → **`petclinic-tg`** |
| 상태 검사 유예 기간 | **120** |

> ⚠️ **상태 검사 유예 기간(health check grace period)을 120초로 두세요.**
> Spring Boot가 뜨기 전에 ALB가 헬스체크를 시작하면, 아직 준비 안 된 태스크를 비정상으로 판정해 죽입니다. 그러면 무한 재시작 루프에 빠집니다.

**서비스 오토스케일링:** 지금은 **사용 안 함**. Step 14에서 추가합니다.

**생성** 클릭.

---

### 16-2. 태스크가 뜨는지 지켜보기

**ECS → 클러스터 → `petclinic-cluster` → 서비스 → `petclinic-service` → 태스크 탭**

**약 2~3분간 상태가 이렇게 바뀝니다:**

```
PROVISIONING  →  PENDING  →  ACTIVATING  →  RUNNING
```

📍 작업 위치: **로컬 PC** — 터미널에서 실시간으로 보려면:

```bash
watch -n 5 'aws ecs describe-services \
  --cluster petclinic-cluster \
  --services petclinic-service \
  --region us-west-2 \
  --query "services[0].[desiredCount,runningCount,pendingCount]" \
  --output text'
```

**기대 결과 (최종):**
```
2    2    0
```

`desiredCount=2, runningCount=2, pendingCount=0` 이면 성공입니다. `Ctrl+C` 로 종료하세요.

---

### 16-3. 태스크가 안 뜰 때 — 오류 진단표

**ECS → 서비스 → `petclinic-service` → 이벤트 탭** 에서 메시지를 확인하세요.

또는 중지된 태스크의 상세 화면에서 `Stopped reason` 을 봅니다.

| 실제 오류 메시지 | 원인 | 돌아갈 곳 |
|----------------|------|----------|
| `CannotPullContainerError: ... i/o timeout` | VPC 엔드포인트 누락 (특히 **S3 Gateway**) | **Step 5 (9-2)** |
| `CannotPullContainerError: ... 403 Forbidden` | 태스크 실행 역할에 ECR 권한 없음 | **Step 9 (13-1)** |
| `ResourceInitializationError: unable to pull secrets` | 실행 역할에 `secretsmanager:GetSecretValue` 없음 | **Step 9 (13-2)** |
| `ResourceInitializationError: ... secretsmanager: RequestError` | Secrets Manager 엔드포인트 누락 | **Step 5 (9-1)** |
| `Task failed ELB health checks` | 유예 기간이 짧음 / 8080 포트 불일치 | **Step 12 (16-1)** 유예 120초 확인 |
| `OutOfMemoryError` (로그) | 메모리 1GB로 설정 | **Step 10 (14-3)** 2GB로 |
| `unable to place a task` | 서브넷·SG 설정 오류 | **Step 12 (16-1)** 네트워킹 재확인 |

> 📌 **오류 메시지 전문을 반드시 읽으세요.** 위 표의 어느 것과 맞는지 대조하면 대부분 5분 안에 해결됩니다.

---

### 16-4. 컨테이너 로그 확인

태스크는 떴는데 앱이 이상하면 로그를 봅니다.

📍 작업 위치: **로컬 PC**

```bash
aws logs tail /ecs/petclinic --follow --region us-west-2
```

**기대 결과 (정상):**
```
2026-07-10T05:12:33 ecs/petclinic/abc123  Started PetClinicApplication in 24.5 seconds
2026-07-10T05:12:33 ecs/petclinic/abc123  Tomcat started on port 8080
```

**비정상 로그와 해결:**

| 로그에 보이는 메시지 | 원인 | 해결 |
|--------------------|------|------|
| `Communications link failure` | RDS 보안 그룹이 태스크를 막음 | **Step 4 (8-3)** — `petclinic-db-sg` 인바운드에 `petclinic-app-sg` 참조 확인 |
| `Access denied for user 'petclinic'` | 시크릿의 비밀번호 불일치 | **Step 8 (12-1)** — RDS 마스터 암호와 대조 |
| `Unknown database 'petclinic'` | RDS 초기 DB 이름 미지정 | **Step 7 (11-2)** — 초기 데이터베이스 이름 확인 |
| `No suitable driver found for jdbc:mariadb` | URL 스킴 오류 | **Step 8 (12-1)** — `jdbc:mysql://` 로 수정 |
| `Table 'petclinic.owners' doesn't exist` | 스키마 미생성 | `SPRING_PROFILES_ACTIVE=mysql` 확인 (**14-3**) |
| 로그 자체가 안 나옴 | CloudWatch Logs 엔드포인트 누락 | **Step 5 (9-1)** |

---

### 16-5. 접속 확인 (첫 성공 지점)

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://<ALB DNS 이름>
```

**기대 결과:**
```
200
```

브라우저에서도 열어 보세요.

```
http://petclinic-alb-1234567890.us-west-2.elb.amazonaws.com
```

**기대 결과:** PetClinic 홈 화면(강아지 그림과 "Welcome" 문구)이 보입니다.

> 🎉 **여기까지 왔으면 절반은 끝난 것입니다.**
> 컨테이너 이미지 → ECR → Fargate → ALB → RDS(시크릿 주입)까지 전 경로가 동작한다는 뜻입니다.

**여전히 503이면:**

| 확인 순서 | 명령 |
|----------|------|
| ① 태스크가 RUNNING인가 | `aws ecs describe-services --cluster petclinic-cluster --services petclinic-service --region us-west-2 --query 'services[0].runningCount'` |
| ② 대상 그룹에 등록됐는가 | 아래 명령 |

```bash
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups \
    --names petclinic-tg --region us-west-2 \
    --query 'TargetGroups[0].TargetGroupArn' --output text) \
  --region us-west-2 \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
  --output table
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

> 📌 **목표값을 50으로 두는 이유:** 실습에서 스케일 아웃을 빨리 보기 위해서입니다.
> 실무에서는 보통 70을 씁니다. 50이면 조금만 부하가 와도 태스크가 늘어납니다.

> 📌 **규모 확장 휴지 60초 / 규모 축소 300초** 인 이유:
> 늘릴 때는 빠르게(장애 방지), 줄일 때는 천천히(플래핑 방지) 하는 것이 원칙입니다.

**업데이트** 클릭.

---

### 17-2. 오토스케일링이 등록됐는지 확인

📍 작업 위치: **로컬 PC**

```bash
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --region us-west-2 \
  --query 'ScalableTargets[*].[ResourceId,MinCapacity,MaxCapacity]' \
  --output table
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

```bash
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --region us-west-2 \
  --query 'ScalingPolicies[*].[PolicyName,TargetTrackingScalingPolicyConfiguration.TargetValue]' \
  --output table
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
> 📍 작업 위치: **로컬 PC** (또는 별도 EC2)

---

### 18-1. 부하 생성 도구 설치

로컬 PC에서 실행합니다. `hey` 는 가볍고 설치가 쉬운 HTTP 부하 도구입니다.

```bash
# Ubuntu / Debian
sudo apt install -y hey

# 위 명령이 안 되면 바이너리 직접 다운로드
curl -L -o hey https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
chmod +x hey
sudo mv hey /usr/local/bin/
```

확인합니다.

```bash
hey -h 2>&1 | head -3
```

**기대 결과:**
```
Usage: hey [options...] <url>

Options:
```

> 📌 `hey` 가 안 되면 `ab`(Apache Bench)를 써도 됩니다.
> `sudo apt install -y apache2-utils` 후 `ab -n 100000 -c 100 http://<ALB>/`

---

### 18-2. 모니터링 창 먼저 열기 (중요)

**부하를 주기 전에** 관찰 창을 열어야 변화를 볼 수 있습니다.

**터미널 1** — 태스크 개수 감시:

```bash
watch -n 10 'aws ecs describe-services \
  --cluster petclinic-cluster \
  --services petclinic-service \
  --region us-west-2 \
  --query "services[0].[desiredCount,runningCount]" \
  --output text'
```

**터미널 2** — CPU 사용률 감시 (콘솔이 더 편합니다):

**ECS → 클러스터 → `petclinic-cluster` → 서비스 → `petclinic-service` → 지표 탭**

CPU 사용률 그래프를 띄워 두세요.

---

### 18-3. 부하 인가

**터미널 3** 에서 실행합니다.

```bash
export ALB_DNS=<여기에 ALB DNS 이름>

hey -z 10m -c 100 http://${ALB_DNS}/
```

옵션 의미:

| 옵션 | 의미 |
|------|------|
| `-z 10m` | 10분간 계속 요청 |
| `-c 100` | 동시 연결 100개 |

> ⚠️ **`-c` 를 1000 이상으로 올리지 마세요.** 로컬 PC의 파일 디스크립터 한계에 걸려 도구 자체가 죽습니다.

---

### 18-4. 관찰 — 무엇을 봐야 하는가

**시간 순서대로 이렇게 진행됩니다.**

```
0분    부하 시작.  태스크 2개.  CPU 20% → 상승
       │
1~3분  CPU 50% 초과.  CloudWatch 경보가 ALARM 상태로 전환
       │
3~5분  desiredCount 2 → 3 또는 4 로 증가
       │  ┌─ 터미널 1에서 "2  2" → "4  2" 로 바뀜
       │  └─ 새 태스크가 PROVISIONING → RUNNING
       │
5~7분  runningCount 도 따라 증가.  "4  4"
       │  ALB가 새 태스크를 대상 그룹에 자동 등록
       │
7~10분 태스크가 늘어 CPU가 분산.  50% 아래로 하강
       │
10분   부하 종료 (hey 자동 종료)
       │
15~20분 규모 축소 휴지 300초 경과 후 desiredCount 감소
       │  "4  4" → "2  2"
```

> 📌 **스케일 아웃까지 3~5분이 걸립니다.** 즉시 반응하지 않습니다.
> CloudWatch 지표는 1분 단위로 수집되고, 경보는 여러 데이터 포인트를 보고 판단하기 때문입니다.
> 학생들이 "왜 안 늘어나요?"라고 물으면 이 지연을 설명하세요.

---

### 18-5. 스케일링 이력 확인

부하 테스트가 끝난 뒤 무슨 일이 있었는지 봅니다.

```bash
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/petclinic-cluster/petclinic-service \
  --region us-west-2 \
  --query 'ScalingActivities[*].[StartTime,Description,StatusCode]' \
  --output table
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

---

### 18-6. ECS Exec으로 태스크 내부 확인 (선택)

부하 중에 태스크 안에서 무슨 일이 벌어지는지 보고 싶다면 셸로 들어갈 수 있습니다. SSH가 필요 없습니다.

먼저 서비스에 Exec을 활성화합니다.

```bash
aws ecs update-service \
  --cluster petclinic-cluster \
  --service petclinic-service \
  --enable-execute-command \
  --force-new-deployment \
  --region us-west-2
```

> ⚠️ 이 명령은 태스크를 전부 재시작합니다. 부하 테스트 전에 미리 해 두세요.
> 또한 **태스크 역할**에 SSM 권한이 필요합니다. (개념 6 — 이번엔 태스크 역할입니다)

태스크 ID를 확인하고 접속합니다.

```bash
TASK_ID=$(aws ecs list-tasks --cluster petclinic-cluster \
  --service-name petclinic-service --region us-west-2 \
  --query 'taskArns[0]' --output text | awk -F/ '{print $NF}')

aws ecs execute-command \
  --cluster petclinic-cluster \
  --task $TASK_ID \
  --container petclinic \
  --interactive \
  --command "/bin/sh" \
  --region us-west-2
```

> 📌 **이 단계는 실패해도 괜찮습니다.** 본 실습의 필수 경로가 아닙니다.
> `TargetNotConnectedException` 이 나오면 SSM 권한이 없는 것입니다. 건너뛰세요.

---

## 19. Step 15 — 롤링 업데이트 (무중단 배포)

> ⏱ 예상 소요시간: 약 25분
> 📍 작업 위치: **로컬 PC** → **AWS 콘솔**

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
┌─ 최소 100%... 가 아니라 50%  → 1개까지 줄어들어도 됨
└─ 최대 100%                  → 2개를 넘길 수 없음

결과: 구 태스크 1개를 먼저 내리고 → 새 태스크 1개 띄움 → 반복
      순간 용량이 절반으로 떨어짐.  트래픽이 많으면 지연 발생.
```

| 설정 | 배포 중 최대 태스크 | 배포 중 최소 태스크 | 용량 손실 | 추가 비용 |
|------|-------------------|-------------------|----------|----------|
| **100 / 200** | 4 | 2 | **없음** | 배포 중 2배 |
| 50 / 100 | 2 | 1 | **50%** | 없음 |

**이 실습은 100/200 을 씁니다.** 무중단을 보여주는 것이 목적이기 때문입니다.

---

### 19-2. v2 이미지 만들기

📍 작업 위치: **로컬 PC** (`~/spring-petclinic`)

변경을 눈으로 확인할 수 있도록 화면 문구를 바꿉니다.

```bash
cd ~/spring-petclinic
grep -rn "Welcome" src/main/resources/messages/messages.properties
```

**기대 결과:**
```
1:welcome=Welcome
```

문구를 수정합니다.

```bash
sed -i 's/^welcome=Welcome$/welcome=Welcome - Version 2/' src/main/resources/messages/messages.properties
grep "^welcome=" src/main/resources/messages/messages.properties
```

**기대 결과:**
```
welcome=Welcome - Version 2
```

---

### 19-3. 빌드 및 푸시

```bash
./mvnw clean package -DskipTests

docker build -t petclinic:v2 .
docker tag petclinic:v2 ${ECR_URI}:v2
docker push ${ECR_URI}:v2
```

> ⚠️ **`$ECR_URI` 변수가 비어 있으면** 새 터미널을 연 것입니다. **10-2를 다시 실행**하세요.

**기대 결과 (마지막 줄):**
```
v2: digest: sha256:yyyy... size: 1573
```

ECR에 두 태그가 모두 있는지 확인합니다.

```bash
aws ecr describe-images --repository-name petclinic --region us-west-2 \
  --query 'imageDetails[*].imageTags[0]' --output text
```

**기대 결과:**
```
v2   v1
```

---

### 19-4. 태스크 정의 새 리비전 생성

**개념 3을 기억하세요.** 태스크 정의는 수정할 수 없습니다. 새 리비전을 만듭니다.

**ECS → 태스크 정의 → `petclinic-task` → 최신 리비전 선택 → 새 리비전 생성**

**컨테이너 정의의 이미지 URI만** 바꿉니다.

```
변경 전: <계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic:v1
변경 후: <계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic:v2
                                                          ↑ 여기만
```

**나머지는 전부 그대로 두세요.** 시크릿, 로깅, 헬스체크 설정을 건드리지 마세요.

**생성** 클릭.

**기대 결과:** `petclinic-task:2` 가 만들어집니다.

```bash
aws ecs describe-task-definition \
  --task-definition petclinic-task \
  --region us-west-2 \
  --query 'taskDefinition.[revision,containerDefinitions[0].image]' \
  --output text
```

**기대 결과:**
```
2    123456789012.dkr.ecr.us-west-2.amazonaws.com/petclinic:v2
```

---

### 19-5. 무중단 확인용 감시 시작 (배포 전에 먼저)

**터미널 1** — 1초마다 접속해 응답 코드를 기록합니다.

```bash
while true; do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://${ALB_DNS}/)
  echo "$(date +%H:%M:%S)  $code"
  sleep 1
done
```

**기대 결과 (배포 전):**
```
06:40:01  200
06:40:02  200
06:40:03  200
```

**이 창을 계속 열어 둔 채로** 다음 단계를 진행합니다.

---

### 19-6. 서비스 업데이트 (배포 실행)

**터미널 2** 또는 콘솔에서 수행합니다.

**콘솔:** ECS → 서비스 → `petclinic-service` → **업데이트** → 개정을 `2(최신)` 로 변경 → **업데이트**

**CLI:**

```bash
aws ecs update-service \
  --cluster petclinic-cluster \
  --service petclinic-service \
  --task-definition petclinic-task:2 \
  --region us-west-2 \
  --query 'service.[serviceName,taskDefinition]' \
  --output text
```

**기대 결과:**
```
petclinic-service   arn:aws:ecs:us-west-2:...:task-definition/petclinic-task:2
```

---

### 19-7. 배포 진행 관찰

**터미널 3:**

```bash
watch -n 5 'aws ecs describe-services \
  --cluster petclinic-cluster \
  --services petclinic-service \
  --region us-west-2 \
  --query "services[0].deployments[*].[status,taskDefinition,desiredCount,runningCount]" \
  --output table'
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

`ACTIVE` 줄이 사라지고 `PRIMARY` 하나만 남으면 완료입니다.

---

### 19-8. 무중단이었는지 검증

**터미널 1** 을 다시 봅니다.

**기대 결과:**
```
06:42:01  200
06:42:02  200
...
06:45:33  200      ← 배포 중에도 계속 200
06:45:34  200
```

**단 한 번도 `502`, `503`, `504` 가 나오지 않아야 합니다.**

> 🎉 **한 줄도 실패하지 않았다면 무중단 배포에 성공한 것입니다.**

`Ctrl+C` 로 종료하세요.

**만약 503이 섞여 나왔다면:**

| 원인 | 해결 |
|------|------|
| 최소 비율이 100 미만 | **Step 12 (16-1)** — 100으로 수정 |
| 헬스체크 유예 기간 부족 | **Step 12 (16-1)** — 120초로 수정 |
| 대상 그룹 등록 해제 지연 시간 너무 짧음 | 대상 그룹 속성 → `deregistration_delay` 30초 이상 |

---

### 19-9. 브라우저에서 v2 확인

```
http://<ALB DNS 이름>
```

**기대 결과:** 화면에 **"Welcome - Version 2"** 가 보입니다.

여전히 "Welcome"만 보이면 브라우저 캐시입니다. `Ctrl+Shift+R` 로 강제 새로고침하세요.

---

### 19-10. 롤백 실습

배포가 잘못됐을 때 되돌리는 방법입니다. **이전 리비전을 다시 지정**하면 끝입니다.

```bash
aws ecs update-service \
  --cluster petclinic-cluster \
  --service petclinic-service \
  --task-definition petclinic-task:1 \
  --region us-west-2
```

3~5분 뒤 브라우저를 새로고침하면 "Welcome"으로 돌아옵니다.

> 📌 **이것이 태스크 정의를 수정 불가로 만든 이유입니다.**
> 모든 리비전이 영구 보존되므로, 언제든 특정 시점으로 되돌릴 수 있습니다.

다시 v2로 올려 두고 다음 단계로 갑니다.

```bash
aws ecs update-service \
  --cluster petclinic-cluster \
  --service petclinic-service \
  --task-definition petclinic-task:2 \
  --region us-west-2
```

---

## 20. Step 16 — CodePipeline으로 CI/CD 자동화

> ⏱ 예상 소요시간: 약 40분
> 📍 작업 위치: **로컬 PC** → **AWS 콘솔**

**지금까지는 이미지를 손으로 빌드해서 손으로 배포했습니다.** 이제 Git에 푸시하면 자동으로 되게 만듭니다.

---

### 20-1. 자동화될 흐름

```
개발자가 GitHub에 push
         │
         ▼
   ① CodePipeline이 감지 (Source 단계)
         │
         ▼
   ② CodeBuild가 실행 (Build 단계)
      ├─ mvn package
      ├─ docker build
      └─ ECR에 push
         │
         ▼
   ③ ECS가 새 이미지로 롤링 업데이트 (Deploy 단계)
```

---

### 20-2. GitHub 저장소 준비

📍 작업 위치: **로컬 PC**

본인 GitHub 계정에 저장소를 만들고 코드를 올립니다.

```bash
cd ~/spring-petclinic
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

---

### 20-3. buildspec.yml 작성

CodeBuild에게 무엇을 할지 알려주는 파일입니다. 프로젝트 루트에 `buildspec.yml` 로 저장하세요.

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

**각 단계가 하는 일:**

| 단계 | 하는 일 |
|------|--------|
| `pre_build` | ECR 로그인, 커밋 해시로 이미지 태그 생성 |
| `build` | Maven 빌드 → Docker 이미지 빌드 |
| `post_build` | ECR 푸시, `imagedefinitions.json` 생성 |

> 📌 **`imagedefinitions.json` 이 핵심입니다.**
> Deploy 단계가 이 파일을 읽어 "어느 컨테이너를 어느 이미지로 바꿀지" 판단합니다.
> `"name"` 값 `petclinic` 은 **태스크 정의의 컨테이너 이름과 정확히 일치**해야 합니다. (14-3에서 설정한 값)

> 📌 **커밋 해시를 태그로 쓰는 이유:** `latest` 만 쓰면 어느 커밋이 배포됐는지 추적할 수 없습니다.
> 실무에서는 반드시 불변 태그(커밋 해시, 빌드 번호)를 씁니다.

커밋하고 푸시합니다.

```bash
git add buildspec.yml
git commit -m "Add buildspec for CodeBuild"
git push
```

---

### 20-4. CodeBuild 서비스 역할에 필요한 권한

CodeBuild가 ECR에 푸시하려면 권한이 필요합니다. 파이프라인 생성 시 역할을 자동 생성하되, ECR 권한을 추가해야 합니다.

**미리 알아 두세요.** 20-6 이후에 아래 오류가 나면 이 권한 문제입니다.

```
denied: User: arn:aws:sts::...:assumed-role/codebuild-... is not authorized to perform: ecr:InitiateLayerUpload
```

---

### 20-5. CodeBuild 프로젝트 생성

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
> 컨테이너 안에서 Docker 데몬을 돌리려면 특권 모드가 필요합니다.
> 오류 메시지: `Cannot connect to the Docker daemon`

**환경 변수 추가:**

| 이름 | 값 | 유형 |
|------|-----|------|
| `ECR_URI` | `<계정ID>.dkr.ecr.us-west-2.amazonaws.com/petclinic` | 일반 텍스트 |

**프로젝트 생성** 클릭.

---

### 20-6. CodeBuild 역할에 ECR 권한 추가

**IAM → 역할 → `codebuild-petclinic-build-service-role` → 권한 추가 → 정책 연결**

`AmazonEC2ContainerRegistryPowerUser` 를 검색해 연결합니다.

확인합니다.

```bash
aws iam list-attached-role-policies \
  --role-name codebuild-petclinic-build-service-role \
  --query 'AttachedPolicies[*].PolicyName' --output text
```

**기대 결과:** 목록에 `AmazonEC2ContainerRegistryPowerUser` 가 포함됩니다.

---

### 20-7. 빌드 단독 테스트 (파이프라인 만들기 전)

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
| `Cannot connect to the Docker daemon` | 특권 모드 미체크 | **20-5** — 프로젝트 편집 → 권한 있음 체크 |
| `denied: ... not authorized to perform: ecr:*` | ECR 권한 없음 | **20-6** |
| `Unsupported class file major version` | 빌드 이미지의 Java가 17 미만 | 환경 이미지를 `standard:5.0` 으로 |
| `buildspec.yml not found` | 파일이 저장소 루트에 없음 | **20-3** — 커밋·푸시 확인 |

---

### 20-8. CodePipeline 생성

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

> 📌 이 파일 이름이 buildspec의 `printf ... > imagedefinitions.json` 과 일치해야 합니다.

**파이프라인 생성** 클릭. 생성 즉시 첫 실행이 시작됩니다.

---

### 20-9. 전체 파이프라인 동작 확인

**CodePipeline → `petclinic-pipeline`**

**기대 결과 (약 10분 후):**
```
Source   ✓ 성공
Build    ✓ 성공
Deploy   ✓ 성공
```

세 단계 모두 초록색이면 자동화가 완성된 것입니다.

---

### 20-10. 실제 자동 배포 시연

📍 작업 위치: **로컬 PC**

코드를 바꾸고 푸시하기만 하면 됩니다.

```bash
cd ~/spring-petclinic
sed -i 's/^welcome=.*/welcome=Welcome - Version 3 (Auto Deployed)/' \
  src/main/resources/messages/messages.properties

git add .
git commit -m "Update welcome message to v3"
git push
```

**터미널에서 무중단을 감시하면서:**

```bash
while true; do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://${ALB_DNS}/)
  echo "$(date +%H:%M:%S)  $code"
  sleep 2
done
```

**기대 결과:**

```
1분 후   CodePipeline의 Source 단계가 In Progress
3분 후   Build 단계 진행 (mvn + docker)
9분 후   Deploy 단계 시작 → ECS 롤링 업데이트
14분 후  브라우저에 "Welcome - Version 3 (Auto Deployed)" 표시
```

**그동안 curl 응답은 계속 200이어야 합니다.**

> 🎉 **여기까지 성공하면 실습 전 과정이 완료된 것입니다.**
> `git push` 한 번으로 빌드·이미지 생성·무중단 배포가 자동 수행되었습니다.

**Deploy 단계가 실패하면:**

| 오류 메시지 | 원인 | 해결 |
|-----------|------|------|
| `The image definitions file cannot be found` | `imagedefinitions.json` 미생성 | **20-3** buildspec의 artifacts 확인 |
| `Invalid action configuration: container name` | JSON의 `name` 과 태스크 정의 컨테이너명 불일치 | **20-3** — `petclinic` 으로 통일 |
| `Deployment failed: tasks failed to start` | 새 이미지가 기동 실패 | **16-4** 로그 확인 |
---

## 21. 정상 동작 최종 확인

모든 단계가 끝났으면 아래를 순서대로 확인하세요.

📍 작업 위치: **로컬 PC**

---

### 21-1. 서비스 상태

```bash
aws ecs describe-services \
  --cluster petclinic-cluster \
  --services petclinic-service \
  --region us-west-2 \
  --query 'services[0].[status,desiredCount,runningCount,pendingCount]' \
  --output text
```

**기대 결과:**
```
ACTIVE   2   2   0
```

---

### 21-2. 대상 그룹 상태

```bash
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups \
    --names petclinic-tg --region us-west-2 \
    --query 'TargetGroups[0].TargetGroupArn' --output text) \
  --region us-west-2 \
  --query 'TargetHealthDescriptions[*].TargetHealth.State' \
  --output text
```

**기대 결과:**
```
healthy   healthy
```

---

### 21-3. 애플리케이션 응답

```bash
curl -s http://${ALB_DNS}/ | grep -o "Welcome[^<]*"
```

**기대 결과:**
```
Welcome - Version 3 (Auto Deployed)
```

---

### 21-4. DB 연결 확인 (앱 기능 테스트)

브라우저에서 아래를 수행합니다.

1. `http://<ALB DNS>/owners/find` 접속
2. 검색창을 비운 채 **Find Owner** 클릭
3. 소유자 목록이 표시되면 **RDS 연결 성공**

**기대 결과:** George Franklin, Betty Davis 등 10명의 목록이 나옵니다.

> ⚠️ **목록이 비어 있거나 500 오류가 나면** DB 연결에 문제가 있습니다.
> **16-4** 로그 확인표로 돌아가세요.

---

### 21-5. 비정상 로그와 해결 (종합)

```bash
aws logs tail /ecs/petclinic --since 30m --region us-west-2 | grep -iE "error|exception|fail"
```

| 로그에 보이는 메시지 | 원인 | 해결 |
|--------------------|------|------|
| `Communications link failure` | RDS SG가 태스크 차단 | **Step 4 (8-3)** |
| `Access denied for user` | 시크릿 비밀번호 불일치 | **Step 8 (12-1)** |
| `Unknown database 'petclinic'` | RDS 초기 DB 미생성 | **Step 7 (11-2)** |
| `No suitable driver` | JDBC URL 스킴 오류 | **Step 8 (12-1)** — `jdbc:mysql://` |
| `Table ... doesn't exist` | 프로파일 미적용 | **Step 10 (14-3)** — `SPRING_PROFILES_ACTIVE=mysql` |
| `OutOfMemoryError` | 메모리 부족 | **Step 10 (14-3)** — 2GB |
| `Connection refused` (헬스체크) | 포트 불일치 | **Step 11 (15-1)** — 대상 그룹 8080 |
| 로그가 아예 없음 | Logs 엔드포인트 누락 | **Step 5 (9-1)** |

---

## 22. 변경·롤백 절차

운영 중 값을 바꿔야 할 때 **반드시 아래 순서를 지키세요.** 순서를 어기면 서비스가 중단됩니다.

---

### 22-1. DB 비밀번호를 바꿀 때

```
① RDS에서 마스터 암호 변경          (아직 태스크는 구 암호를 갖고 있음)
       ↓
② Secrets Manager의 password 값 수정
       ↓
③ 서비스 강제 재배포                 (새 태스크가 새 암호를 읽어 감)
   aws ecs update-service --force-new-deployment
       ↓
④ 롤링 업데이트로 태스크 교체         (무중단)
```

> ⚠️ **①과 ② 사이에 재배포하면 안 됩니다.** 새 태스크가 구 암호로 접속을 시도해 실패합니다.
> **②를 먼저 끝내고 ③을 하세요.**

```bash
aws ecs update-service \
  --cluster petclinic-cluster \
  --service petclinic-service \
  --force-new-deployment \
  --region us-west-2
```

> 📌 **시크릿 값만 바꾸면 자동으로 반영되지 않습니다.**
> 시크릿은 태스크가 **시작될 때 한 번만** 읽힙니다. 반드시 재배포해야 합니다.

---

### 22-2. 이미지를 바꿀 때

```
① 새 태그로 빌드·푸시  (v3)
       ↓
② 태스크 정의 새 리비전 생성  (이미지 URI만 변경)
       ↓
③ 서비스 업데이트  (--task-definition petclinic-task:3)
```

CodePipeline을 쓰면 ①②③이 자동입니다. (Step 16)

---

### 22-3. 롤백 절차

| 상황 | 명령 |
|------|------|
| 직전 리비전으로 | `aws ecs update-service --task-definition petclinic-task:<N-1> ...` |
| 특정 시점으로 | 리비전 목록에서 원하는 번호 선택 |
| 파이프라인 배포를 되돌림 | 이전 커밋으로 `git revert` 후 푸시 |

리비전 목록 확인:

```bash
aws ecs list-task-definitions \
  --family-prefix petclinic-task \
  --region us-west-2 \
  --query 'taskDefinitionArns' --output text | tr '\t' '\n'
```

---

## 23. 보안 주의사항 (반드시 지킬 것)

| 번호 | 규칙 | 이유 |
|------|------|------|
| 1 | **DB 비밀번호를 태스크 정의의 `environment` 에 넣지 않는다** | 콘솔·CLI로 누구나 평문 조회 가능. 반드시 `secrets` (ValueFrom) 사용 |
| 2 | **Dockerfile에 자격증명을 넣지 않는다** | 이미지 레이어에 영구 기록됨. `docker history` 로 노출 |
| 3 | **buildspec.yml에 시크릿을 직접 쓰지 않는다** | Git에 커밋되어 영구 노출. CodeBuild 환경변수(Secrets Manager 유형) 사용 |
| 4 | **컨테이너를 root로 실행하지 않는다** | 컨테이너 탈출 시 피해 확대. `USER app` 필수 (Step 2) |
| 5 | **보안 그룹 소스에 IP 대신 SG를 참조한다** | Fargate는 IP가 매번 바뀜. SG 체이닝만 안정적 |
| 6 | **RDS 퍼블릭 액세스를 "아니요"로 둔다** | 인터넷에서 DB에 직접 접근 가능해짐 |
| 7 | **ECR 이미지 스캔을 켠다** | 알려진 CVE를 푸시 시점에 탐지 |
| 8 | **`latest` 태그만으로 배포하지 않는다** | 어느 커밋이 배포됐는지 추적 불가. 불변 태그(커밋 해시) 병용 |
| 9 | **태스크 실행 역할의 시크릿 권한을 특정 ARN으로 제한한다** | `Resource: "*"` 로 두면 계정의 모든 시크릿을 읽을 수 있음 |
| 10 | **VPC 엔드포인트 SG를 443만 열고 소스를 app-sg로 제한한다** | 0.0.0.0/0 으로 열면 VPC 내 모든 리소스가 API 호출 가능 |

---

## 24. 리소스 정리 (실습 종료 시 필수)

> ⚠️ **정리하지 않으면 계속 과금됩니다.**
> ALB, RDS, VPC 엔드포인트, NAT(없음)가 시간당 요금을 발생시킵니다.

**삭제는 생성의 역순입니다.** 순서를 어기면 삭제가 거부됩니다.

```
① CodePipeline 삭제
② CodeBuild 프로젝트 삭제
③ ECS 서비스 삭제  (desiredCount를 0으로 먼저 내림)
④ ECS 클러스터 삭제
⑤ ALB 삭제 → 대상 그룹 삭제
⑥ RDS 삭제 (최종 스냅샷 없음 선택)
⑦ VPC 엔드포인트 5개 삭제   ← 잊기 쉬움
⑧ Secrets Manager 시크릿 삭제 (강제 삭제)
⑨ ECR 리포지토리 삭제 (이미지 포함)
⑩ CloudWatch 로그 그룹 삭제
⑪ 보안 그룹 4개 삭제
⑫ VPC 삭제
⑬ IAM 역할 삭제
```

---

### 24-1. 일괄 정리 스크립트

📍 작업 위치: **로컬 PC**

```bash
export REGION=us-west-2

# ③ 서비스 축소 후 삭제
aws ecs update-service --cluster petclinic-cluster \
  --service petclinic-service --desired-count 0 --region $REGION
sleep 30
aws ecs delete-service --cluster petclinic-cluster \
  --service petclinic-service --force --region $REGION

# ④ 클러스터 삭제
aws ecs delete-cluster --cluster petclinic-cluster --region $REGION

# ⑤ ALB · 대상 그룹
ALB_ARN=$(aws elbv2 describe-load-balancers --names petclinic-alb \
  --region $REGION --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN --region $REGION
sleep 60
TG_ARN=$(aws elbv2 describe-target-groups --names petclinic-tg \
  --region $REGION --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 delete-target-group --target-group-arn $TG_ARN --region $REGION

# ⑥ RDS
aws rds delete-db-instance --db-instance-identifier petclinic-db \
  --skip-final-snapshot --delete-automated-backups --region $REGION

# ⑧ 시크릿 (강제 즉시 삭제)
aws secretsmanager delete-secret --secret-id petclinic/db \
  --force-delete-without-recovery --region $REGION

# ⑨ ECR
aws ecr delete-repository --repository-name petclinic --force --region $REGION

# ⑩ 로그 그룹
aws logs delete-log-group --log-group-name /ecs/petclinic --region $REGION
```

> 📌 **⑦ VPC 엔드포인트와 ⑫ VPC는 콘솔에서 지우는 편이 안전합니다.**
> RDS 삭제(약 5분)가 끝난 뒤에 진행하세요. 먼저 지우면 종속성 오류가 납니다.

---

### 24-2. 삭제 확인

```bash
# 남아 있는 리소스가 없는지 확인
aws ecs list-clusters --region $REGION
aws elbv2 describe-load-balancers --region $REGION --query 'LoadBalancers[*].LoadBalancerName'
aws rds describe-db-instances --region $REGION --query 'DBInstances[*].DBInstanceIdentifier'
aws ec2 describe-vpc-endpoints --region $REGION --query 'VpcEndpoints[*].ServiceName'
```

**기대 결과:** 모두 빈 배열 `[]` 또는 빈 목록이면 정리 완료입니다.

**다음 날 Cost Explorer에서 잔여 과금이 없는지 교차 확인하세요.**

---

## 25. 완료 체크리스트 (인수인계용)

### 로컬 PC에서 한 일

- [ ] `spring-petclinic` 저장소 복제
- [ ] `Dockerfile` 작성 (root 비실행, `USER app`)
- [ ] `.dockerignore` 작성
- [ ] 로컬에서 `docker run` 으로 200 응답 확인
- [ ] `buildspec.yml` 작성 및 커밋
- [ ] `hey` 로 부하 테스트 수행

### AWS 콘솔에서 한 일

- [ ] VPC `petclinic-vpc` 생성 (NAT 없음, DNS 활성화)
- [ ] 퍼블릭·프라이빗 서브넷 각 2개
- [ ] 보안 그룹 4개 (`-sg` 접미어, SG 체이닝)
- [ ] VPC 엔드포인트 5개 (Interface 4 + Gateway 1)
- [ ] ECR 리포지토리 `petclinic`
- [ ] RDS `petclinic-db` (프라이빗, 퍼블릭 액세스 아니요)
- [ ] Secrets Manager `petclinic/db` (url/username/password)
- [ ] IAM `ecsTaskExecutionRole-petclinic` + 인라인 정책
- [ ] ECS 클러스터 `petclinic-cluster`
- [ ] 태스크 정의 `petclinic-task` (1 vCPU / 2GB)
- [ ] ALB `petclinic-alb` + 대상 그룹 `petclinic-tg` (IP 유형)
- [ ] ECS 서비스 `petclinic-service` (100/200, 유예 120초)
- [ ] 오토스케일링 (min 2, max 6, CPU 50%)
- [ ] CodeBuild `petclinic-build` (특권 모드)
- [ ] CodePipeline `petclinic-pipeline`

### 검증 완료 항목

- [ ] ALB DNS로 200 응답
- [ ] `/owners/find` 에서 소유자 목록 조회 (RDS 연결 확인)
- [ ] 부하 테스트로 태스크 2 → 4 증가 확인
- [ ] 부하 종료 후 4 → 2 감소 확인
- [ ] 롤링 업데이트 중 503 발생 0건
- [ ] `git push` 로 자동 배포 성공

---

## 26. 자주 묻는 질문 (FAQ)

---

### Q1. 태스크가 계속 PENDING에서 멈춥니다.

**A**: 대부분 이미지를 못 받는 경우입니다. 순서대로 확인하세요.

```bash
# 중지된 태스크의 실패 이유 확인
aws ecs describe-tasks \
  --cluster petclinic-cluster \
  --tasks $(aws ecs list-tasks --cluster petclinic-cluster \
    --desired-status STOPPED --region us-west-2 \
    --query 'taskArns[0]' --output text) \
  --region us-west-2 \
  --query 'tasks[0].stoppedReason' --output text
```

`CannotPullContainerError` 가 나오면 **S3 Gateway 엔드포인트가 없는 것**입니다. **Step 5 (9-2)** 로 가세요. 이것이 가장 흔한 원인입니다.

---

### Q2. 시크릿을 만들었는데 태스크가 `ResourceInitializationError` 로 죽습니다.

**A**: 두 가지 중 하나입니다.

**① 태스크 실행 역할에 권한이 없다** — 확인 명령:

```bash
aws iam list-role-policies \
  --role-name ecsTaskExecutionRole-petclinic \
  --query 'PolicyNames' --output text
```

`PetClinicSecretsAccess` 가 없으면 **Step 9 (13-2)** 로.

**② Secrets Manager VPC 엔드포인트가 없다** — 확인 명령:

```bash
aws ec2 describe-vpc-endpoints --region us-west-2 \
  --query 'VpcEndpoints[?contains(ServiceName,`secretsmanager`)].State' --output text
```

`available` 이 안 나오면 **Step 5 (9-1)** 로.

---

### Q3. 시크릿 값을 바꿨는데 앱이 여전히 옛날 비밀번호를 씁니다.

**A**: **시크릿은 태스크가 시작될 때 한 번만 읽힙니다.** 실행 중인 태스크에는 반영되지 않습니다.

```bash
aws ecs update-service \
  --cluster petclinic-cluster \
  --service petclinic-service \
  --force-new-deployment \
  --region us-west-2
```

이 명령으로 태스크를 교체해야 새 값이 적용됩니다. **22-1 절차**를 참고하세요.

---

### Q4. ALB에 접속하면 계속 503이 나옵니다.

**A**: 대상 그룹에 healthy 타깃이 없다는 뜻입니다. 3단계로 좁히세요.

```bash
# ① 태스크가 실행 중인가?
aws ecs describe-services --cluster petclinic-cluster \
  --services petclinic-service --region us-west-2 \
  --query 'services[0].runningCount'
```

`0` 이면 태스크가 안 뜬 것 → **Q1**으로.

```bash
# ② 타깃이 등록됐는가?
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups --names petclinic-tg \
    --region us-west-2 --query 'TargetGroups[0].TargetGroupArn' --output text) \
  --region us-west-2 --query 'TargetHealthDescriptions[*].TargetHealth.State' --output text
```

목록이 비었으면 서비스의 로드밸런싱 설정 누락 → **Step 12 (16-1)**.

`unhealthy` 면 헬스체크 실패 → **③** 으로.

**③ 헬스체크 실패 원인:**

| 확인 | 정상값 |
|------|--------|
| 대상 그룹 포트 | 8080 |
| 대상 그룹 유형 | IP 주소 |
| `petclinic-app-sg` 인바운드 | 8080 ← `petclinic-alb-sg` |
| 상태 검사 유예 기간 | 120초 |

---

### Q5. RDS에 연결이 안 됩니다. `Communications link failure`

**A**: 보안 그룹 체인을 확인하세요.

```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=petclinic-db-sg" \
  --region us-west-2 \
  --query 'SecurityGroups[0].IpPermissions[*].[FromPort,UserIdGroupPairs[0].GroupId]' \
  --output table
```

**기대 결과:** 3306 포트에 `petclinic-app-sg` 의 ID가 보여야 합니다.

IP 대역(`CidrIp`)으로 되어 있으면 **오해 ⑤** 를 다시 읽고 **Step 4 (8-3)** 에서 SG 참조로 바꾸세요.

---

### Q6. 부하를 줬는데 태스크가 안 늘어납니다.

**A**: 세 가지를 확인하세요.

**① 시간을 충분히 기다렸는가?** 스케일 아웃까지 **3~5분**이 걸립니다. 1분 만에 판단하지 마세요.

**② CPU가 실제로 50%를 넘었는가?** ECS 콘솔의 지표 탭에서 확인하세요. 안 넘었으면 부하가 부족한 것입니다.

```bash
hey -z 10m -c 200 http://${ALB_DNS}/    # 동시 연결을 200으로
```

**③ 오토스케일링이 등록됐는가?**

```bash
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs --region us-west-2 \
  --query 'ScalableTargets[0].MaxCapacity'
```

`6` 이 나와야 합니다. 없거나 `2` 면 **Step 13 (17-1)** 로.

---

### Q7. CodeBuild에서 `Cannot connect to the Docker daemon` 이 나옵니다.

**A**: **특권 모드(Privileged)** 가 꺼져 있습니다.

**CodeBuild → `petclinic-build` → 편집 → 환경 → "권한 있음" 체크 → 업데이트**

이 옵션 없이는 컨테이너 안에서 `docker build` 를 할 수 없습니다. **Step 16 (20-5)** 참고.

---

### Q8. CodePipeline의 Deploy 단계가 `image definitions file cannot be found` 로 실패합니다.

**A**: `buildspec.yml` 의 `artifacts` 섹션을 확인하세요.

```yaml
artifacts:
  files:
    - imagedefinitions.json
```

이 부분이 없으면 파일이 다음 단계로 전달되지 않습니다. **Step 16 (20-3)** 으로 돌아가세요.

또한 JSON 안의 컨테이너 이름이 태스크 정의와 일치해야 합니다.

```bash
# 태스크 정의의 컨테이너 이름 확인
aws ecs describe-task-definition --task-definition petclinic-task \
  --region us-west-2 --query 'taskDefinition.containerDefinitions[0].name' --output text
```

**기대 결과:** `petclinic` — buildspec의 `"name":"petclinic"` 과 같아야 합니다.

---

### Q9. 롤링 업데이트 중에 503이 나옵니다.

**A**: 최소 실행 비율이 100 미만일 가능성이 큽니다.

```bash
aws ecs describe-services --cluster petclinic-cluster \
  --services petclinic-service --region us-west-2 \
  --query 'services[0].deploymentConfiguration' --output json
```

**기대 결과:**
```json
{
    "maximumPercent": 200,
    "minimumHealthyPercent": 100
}
```

`minimumHealthyPercent` 가 50이면 배포 중 태스크가 1개로 줄어듭니다. **Step 12 (16-1)** 에서 100으로 바꾸세요.

**19-1 표**를 다시 읽으면 이유가 명확합니다.

---

### Q10. `$ECR_URI` 가 비어 있다고 나옵니다.

**A**: 터미널을 새로 열면 환경변수가 사라집니다. 아래를 다시 실행하세요.

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGION=us-west-2
export ECR_URI=${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/petclinic
export ALB_DNS=$(aws elbv2 describe-load-balancers --names petclinic-alb \
  --region us-west-2 --query 'LoadBalancers[0].DNSName' --output text)
```

`~/.bashrc` 에 넣어 두면 매번 실행하지 않아도 됩니다.

---

## 27. 관련 리소스 이름표

문서 전체에서 사용하는 고유명사입니다. **명령을 실행하기 전 이 표와 대조하세요.**

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

### 환경변수 (Spring Boot)

| 이름 | 출처 |
|------|------|
| `SPRING_PROFILES_ACTIVE` | 일반 환경변수 (`mysql`) |
| `SPRING_DATASOURCE_URL` | 시크릿 (`...:url::`) |
| `SPRING_DATASOURCE_USERNAME` | 시크릿 (`...:username::`) |
| `SPRING_DATASOURCE_PASSWORD` | 시크릿 (`...:password::`) |

---

## 28. 한눈에 보는 작업 순서

### 로컬 PC

```
1.  git clone spring-petclinic
2.  ./mvnw clean package -DskipTests
3.  Dockerfile 작성  (USER app 포함)
4.  .dockerignore 작성
5.  docker build -t petclinic:v1 .
6.  docker run 으로 로컬 200 확인      ← 여기서 안 되면 클라우드도 안 됨
```

### AWS 콘솔 — 인프라 준비

```
7.  VPC 생성  (2AZ, NAT 없음, DNS 켜기)
8.  보안 그룹 4개  (①alb → ②app → ③db → ④endpoint 순서)
9.  VPC 엔드포인트 5개  (Interface 4 + S3 Gateway 1)
10. ECR 리포지토리 petclinic
11. RDS petclinic-db  (프라이빗, 초기 DB명 petclinic)
12. Secrets Manager petclinic/db  (url/username/password)
13. IAM ecsTaskExecutionRole-petclinic  (+ 시크릿 인라인 정책)
```

### 로컬 PC — 이미지 배포

```
14. aws ecr get-login-password | docker login
15. docker tag petclinic:v1 $ECR_URI:v1
16. docker push $ECR_URI:v1
```

### AWS 콘솔 — 서비스 기동

```
17. ECS 클러스터 petclinic-cluster
18. CloudWatch 로그 그룹 /ecs/petclinic
19. 태스크 정의 petclinic-task  (1vCPU/2GB, 시크릿 3개, 헬스체크 90초)
20. 대상 그룹 petclinic-tg  (IP 유형, 8080)
21. ALB petclinic-alb  (퍼블릭 서브넷 2개)
22. ECS 서비스 petclinic-service  (프라이빗, 퍼블릭IP 끄기, 유예 120초, 100/200)
23. ALB DNS로 200 확인            ← 첫 번째 성공 지점
```

### 오토스케일링 · 무중단 배포

```
24. 오토스케일링  (min2 / max6 / CPU 50%)
25. hey 로 부하 → 태스크 2→4 증가 관찰  (3~5분 소요)
26. 부하 종료 → 4→2 감소 관찰  (5분 후)
27. v2 이미지 빌드·푸시
28. 태스크 정의 리비전 2 생성  (이미지 URI만 변경)
29. 서비스 업데이트 → curl 감시하며 503 0건 확인
30. 리비전 1로 롤백 실습 → 다시 2로
```

### CI/CD 자동화

```
31. GitHub에 코드 푸시
32. buildspec.yml 작성  (imagedefinitions.json 생성 포함)
33. CodeBuild petclinic-build  (특권 모드 필수)
34. CodeBuild 역할에 ECR 권한 추가
35. 빌드 단독 테스트 → 성공 확인
36. CodePipeline petclinic-pipeline  (Source→Build→Deploy)
37. 코드 수정 → git push → 자동 배포 확인   ← 최종 성공 지점
```

### 정리

```
38. 서비스 desiredCount 0 → 서비스 삭제
39. 클러스터 · ALB · 대상그룹 · RDS 삭제
40. VPC 엔드포인트 5개 삭제      ← 잊기 쉬움
41. 시크릿 · ECR · 로그그룹 삭제
42. 보안그룹 · VPC · IAM 역할 삭제
43. 다음 날 Cost Explorer 확인
```

---

## 29. 참고 문서

- [Amazon ECS 개발자 안내서](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/Welcome.html)
- [AWS Fargate 시작하기](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/getting-started-fargate.html)
- [태스크 정의에서 민감한 데이터 지정](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/specifying-sensitive-data.html)
- [Amazon ECS 인터페이스 VPC 엔드포인트](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/vpc-endpoints.html)
- [Amazon ECR 인터페이스 VPC 엔드포인트](https://docs.aws.amazon.com/ko_kr/AmazonECR/latest/userguide/vpc-endpoints.html)
- [서비스 Auto Scaling](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/service-auto-scaling.html)
- [CodePipeline으로 ECS 배포 자동화 튜토리얼](https://docs.aws.amazon.com/ko_kr/codepipeline/latest/userguide/ecs-cd-pipeline.html)
- [CodeBuild buildspec 참조](https://docs.aws.amazon.com/ko_kr/codebuild/latest/userguide/build-spec-ref.html)
- [Spring PetClinic 저장소](https://github.com/spring-projects/spring-petclinic)

---

> **문서 버전**: 1.0
> **작성 기준**: `us-west-2` 리전, ECS Fargate 플랫폼 LATEST, Spring Boot PetClinic
> **⚠️ 확인 필요**: AWS 콘솔 UI는 수시로 변경됩니다. 메뉴 명칭이 다르면 각 단계의 CLI 명령으로 대체하거나 공식 문서를 확인하세요.
