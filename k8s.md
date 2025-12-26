  

결론(요약)

  

  

백엔드 개발자가 쿠버네티스(K8s)를 잘 쓰려면 배포 단위는 Pod/Deployment, 접근은 Service/Ingress, 상태는 Probe, 설정은 ConfigMap/Secret, 리소스는 requests/limits + HPA, 데이터는 PV/PVC/StatefulSet, 보안은 ServiceAccount/RBAC/NetworkPolicy, 무중단은 RollingUpdate+PDB, 관측은 Logs/Metrics/Tracing를 기본 축으로 잡으면 됩니다. 여기에 graceful shutdown(SIGTERM), readiness 지연(워밍업), DB 마이그레이션(Job/CronJob), **스케줄 작업의 단일 실행(leader election)**만 추가로 챙기면, 실무 대부분을 안정적으로 커버합니다.

  

  

  

  

1) K8s 핵심 오브젝트 한눈에

  

  

- Namespace: 환경/팀 경계.
- Pod: 컨테이너 실행 최소 단위(사이드카·initContainer 가능).
- ReplicaSet/Deployment: 선언적 무중단 배포(rollout, rollback).
- StatefulSet: 고정 네임/스토리지(데이터 베이스·큐 등).
- DaemonSet: 모든 노드에 1개씩(에이전트/로그 수집기).
- Job/CronJob: 배치/예약 작업(마이그레이션, ETL).
- Service: Pod 고정 접근점(ClusterIP/NodePort/LoadBalancer).
- Ingress(+IngressClass/Controller): L7 라우팅/도메인·TLS 종단.
- ConfigMap/Secret: 설정/민감정보 주입(환경변수·파일).
- PVC/PV/StorageClass: 영속 스토리지 추상화(RWO/RWX/CSI).
- ServiceAccount/RBAC: Pod 실행 권한/클러스터 권한 제어.
- NetworkPolicy: Pod 간 네트워크 허용 리스트 방식 통제.
- HorizontalPodAutoscaler(HPA): 부하에 따른 복제수 자동 조정.
- PodDisruptionBudget(PDB): 계획중단 시 최소 가용성 보장.
- Affinity/TopologySpread/Taints: 스케줄링 제어(분산·전용노드).
- ResourceQuota/LimitRange: 네임스페이스별 자원 한도.

  

  

  

  

  

2) 백엔드 서비스 관점의 필수 포인트

  

  

  

무중단/안정성

  

  

- readinessProbe: 트래픽 받기 전 애플리케이션 워밍업 완료 신호.
- livenessProbe: 프로세스 비정상 시 재시작.
- graceful shutdown: K8s는 SIGTERM → (기본 30s) → SIGKILL. 스프링은 server.shutdown=graceful, 커넥션 종료/큐 배출.
- PDB: 노드 드레인·업그레이드 시 최소 N개 파드 살아있게.
- RollingUpdate 전략: maxUnavailable/maxSurge로 카나리성 롤링.

  

  

  

설정/비밀/릴로드

  

  

- ConfigMap/Secret: env/file 마운트. Secret은 base64 인코딩(보안 스토리지(예: KMS)와 함께).
- 설정 롤오버: CM/Secret 변경 → 새 해시를 파드 label/env로 주입하여 재시작 트리거.

  

  

  

네트워킹/외부노출

  

  

- Service 타입  
    

- ClusterIP: 내부 통신(기본).
- NodePort: 디버그용/간단 외부노출.
- LoadBalancer: 클라우드 L4 로드밸런서.

-   
    
- Ingress: 호스트/경로 라우팅, TLS, 웹소켓, 리라이팅. 컨트롤러(NGINX/Traefik/ALB 등) 필요.

  

  

  

스토리지/상태

  

  

- Stateless 우선: 세션은 외부(레디스 등).
- StatefulSet + PVC: 파드별 고정 볼륨, 순서/정체성 보장.
- Access Mode: RWO(단일 마운트), RWX(다중). DB는 보통 RWO.

  

  

  

오토스케일/자원

  

  

- requests/limits(CPU/메모리): 스케줄링/스로틀·OOM 방지의 기준.
- HPA: CPU/메모리/커스텀 메트릭(예: RPS, 큐 길이)로 스케일.
- VPA/라이트사이징: 권장치 계산(프로덕션은 HPA와 동시 수동 권장모드가 일반적).

  

  

  

관측/운영

  

  

- 로그: stdout/stderr → 수집기(Fluent Bit/Vector) → Elasticsearch/Cloud Logging.
- 메트릭: cAdvisor/Metric Server/Prometheus → 대시(그라파나).
- 트레이싱: OpenTelemetry(스프링: Micrometer Tracing) → Jaeger/Tempo.
- 이벤트/상태 확인: kubectl describe로 실패 이유(이미지 풀/스케줄링/프로브/리소스).

  

  

  

  

  

3) 스프링 부트(코틀린/자바) 실전 체크리스트

  

  

- Actuator: /actuator/health를 readiness/liveness로 분리(예: DB 의존은 readiness만).
- Graceful: shutdown hook에서 HTTP 서버 종료 → 인플라이트 처리 → DB/메시지 종료.
- DB 마이그레이션: 애플리케이션 시작 전에 Job으로 flyway/liquibase 실행(한 번만!).
- 스케줄러 단일 실행: 다중 레플리카 시 leader election(ShedLock 등).
- 커넥션 풀: maxLifetime < DB 타임아웃, readiness 전 워밍업(캐시/클래스로드).
- 프로파일 전략: spring.profiles.active ↔ Namespace/Helm values 연동.
- 리소스 가정: CPU는 코어 수 대비 적절한 requests(예: 250m)부터 실측으로 조정.

  

  

  

  

  

4) 배포 전략(간단 패턴)

  

  

- RollingUpdate(기본): 안전·표준. 대부분 상황에서 권장.
- Blue/Green: 두 스택 병행 → 스위치(서비스 셀렉터 or Ingress 라우팅).
- Canary: 소량 트래픽 테스트(서비스 셀렉터 가중치/Ingress/서비스메시/Argo Rollouts).

  

  

  

  

  

5) 보안 필수

  

  

- Least privilege: 네임스페이스 단위 ServiceAccount + Role/RoleBinding.
- 이미지 보안: 레지스트리 스캔, non-root, readOnlyRootFilesystem.
- NetworkPolicy: 인바운드/아웃바운드 최소화(Egress DB만 허용).
- Secret 관리: KMS/KVP(예: External Secrets Operator)로 주기적 싱크.

  

  

  

  

  

6) 장애/이슈 트러블슈팅 흐름

  

  

7. kubectl get events / describe pod로 원인 파악(이미지 오류, PullBackOff, OOMKilled, CrashLoopBackOff).
8. 프로브 실패면 endpoint 확인(health path, 포트, 초기지연).
9. OOM → 메모리 사용 추적 → requests/limits 재조정/메모리 릭 점검.
10. 스로틀링 → CPU requests 상향, HPA 메트릭/타깃 확인.
11. 네트워크 → Service 셀렉터/Endpoints/NetworkPolicy/Ingress 규칙 점검.
12. 디스크/권한 → PVC 상태/AccessMode, SA/RBAC 권한.

  

  

  

  

  

13) 필수 

kubectl

 치트시트 (비행기 모드용)

  

# 리소스 보기

kubectl get pods,svc,ingress,deploy,sts,hpa,pvc -A

kubectl describe pod <pod>

kubectl logs <pod> [-c <container>] --since=10m -f

  

# 배포/롤백

kubectl rollout status deploy/<name>

kubectl rollout history deploy/<name>

kubectl rollout undo deploy/<name>

  

# 디버깅

kubectl exec -it <pod> -- sh

kubectl get events --sort-by=.lastTimestamp

  

  

  

  

8) 최소 예제 YAML (스프링 부트 웹)

  

  

Deployment + Service + Ingress + HPA + PDB

apiVersion: apps/v1

kind: Deployment

metadata:

  name: api

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }

  selector:

    matchLabels: { app: api }

  template:

    metadata:

      labels: { app: api }

    spec:

      serviceAccountName: api-sa

      containers:

        - name: app

          image: ghcr.io/example/api:1.0.0

          ports: [{ containerPort: 8080 }]

          envFrom:

            - configMapRef: { name: api-config }

            - secretRef: { name: api-secret }

          readinessProbe:

            httpGet: { path: /actuator/health/readiness, port: 8080 }

            initialDelaySeconds: 15

            periodSeconds: 5

          livenessProbe:

            httpGet: { path: /actuator/health/liveness, port: 8080 }

            initialDelaySeconds: 30

            periodSeconds: 10

          resources:

            requests: { cpu: "250m", memory: "512Mi" }

            limits:   { cpu: "1",    memory: "1Gi" }

---

apiVersion: v1

kind: Service

metadata:

  name: api

spec:

  type: ClusterIP

  selector: { app: api }

  ports: [{ port: 80, targetPort: 8080 }]

---

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

  name: api

  annotations:

    nginx.ingress.kubernetes.io/proxy-body-size: "20m"

spec:

  ingressClassName: nginx

  rules:

    - host: api.example.com

      http:

        paths:

          - path: /

            pathType: Prefix

            backend: { service: { name: api, port: { number: 80 } } }

  tls:

    - hosts: [api.example.com]

      secretName: api-tls

---

apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: api

spec:

  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }

  minReplicas: 3

  maxReplicas: 10

  metrics:

    - type: Resource

      resource:

        name: cpu

        target: { type: Utilization, averageUtilization: 70 }

---

apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: api

spec:

  minAvailable: 2

  selector:

    matchLabels: { app: api }

  

  

  

  

9) Job/CronJob 예제 (DB 마이그레이션/정기 작업)

  

apiVersion: batch/v1

kind: Job

metadata: { name: flyway-migrate }

spec:

  backoffLimit: 2

  template:

    spec:

      restartPolicy: Never

      containers:

        - name: migrate

          image: ghcr.io/example/flyway:9

          envFrom: [{ secretRef: { name: db-secret } }]

          args: ["-url=jdbc:postgresql://db:5432/app","-locations=filesystem:/sql","migrate"]

---

apiVersion: batch/v1

kind: CronJob

metadata: { name: nightly-report }

spec:

  schedule: "0 2 * * *"

  jobTemplate:

    spec:

      template:

        spec:

          restartPolicy: OnFailure

          containers:

            - name: task

              image: ghcr.io/example/batch:1.0.0

              args: ["generate-report"]

  

  

  

  

10) 헬름/환경별 구성 팁

  

  

- Helm values로 이미지 태그, 리소스, 도메인, 시크릿 레퍼런스만 바꿔 동일 차트 재사용.
- Kustomize는 오버레이(Base→Dev/Prod)로 패치 관리.
- 버전/구성 Drift 방지엔 **GitOps(Argo CD)**가 가장 단순·견고.

  

  

  

  

  

11) 흔한 함정 & 베스트 프랙티스

  

  

- 프로브 오남용: liveness로 비즈니스 장애 감지 금지(과잉 재시작). readiness로만 의존성 체크.
- limits만 설정: requests 누락 시 스케줄링 비예측/CPU 스로틀링.
- ConfigMap 핫리로드 기대: 기본은 재시작 필요(컨트롤러/사이드카 사용 검토).
- 세션 로컬 저장: 파드 재시작/스케일 시 세션 유실 → 외부 스토어.
- DB 연결 폭증: 스케일아웃 시 커넥션 풀 총량이 DB 한계 초과 → 풀 상한 관리/프록시( PgBouncer 등).
- Cron 다중 실행: 레플리카 N개에서 동시에 실행 → 전용 워크로드/락 필요.
- 노드 드레인 테스트 부재: PDB 없으면 계획중단에도 트래픽 드랍.

  

  

  

  

  

12) 공부 순서(기내 1~2시간 코스)

  

  

13. Pod/Deployment/Service/Ingress 관계도 그려보기
14. Probe + Graceful 동작 이해(초기지연, 종료 시그널)
15. requests/limits/HPA 숫자 감 잡기
16. ConfigMap/Secret로 설정 분리해 재배포
17. PDB/롤링업데이트로 무중단 보장
18. Job/CronJob로 마이그레이션/배치 분리
19. NetworkPolicy/RBAC로 최소권한 모델 설계

  

  

원하면 이 내용을 PDF 한 장짜리 치트시트로 정리해 드릴게요.

좋아요 — 친구님처럼 스프링 부트 기반 백엔드 개발자가 쿠버네티스를 “실제로 써야 하는 시점”과 “어떤 역할을 하는지 감이 오도록” 설명 중심으로 다시 구성해드릴게요.

핵심 개념 → 용도/필요 시점 → 실무 예시 형태로, 비행기 안에서도 읽기 좋게 요약했습니다.

  

  

  

  

🧭 백엔드 개발자가 꼭 알아야 할 쿠버네티스 핵심 개념

  

  

(실무 이해 중심 버전)

  

  

  

  

1️⃣ Pod — “컨테이너가 실제로 돌아가는 집”

  

  

- 무엇: 하나 이상의 컨테이너가 함께 실행되는 최소 단위 (보통 1개 컨테이너).
- 언제 필요: 내가 만든 스프링부트 애플리케이션 이미지를 실제로 띄울 때.
- 예시: myapp:1.0.0 Docker 이미지를 Pod으로 띄우면, 쿠버네티스가 노드에 컨테이너를 실행함.
- 비유: 도커 컨테이너 1개를 쿠버네티스가 대신 관리하는 ‘방’

  

  

👉 백엔드 개발자 입장: “kubectl logs로 파드 로그를 본다”가 디버깅의 시작.

  

  

  

  

2️⃣ Deployment — “버전 관리와 무중단 배포 담당”

  

  

- 무엇: Pod 여러 개를 일정 개수로 유지하고, 새 버전으로 바꿀 때 자동으로 롤링 업데이트.
- 언제 필요: 코드 수정 → 새 버전 배포할 때 (서버 끊기지 않게).
- 예시: replicas: 3이면 파드 3개를 계속 유지하며, 새 이미지로 교체 시 순차 업데이트.
- 비유: 배포 담당자이자 오케스트라 지휘자.

  

  

👉 백엔드 개발자 입장: Deployment는 “내 서버 버전 1 → 1.1로 바꿔주는 매니저”.

  

  

  

  

3️⃣ Service — “Pod에 접근할 수 있는 고정 주소”

  

  

- 무엇: Pod는 죽었다 살아나며 IP가 바뀜 → 고정된 이름과 포트로 접근하기 위한 L4 엔드포인트.
- 언제 필요: 다른 서비스나 외부에서 내 API 서버를 호출할 때.
- 예시: Service(api)가 Pod(app) 3개 중 살아있는 것에 로드밸런싱.
- 비유: ‘회사 내 내선번호’ — 사람(Pod)은 바뀌지만 번호(Service)는 그대로.

  

  

👉 백엔드 개발자 입장: “DB 서버 주소 대신 db-service:5432로 접근”

  

  

  

  

4️⃣ Ingress — “외부 트래픽 들어오는 문(L7 라우터)”

  

  

- 무엇: 도메인, 경로, HTTPS 등 L7 라우팅 설정.
- 언제 필요: 사용자 브라우저나 외부 시스템이 내 API를 호출할 때.
- 예시: api.example.com → Service(api)로 연결, TLS 적용도 여기서.
- 비유: “회사 정문 + 안내데스크”.

  

  

👉 백엔드 개발자 입장: “외부 요청이 Nginx Ingress → 내 Spring Boot 서버까지 들어오는 구조”

  

  

  

  

5️⃣ ConfigMap & Secret — “설정 관리”

  

  

- 무엇: 환경 변수나 설정 파일을 외부에서 주입하는 오브젝트.
- 언제 필요: 환경별(DB URL, Redis 주소, Feature Toggle 등) 설정 분리할 때.
- 예시: spring.datasource.url을 ConfigMap에 정의하고 배포 시 주입.
- 비유: “환경설정 노트북”. Secret은 비밀번호 노트.

  

  

👉 백엔드 개발자 입장: 코드 안에서 하드코딩 대신 env로 읽기.

  

  

  

  

6️⃣ Probe (Liveness / Readiness) — “건강검진”

  

  

- 무엇: 애플리케이션이 정상인지 쿠버네티스가 주기적으로 확인.
- 언제 필요: 앱이 뻗었거나 아직 준비 안 된 상태에서 트래픽을 차단해야 할 때.
- 예시: /actuator/health/readiness OK → 서비스 시작, 실패 시 트래픽 안 받음.
- 비유: “헬스체크 간호사”.

  

  

👉 백엔드 개발자 입장: Spring Actuator로 readiness/liveness 분리 구현.

  

  

  

  

7️⃣ Resource Requests / Limits — “서버 스펙 예약”

  

  

- 무엇: 파드가 필요한 CPU/메모리를 지정 → 스케줄러가 배치 결정을 내림.
- 언제 필요: 같은 노드에서 여러 파드가 자원 경쟁할 때.
- 예시: requests.cpu=250m, limits.memory=512Mi → 이 파드는 최소 0.25코어, 최대 512Mi 사용.
- 비유: “서버 자원 예약표”.

  

  

👉 백엔드 개발자 입장: API 부하 테스트로 적정 자원 산정 → HPA와 함께 튜닝.

  

  

  

  

8️⃣ Horizontal Pod Autoscaler (HPA) — “자동 확장 매니저”

  

  

- 무엇: CPU나 요청량에 따라 파드 수 자동 증가/감소.
- 언제 필요: 특정 시간대 트래픽이 급증할 때.
- 예시: CPU 사용률 70% 넘으면 파드 3개 → 6개로 증가.
- 비유: “퇴근시간엔 콜센터 상담원 2배 투입”.

  

  

👉 백엔드 개발자 입장: 트래픽 피크 시 스케일아웃으로 안정적 처리.

  

  

  

  

9️⃣ StatefulSet + PVC — “데이터가 있는 서비스용 (DB, MQ 등)”

  

  

- 무엇: Pod 이름·볼륨이 고정된 상태로 유지되는 배포 방식.
- 언제 필요: DB, Redis, Kafka처럼 데이터가 각 인스턴스에 귀속될 때.
- 예시: mysql-0, mysql-1 각각 자신의 PVC를 붙여서 실행.
- 비유: “자리 지정석이 있는 좌석 시스템”.

  

  

👉 백엔드 개발자 입장: 보통 직접 운영하기보다 클라우드 매니지드 DB로 대체하지만, 원리를 이해해야 함.

  

  

  

  

🔟 Job / CronJob — “한 번 혹은 정기적으로 일하는 일꾼”

  

  

- 무엇: 한번만 실행(Job), 주기적으로 실행(CronJob).
- 언제 필요: DB 마이그레이션, 로그 정리, 정산 배치 등.
- 예시: 0 3 * * * → 매일 새벽 3시에 배치 실행.
- 비유: “자동 예약 스크립트”.

  

  

👉 백엔드 개발자 입장: Spring Batch → CronJob으로 옮길 때 사용.

  

  

  

  

11️⃣ ServiceAccount / RBAC — “권한 관리”

  

  

- 무엇: Pod가 쿠버네티스 API나 다른 리소스에 접근할 때의 ID/권한.
- 언제 필요: 특정 Pod만 ConfigMap 수정, S3 접근 등 제한할 때.
- 예시: serviceAccountName: myapp-sa + RoleBinding으로 권한 부여.
- 비유: “사내 사번 + 접근권한표”.

  

  

👉 백엔드 개발자 입장: CI/CD 파이프라인이나 S3 접근 시 최소권한 원칙 중요.

  

  

  

  

12️⃣ NetworkPolicy — “네트워크 방화벽”

  

  

- 무엇: Pod 간 통신 허용/차단 규칙.
- 언제 필요: 보안 강화 (DB 접근은 API서버만 허용 등).
- 예시: app=frontend → app=api만 허용, 나머지는 차단.
- 비유: “내부망 ACL”.

  

  

👉 백엔드 개발자 입장: QA/Prod 간 데이터 접근 차단 정책을 여기에 설정.

  

  

  

  

13️⃣ Rolling Update + PodDisruptionBudget — “무중단 배포와 안전장치”

  

  

- 무엇: 새 버전으로 순차 교체, 동시에 최소 개수 유지 보장.
- 언제 필요: 운영 배포 중 서비스 끊김 방지.
- 예시: maxUnavailable: 0, minAvailable: 2 → 항상 2개 이상 살아있게.
- 비유: “식당 테이블을 한 번에 다 안 바꾸고 순차 교체”.

  

  

👉 백엔드 개발자 입장: 롤링 업데이트 중에도 API 요청이 중단되지 않도록 readinessProbe + graceful shutdown 필수.

  

  

  

  

14️⃣ Observability (로그, 메트릭, 트레이스) — “눈과 귀”

  

  

- 무엇: 파드 상태·로그·성능을 수집/시각화.
- 언제 필요: 장애 분석, 성능 병목 찾기.
- 예시: Prometheus + Grafana로 CPU/RPS 추적, Loki로 로그 검색.
- 비유: “건강 모니터링 대시보드”.

  

  

👉 백엔드 개발자 입장: /actuator/metrics, 로그 stdout → 중앙 로깅으로 보냄.

  

  

  

  

15️⃣ Graceful Shutdown — “예쁘게 종료하기”

  

  

- 무엇: 쿠버네티스가 파드를 종료할 때 SIGTERM을 보내고 일정시간 후 SIGKILL.
- 언제 필요: 연결 끊김 없이 기존 요청 마무리 후 종료하려고.
- 예시: Spring Boot의 server.shutdown=graceful 설정.
- 비유: “손님 밥 다 먹고 나갈 때까지 불 끄지 않기”.

  

  

👉 백엔드 개발자 입장: readiness→false 후 기존 요청 처리 완료 후 exit.

  

  

  

  

16️⃣ 실무 예시 흐름 (API 서버 배포 시)

  

|   |   |   |   |
|---|---|---|---|
|단계|쿠버네티스 리소스|역할|백엔드 개발자가 신경쓸 포인트|
|1|Docker Image|앱 빌드 결과물|Dockerfile 최적화, JVM 옵션|
|2|Deployment|앱 실행 관리|replicas, rollout 전략|
|3|Service|로드밸런싱|내부 호출 주소|
|4|Ingress|외부 노출|도메인, HTTPS|
|5|ConfigMap/Secret|설정 분리|환경별 DB 연결 정보|
|6|Probe|헬스체크|/actuator/health|
|7|HPA|오토스케일|CPU 타깃값, 최대 인스턴스|
|8|PDB|무중단 배포|최소 가용 Pod 개수|
|9|Logs/Metrics|모니터링|장애 감지, 리소스 확인|

  

  

  

  

17️⃣ 친구님 기준 학습 우선순위 (비행기 공부용)

  

  

1️⃣ Pod / Deployment / Service / Ingress 구조 이해

2️⃣ Probe + Graceful Shutdown

3️⃣ ConfigMap / Secret

4️⃣ Resource Requests + HPA

5️⃣ RollingUpdate + PDB

6️⃣ Job / CronJob

7️⃣ Observability

  

  

  

원하신다면 이 내용을 한 페이지짜리 PDF(비행기용 요약판) 으로 만들어드릴게요.

👉 “예”라고 하면, 제목/표/도식 포함한 Kubernetes for Backend Developer – Quick Guide (한글판) PDF로 변환해드리겠습니다. 만들까요?