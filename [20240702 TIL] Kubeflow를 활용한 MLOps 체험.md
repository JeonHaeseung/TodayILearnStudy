# TIL - Kubeflow를 활용한 MLOps 체험

## 🌊Knative, Istio, KServe

---

[데보션 글](https://devocean.sk.com/blog/techBoardDetail.do?ID=163645)에 따르면, AI 모델은 Language/Framework 종속적이고, 지속적으로 학습을 통해 발전해야 하기 때문에 응용에 직접 포함되기 보다 Backed API형태로 제공하는 것이 효율적입니다. 이렇게 모델을 API 형태로 제공할 수 있는 방법이 쿠버네티스에서는 KNative, Istio, KServe 등의 기술입니다.

모델을 API 형태로 사용할 때 Istio는 마이크로서비스 관리를, KNative는 서버리스 어플리케이션 관리를, 그리고 KServe는 ML/AI 모델 관리를 맡습니다. Istio+KNative인 Serverless 레이어 위에서 KServe가 작동하면서 모델에 접근하고, 그 결과를 반환해서 돌려줍니다.

![1. KNative, Istio, KServe](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/ab666555-6035-406a-a18b-cb0b62cb0b98)

좀 더 자세한 흐름도를 살펴보면 다음과 같습니다.

- **Istio**는 Gateway로서, 들어오거나 나가는 HTTP/TCP 연결을 수신하는 역할을 합니다. 노출해야 할 포트-프로토콜 세트를 결정할 수 있습니다.
- **KNative**는 Istio의 추론 요청을 받아 새로운 revision을 생성하고, 트래픽에 따라서 Pod를 생성하거나 내리는(scale up-down) 역할을 합니다.
- **KServe**는 AI 모델이 돌아가는 서버 자체를 container로 구성해 관리합니다. KServe는 Tensorflow, Pytorch 등 다양한 프레임워크로 구현된 모델을 모두 관리할 수 있습니다.

```bash
  +----------------------+        +-----------------------+      +--------------------------+
  |Istio Virtual Service |        |Istio Virtual Service  |      | K8S Service              |
  |                      |        |                       |      |                          |
  |sklearn-iris          |        |sklearn-iris-predictor |      | sklearn-iris-predictor   |
  |                      +------->|-default               +----->| -default-$revision       |
  |                      |        |                       |      |                          |
  |KServe Route          |        |Knative Route          |      | Knative Revision Service |
  +----------------------+        +-----------------------+      +------------+-------------+
   Knative Ingress Gateway           Knative Local Gateway                    Kube Proxy
   (Istio gateway)                   (Istio gateway)                          |
                                                                              |
                                                                              |
  +-------------------------------------------------------+                   |
  |  Knative Revision Pod                                 |                   |
  |                                                       |                   |
  |  +-------------------+      +-----------------+       |                   |
  |  |                   |      |                 |       |                   |
  |  |kserve-container   |<-----+ Queue Proxy     |       |<------------------+
  |  |                   |      |                 |       |
  |  +-------------------+      +--------------^--+       |
  |                                            |          |
  +-----------------------^-------------------------------+
                          | scale deployment   |
                 +--------+--------+           | pull metrics
                 |  Knative        |           |
                 |  Autoscaler     |-----------
                 |  KPA/HPA        |
                 +-----------------+
```

구조도로 다시 확인해보면 다음과 같습니다.

![2. Flow of KNative, Istio, KServe](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/101734e0-f5c5-4a8d-8824-1c48591eafcb)

### 버전 설정

- Kubernetes: 1.29
- Istio: 1.21.0
- KNative: 1.14.1

### 참고 문헌

아래 글을 참고해서 설치를 진행했습니다.

https://devocean.sk.com/blog/techBoardDetail.do?ID=164093

### Istio 설치하기

Istio란 클라우드 기반 애플리케이션을 구성하는 다양한 **마이크로서비스**를 관리하기 위한 솔루션입니다. Istio는 마이크로서비스가 서로 통신하고 데이터를 공유하는 방법을 지원합니다.

먼저  helm chart를 통해서 Istio를 다운로드 받습니다.

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/manifests/charts/base/crds/crd-all.gen.yaml
helm repo update

# Knative의 auto injection 비활성화
nano istiod_1.21.0_default_values.yaml
global:
  proxy:
    autoInject: disabled
```

그 후 Istio의 CRD(Custom Resource Definitions)를 적용시킵니다.

```bash
kubectl create namespace istio-system
helm upgrade -i istio-base istio/base --version 1.21.0 -n istio-system -f istiod_1.21.0_default_values.yaml
helm upgrade -i istiod istio/istiod --version 1.21.0 -n istio-system -f istiod_1.21.0_default_values.yaml --wait
```

외부에서 서비스에 접근하기 위한 Istio Ingress Gateway를 따로 만들어주어야 합니다. 이때 Istio 사이드카 프록시를 Pod에 자동으로 주입하도록 설정합니다.

```bash
nano gateway_1.21.0_default_values.yaml
podAnnotations:
  prometheus.io/port: "15020"
  prometheus.io/scrape: "true"
  prometheus.io/path: "/stats/prometheus"
  inject.istio.io/templates: "gateway"
  sidecar.istio.io/inject: "true"
kubectl create ns istio-ingress
helm upgrade -i istio-ingress istio/gateway --version 1.21.0 -n istio-ingress -f gateway_1.21.0_default_values.yaml
```

사이드카 프록시란 다음과 같은 역할을 합니다. 보통 A 서비스에서 B서비스를 호출하기 위해서는 서비스 코드의 변경이 필요합니다(예시: Springboot에서 FeignClient를 통해서 외부 API를 호출하려면, FeignClient가 서비스 코드에 구현되어 있어야 합니다). 그러나 Istio를 사용하면 서비스 코드 변경 없이 로드밸런싱, 인증, 모니터링 등을 적용할 수 있습니다. 그 이유는 Istio가 경량화 프록시(Proxy)를 사이드카(sidecar) 방식으로 배치하여 Load Balancing, Dynamic Request Routing 등을 지원하기 때문입니다.

![3. Sidecar](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/1fc98982-95d8-44c4-8b63-4cc55de699e3)

이제 확인해보면 잘 작동하는 것을 볼 수 있습니다.

```bash
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
```

```bash
NAME                      READY   STATUS    RESTARTS   AGE
istiod-7fcf9b89b7-ptpbc   1/1     Running   0          4m2s

NAME                             READY   STATUS    RESTARTS   AGE
istio-ingress-5c995b5d96-z8p8p   1/1     Running   0          17s
```

### KNative 설치하기

KNative는 scale-to-zero, 자동 확장, 클러스터 내 빌드, 이벤트 프레임워크 등의 기능 등, **서버리스 애플리케이션**을 빌드하고 실행하는 데 필수적인 구성요소를 제공하는 솔루션입니다.

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.14.1/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.14.1/serving-core.yaml
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.14.1/net-istio.yaml
```

그대로 설치하면 Isito gateway와 연동되지 않아서, 아래 내용을 추가해서 edit해 주어야 합니다.

```bash
$ kubectl edit gateway -n knative-serving knative-ingress-gateway
...
spec:
  selector:
    istio: ingressgateway
    istio: ingress          # 추가

$ kubectl edit gateway -n knative-serving knative-local-gateway
...
spec:
  selector:
    istio: ingressgateway
    istio: ingress          # 추가
```

그 후 KNative에서 제공하는 샘플 프로그램인 helloworld-go를 배포합니다.

```bash
git clone https://github.com/knative/docs knative-docs
cd knative-docs/code-samples/serving/hello-world/helloworld-go

nano service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Go Sample v1"

kubectl apply -f service.yaml
```

이제 해당 프로그램을 확인해보면, 배포가 모두 완료된 것을 볼 수 있습니다.

```bash
$ kubectl get route
NAME            URL                                              READY   REASON
helloworld-go   http://helloworld-go.default.svc.cluster.local   True
```

```bash
curl -H "Host: helloworld-go.default.svc.cluster.local" http://35.243.111.196
```

정리하자면, Istio Ingress Gateway를 통해서 들어온 트래픽은 쿠버네티스의 Routing Rule에 의해서 어떤 도메인 이름이 어느 KNative Pod로 가야 하는지 결정됩니다. 구조도로 보면 아래와 같습니다.

![4. Istio Ingress Gateway](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/c61aa588-27bf-4eea-91a3-6f39fcd44e07)

### Zero Scale

KNative는 Serverless 솔루션이기 때문에, 사용되지 않을 때는 Zero로 Scale down됩니다. 트래픽이 1분 이상 없을 때는 zero replicas를 사용하다가, 트래픽이 들어오면 Pod의 수가 늘어납니다.

```bash
$ kubectl get deploy
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-go-00001-deployment   0/0     0            0           29m
```

### KServe 설치하기

KServe는 Tensorflow, XGBoost, ScikitLearn, PyTorch, Huggingface Transformer/LLM 모델에 대한 높은 추상화 인터페이스를 제공하여 prediction, pre-processing, post-processing등을 편리하게 돕는 솔루션입니다.

```bash
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.8.0/kserve.yaml
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.8.0/kserve-runtimes.yaml
```

## ☁️GCP 세팅하기

---

Minikube 배포의 쿠브플로는 최소한 12기가바이트의 메모리와 4개의 GPU가 할당되어야 하므로, 로컬에서 실행하기에는 어려울 수 있습니다. 이 실습에서는 Google Cloud를 사용해서 kubeflow를 구축하겠습니다.

### GCP 프로젝트 생성

구글 클라우드에서는 모든 리소스가 프로젝트 아래에 종속되기 때문에, 실습을 하기 전 프로젝트를 생성합니다.

![5. GCP project](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/ddb33caa-5ea5-4887-a7fd-b01c67202750)


### 클라우드 쉘 활성화하기

Cloud Shell은 5GB 홈 디렉터리를 갖춘 Debian 기반의 가상 머신으로, Gcloud 명령줄 도구, kubectl 등의 유틸리티가 미리 로드된 온라인 터미널 역할을 합니다.

![6. GCP cloud shell](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/369720ca-c84f-43a6-b87d-28889a5a3569)

### 로그인하기

클라우드 쉘에서 구글 계정으로 로그인하기 위해서 다음 명령어를 입력하고, 로그인을 진행합니다.

```bash
gcloud auth login
```

![7. GCP login](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/e2d73dbc-9089-4ef6-bf04-758ba92fa8cd)

```bash
You are now logged in as [sample@gmail.com].
Your current project is [kubeflow-428202].
```

### API 활성화하기

Kubeflow를 사용하기 위해서는 다음과 같은 API를 활성화해야 합니다. Cloud Shell 터미널에 아래 명령어를 복사붙여넣기해 API를 활성화합니다.

```bash
gcloud services enable \
  anthos.googleapis.com \
  serviceusage.googleapis.com \
  compute.googleapis.com \
  container.googleapis.com \
  iam.googleapis.com \
  servicemanagement.googleapis.com \
  cloudresourcemanager.googleapis.com \
  ml.googleapis.com \
  iap.googleapis.com \
  sqladmin.googleapis.com \
  meshconfig.googleapis.com \
  krmapihosting.googleapis.com \
  servicecontrol.googleapis.com \
  endpoints.googleapis.com \
  cloudbuild.googleapis.com
```

```bash
Operation "operations/acf.p2-183446103859-c30e5c09-467f-4652-9e17-8d9c07a8ed90" finished successfully.
```

### **OAuth 동의 화면 세팅하기**

Kubeflow는 사용자들이 로컬 머신에서 접근할 수 있는 여러 개의 웹 어플리케이션(Jupyter notebook, pipeline UI 등)과 API(Katib 등)으로 이루어져 있습니다. 이때 사용자의 아이덴티티를 확인할 수 있고, 아이덴티티에 적합한 정보를 보여줄 수 있는 인증과 승인 과정이 필요합니다. 예를 들어서 다른 사용자가 만든 주피터 노트북을 볼 수 없도록 만들어야 합니다.

GCP에서는 인증과 승인을 Cloud IAP(Cloud Identity-Aware Proxy) 방식으로, 사용자나 서비스 계정을 기반에 권한을 부여하게 만들어 진행할 수 있습니다. 이 Cloud IAP를 사용하기 위해서 OAuth 클라이언트를 사용할 수 있습니다. 아래는 IAP가 작동하는 흐름도입니다.

![8. Cloud IAP](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/98daf53f-6555-4c1f-a280-f94e2c43afec)

OAuth 동의 화면으로 이동해 셋업합니다.

![9. GCP OAuth](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/d38fc251-480f-4fad-a927-8ae4a8b2299f)

다음과 같은 설정만 필수적으로 입력해주면 됩니다.

- **애플리케이션 이름:** 애플리케이션 이름을 원하는 대로 입력합니다.
- **사용자 지원 이메일:** 공개 연락처로 표시할 이메일 주소나 자신이 소유한 Google 그룹을 선택합니다.
- **승인된 도메인:** 아래와 같은 형식으로 입력합니다(따로 사용하는 도메인이 있다면 추가적으로 등록합니다).
    
    ```bash
    <GCP project>.cloud.goog
    ```

    ![10. GCP OAuth domain](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/70445eb7-57e4-4511-b0a2-18b0ee0b4d1b)    

나머지 설정은 전부 기본으로 세팅합니다.

### 사용자 인증 정보 설정하기

사용자 인증 정보에서 OAuth 클라이언트 ID를 생성합니다.

![11. OAuth client ID](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/036503ee-9b13-4980-a2db-1b0cbf167a5e)

다음 설정을 세팅하고 저장합니다.

- **애플리케이션 유형:** 웹 애플리케이션을 선택합니다.
- **이름:** OAuth 클라이언트 ID를 식별하는 데 도움이 되는 이름을 입력합니다.

우선 OAuth 클라이언트 ID를 만든 후, 클라이언트 ID가 생성되면 편집을 클릭해 다음 값을 설정합니다.

- **승인된 리디렉션 URI:** 다음과 같이 설정합니다. 여기서 `CLIENT_ID`는 생성된 OAuth 클라이언트 ID 값입니다.
    
    ```bash
    https://iap.googleapis.com/v1/oauth/clientIds/<CLIENT_ID>:handleRedirect
    ```
    

생성이 완료되면 다음과 같이 목록에서 확인 가능합니다.

![12. List of OAuth client ID](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/7da4d771-c740-49de-a480-5ce2a5e470ef)

## 🚀Kubeflow 설치하기

---

### 관리 클러스터 배포

먼저 아래 명령어로 GPU를 사용 가능한 리전을 확인합니다. 여기서는 `nvidia-l4`를 사용 가능한 `asia-northeast3`로 세팅했습니다.

```bash
gcloud compute accelerator-types list
```

아래 코드로 gcloud 구성요소 설치, 관리 클러스터를 설치할 Github 리포지토리 복제, 환경변수 구성를 진행합니다.

```bash
# 필요한 구성 요소 가져오기
sudo apt-get install kubectl google-cloud-cli-kpt google-cloud-cli-anthoscli google-cloud-cli
# 관련 클러스터 설정 가져오기
git clone https://github.com/googlecloudplatform/kubeflow-distribution.git 
cd kubeflow-distribution
git checkout master
cd management
# 클러스터 이름, 프로젝트 아이디, 위치 설정
cat << EOF > env.sh
MGMT_PROJECT=kubeflow-1-428202
MGMT_NAME=mgt-cluster
LOCATION=asia-northeast3
EOF
source env.sh
kpt fn eval --image gcr.io/kpt-fn/apply-setters:v0.1 ./kptconfig -- \
    name="${MGMT_NAME}" \
    gcloud.core.project="${MGMT_PROJECT}" \
    location="${LOCATION}" \

kpt fn eval --image gcr.io/kpt-fn/apply-setters:v0.1 ./manifests --fn-config ./kptconfig/kpt-setter-config.yaml --truncate-output=false
# 클러스터 배포 시작
make create-cluster
make create-context
kubectl config delete-context ${NAME} || echo "Context ${NAME} will be created."
gcloud anthos config controller get-credentials ${NAME} --location=${LOCATION}
kubectl config rename-context $(kubectl config current-context) ${NAME}
make grant-owner-permission
MGMT_PROJECT=${PROJECT} $(PACKAGE_DIR)/hack/grant_owner_permission.sh
```

### Kubeflow 클러스터 배포

```bash
# Kubeflow 설정 가져오기
cd kubeflow-distribution
git checkout master
cd kubeflow
# 업스트림 설정
bash ./pull-upstream.sh
# kubeflow/env.sh의 변수들 모두 값 채우기
export KF_PROJECT=kubeflow-1-428202 # google cloud project id
export KF_PROJECT_NUMBER=1098488231933 # gcloud projects describe --format='value(projectNumber)' "${KF_PROJECT}"
export ADMIN_EMAIL=springinflearn1886@gmail.com #  Kubeflow admin's email address
export MGMT_NAME=mgt-cluster # name of the cluster
export CLIENT_ID= # OAuth client ID
export CLIENT_SECRET= # OAuth client secret
source env.sh
bash ./kpt-set.sh
# Cloud Config Connector 승인
export MGMT_PROJECT=kubeflow-1-428202
make apply-kcc
# 클러스터 배포
make apply
# 자격 증명 향하게 하기
gcloud container clusters get-credentials "${KF_NAME}" --zone "${ZONE}" --project "${KF_PROJECT}"
gcloud projects add-iam-policy-binding "${KF_PROJECT}" --member=user:<EMAIL> --role=roles/iap.httpsResourceAccessor
kubectl -n kubeflow get all
```

생성된 결과는 다음과 같습니다.

```bash
NAME                                                         READY   STATUS    RESTARTS        AGE
pod/admission-webhook-deployment-54df8774c8-z5ls2            1/1     Running   0               8m5s
pod/cache-deployer-deployment-598bdfddbb-pqxm4               2/2     Running   1 (5m2s ago)    6m26s
pod/cache-server-956767dd5-5pl9q                             2/2     Running   0               6m25s
pod/centraldashboard-787f75d7df-d5sck                        2/2     Running   0               7m55s
pod/cloudsqlproxy-7764cbd9c6-wj56r                           2/2     Running   1 (5m21s ago)   6m25s
pod/jupyter-web-app-deployment-c9fb9467b-xvwxn               2/2     Running   0               7m47s
pod/kserve-controller-manager-854d4fb9c8-gznml               2/2     Running   0               5m5s
pod/kserve-models-web-app-75fdc88564-ts86f                   2/2     Running   0               5m4s
pod/kubeflow-pipelines-profile-controller-6f96745f5d-gbhqj   1/1     Running   0               6m47s
pod/metacontroller-0                                         1/1     Running   0               8m17s
pod/metadata-envoy-deployment-bc48d4554-t8wbc                1/1     Running   0               6m47s
pod/metadata-grpc-deployment-64df6fb54f-fzdnv                2/2     Running   4 (5m21s ago)   6m23s
pod/metadata-writer-689874db67-w9zlr                         2/2     Running   0               6m31s
pod/minio-b5b9b5ccc-gxd9q                                    2/2     Running   0               6m31s
pod/ml-pipeline-5569544dc7-gm9lf                             2/2     Running   0               6m30s
pod/ml-pipeline-persistenceagent-bfbf94866-h87lz             2/2     Running   0               6m31s
pod/ml-pipeline-scheduledworkflow-798b6cd456-8rgcb           2/2     Running   0               6m30s
pod/ml-pipeline-ui-b97c7874d-gmhjr                           2/2     Running   0               6m30s
pod/ml-pipeline-viewer-crd-74998d7d7c-cbccs                  2/2     Running   1 (6m24s ago)   6m30s
pod/ml-pipeline-visualizationserver-6c47d74fd6-htdnf         2/2     Running   0               6m30s
pod/notebook-controller-deployment-7f44fdc4c5-cch89          2/2     Running   2 (7m40s ago)   7m46s
pod/profiles-deployment-75d9dcdbc7-tbzpd                     3/3     Running   2 (7m52s ago)   8m
pod/tensorboard-controller-deployment-6cf44dc486-rvtgm       3/3     Running   1 (7m34s ago)   7m38s
pod/tensorboards-web-app-deployment-675c66b476-6v457         2/2     Running   0               7m38s
pod/training-operator-77dc7667fc-gjzgr                       1/1     Running   0               7m19s
pod/volumes-web-app-deployment-996f7bdbb-nn792               2/2     Running   0               7m42s
pod/workflow-controller-56cd6ff675-5p59n                     2/2     Running   1 (5m34s ago)   6m30s

NAME                                                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/admission-webhook-service                                   ClusterIP   10.45.89.46    <none>        443/TCP             8m5s
service/cache-server                                                ClusterIP   10.45.92.151   <none>        443/TCP             6m34s
service/centraldashboard                                            ClusterIP   10.45.91.252   <none>        80/TCP              7m56s
service/jupyter-web-app-service                                     ClusterIP   10.45.87.242   <none>        80/TCP              7m49s
service/kserve-controller-manager-metrics-service                   ClusterIP   10.45.87.133   <none>        8443/TCP            5m8s
service/kserve-controller-manager-service                           ClusterIP   10.45.89.83    <none>        8443/TCP            5m8s
service/kserve-models-web-app                                       ClusterIP   10.45.95.100   <none>        80/TCP              5m8s
service/kserve-webhook-server-service                               ClusterIP   10.45.88.88    <none>        443/TCP             5m7s
service/kubeflow-pipelines-profile-controller                       ClusterIP   10.45.86.192   <none>        80/TCP              6m34s
service/metadata-envoy-service                                      ClusterIP   10.45.89.216   <none>        9090/TCP            6m34s
service/metadata-grpc-service                                       ClusterIP   10.45.82.85    <none>        8080/TCP            6m34s
service/minio-service                                               ClusterIP   10.45.84.99    <none>        9000/TCP            6m34s
service/ml-pipeline                                                 ClusterIP   10.45.83.171   <none>        8888/TCP,8887/TCP   6m33s
service/ml-pipeline-ui                                              ClusterIP   10.45.92.36    <none>        80/TCP              6m34s
service/ml-pipeline-visualizationserver                             ClusterIP   10.45.89.105   <none>        8888/TCP            6m33s
service/mysql                                                       ClusterIP   10.45.91.179   <none>        3306/TCP            6m33s
service/notebook-controller-service                                 ClusterIP   10.45.86.185   <none>        443/TCP             7m49s
service/profiles-kfam                                               ClusterIP   10.45.91.6     <none>        8081/TCP            8m1s
service/tensorboard-controller-controller-manager-metrics-service   ClusterIP   10.45.88.214   <none>        8443/TCP            7m39s
service/tensorboards-web-app-service                                ClusterIP   10.45.91.194   <none>        80/TCP              7m39s
service/training-operator                                           ClusterIP   10.45.89.55    <none>        8080/TCP            7m21s
service/volumes-web-app-service                                     ClusterIP   10.45.84.26    <none>        80/TCP              7m44s
service/workflow-controller-metrics                                 ClusterIP   10.45.94.25    <none>        9090/TCP            6m33s

NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/admission-webhook-deployment            1/1     1            1           8m7s
deployment.apps/cache-deployer-deployment               1/1     1            1           6m48s
deployment.apps/cache-server                            1/1     1            1           6m48s
deployment.apps/centraldashboard                        1/1     1            1           7m58s
deployment.apps/cloudsqlproxy                           1/1     1            1           6m48s
deployment.apps/jupyter-web-app-deployment              1/1     1            1           7m53s
deployment.apps/kserve-controller-manager               1/1     1            1           5m17s
deployment.apps/kserve-models-web-app                   1/1     1            1           5m16s
deployment.apps/kubeflow-pipelines-profile-controller   1/1     1            1           6m48s
deployment.apps/metadata-envoy-deployment               1/1     1            1           6m47s
deployment.apps/metadata-grpc-deployment                1/1     1            1           6m47s
deployment.apps/metadata-writer                         1/1     1            1           6m46s
deployment.apps/minio                                   1/1     1            1           6m46s
deployment.apps/ml-pipeline                             1/1     1            1           6m45s
deployment.apps/ml-pipeline-persistenceagent            1/1     1            1           6m46s
deployment.apps/ml-pipeline-scheduledworkflow           1/1     1            1           6m46s
deployment.apps/ml-pipeline-ui                          1/1     1            1           6m45s
deployment.apps/ml-pipeline-viewer-crd                  1/1     1            1           6m45s
deployment.apps/ml-pipeline-visualizationserver         1/1     1            1           6m45s
deployment.apps/notebook-controller-deployment          1/1     1            1           7m53s
deployment.apps/profiles-deployment                     1/1     1            1           8m2s
deployment.apps/tensorboard-controller-deployment       1/1     1            1           7m41s
deployment.apps/tensorboards-web-app-deployment         1/1     1            1           7m41s
deployment.apps/training-operator                       1/1     1            1           7m23s
deployment.apps/volumes-web-app-deployment              1/1     1            1           7m45s
deployment.apps/workflow-controller                     1/1     1            1           6m45s

NAME                                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/admission-webhook-deployment-54df8774c8            1         1         1       8m7s
replicaset.apps/cache-deployer-deployment-598bdfddbb               1         1         1       6m48s
replicaset.apps/cache-server-956767dd5                             1         1         1       6m48s
replicaset.apps/centraldashboard-787f75d7df                        1         1         1       7m58s
replicaset.apps/cloudsqlproxy-7764cbd9c6                           1         1         1       6m48s
replicaset.apps/jupyter-web-app-deployment-c9fb9467b               1         1         1       7m53s
replicaset.apps/kserve-controller-manager-854d4fb9c8               1         1         1       5m17s
replicaset.apps/kserve-models-web-app-75fdc88564                   1         1         1       5m16s
replicaset.apps/kubeflow-pipelines-profile-controller-6f96745f5d   1         1         1       6m47s
replicaset.apps/metadata-envoy-deployment-bc48d4554                1         1         1       6m47s
replicaset.apps/metadata-grpc-deployment-64df6fb54f                1         1         1       6m47s
replicaset.apps/metadata-writer-689874db67                         1         1         1       6m46s
replicaset.apps/minio-b5b9b5ccc                                    1         1         1       6m46s
replicaset.apps/ml-pipeline-5569544dc7                             1         1         1       6m45s
replicaset.apps/ml-pipeline-persistenceagent-bfbf94866             1         1         1       6m46s
replicaset.apps/ml-pipeline-scheduledworkflow-798b6cd456           1         1         1       6m45s
replicaset.apps/ml-pipeline-ui-b97c7874d                           1         1         1       6m45s
replicaset.apps/ml-pipeline-viewer-crd-74998d7d7c                  1         1         1       6m45s
replicaset.apps/ml-pipeline-visualizationserver-6c47d74fd6         1         1         1       6m45s
replicaset.apps/notebook-controller-deployment-7f44fdc4c5          1         1         1       7m53s
replicaset.apps/profiles-deployment-75d9dcdbc7                     1         1         1       8m2s
replicaset.apps/tensorboard-controller-deployment-6cf44dc486       1         1         1       7m41s
replicaset.apps/tensorboards-web-app-deployment-675c66b476         1         1         1       7m41s
replicaset.apps/training-operator-77dc7667fc                       1         1         1       7m23s
replicaset.apps/volumes-web-app-deployment-996f7bdbb               1         1         1       7m45s
replicaset.apps/workflow-controller-56cd6ff675                     1         1         1       6m45s

NAME                              READY   AGE
statefulset.apps/metacontroller   1/1     8m18s
```

GKE에서 확인해보면 다음과 같이 클러스터가 2개 생성된 것을 볼 수 있습니다.

![13. managment & kubeflow cluster](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/dd530a93-cb43-4780-8533-2deb909c9d9b)

## 👩‍💻Kubeflow 사용하기

---

### Kubeflow를 사용하는 이유

ML/AI 모델은 많은 리소스를 사용하기 때문에 OOM(Out of Memory) 에러를 일으키거나, 서버를 일시적으로 다운시키기도 합니다. 문제는 이럴 경우 서버에서 같이 실행되고 있는 다른 프로그램이 강제 종료될 수 있는데, 이럴 때 일반적인 서비스처럼 Scale up이나 Scale out을 쉽게 하기 어렵고 리소스를 폭발적으로 쓰는 구간이 예측하리 어려우므로, 애초에 모델에 대한 리소스를 제한해서 줄 필요가 있습니다.

### Kubeflow 화면 띄우기

아래 명령어로 kubeflow 서비스 엔드포인트를 확인하고 웹 브라우저에 입력해 접근합니다.

```bash
# URI 엔드포인트 확인하기
kubectl -n istio-system get ingress
```

```bash
NAME            CLASS    HOSTS                                             ADDRESS       PORTS   AGE
envoy-ingress   <none>   kubeflow.endpoints.kubeflow-1-428202.cloud.goog   34.54.7.103   80      12m
```

IAP를 사용했기 때문에, 구글 계정으로 로그인 및 인증을 할 수 있습니다. 위에서 IAM에 바인딩해준 구글 계정을 사용해서 접근해야 합니다(다른 구글 계정은 접근이 차단됩니다).

![14. IAP login](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/be8fda24-e460-4375-82d3-9e085293e6aa)

Kubeflow에 들어가면 다음과 같이 네임스페이스를 생성 가능합니다. 사용자 개인의 네임스페이스를 생성되고, 해당 사용자의 리소스는 그 네임스페이스에서 관리됩니다. 이를 통해서 사용자별 리소스 관리가 가능합니다.

![15. kubeflow user namespace](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/1cb97a4e-3ac0-4e85-9c96-f202be5a18e6)

Kubeflow 홈페이지에 들어가면 다음과 같이 웹 페이지가 보이는 것을 확인할 수 있습니다.

![16. kubeflow homepage](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/94904b15-1c56-4771-bf2b-475802fcf2cc)

### Jupyter notebook 띄우기

주피터 노트북은 데이터 분석, 기계 학습, 대화형 실험 및 문서화 작업에 유용한 대화형 개발 환경입니다. 파이썬 스크립트를 바로 실행 시켜서 AI 모델을 개발하는 경우도 있지만, 주피터 노트북 파일을 사용해서 개발하는 경우도 있습니다.

주피터 노트북을 사용하면 전체 프로그램을 작은 코드 조각인 셀{cell) 단위로 나누어 실행할 수도 있고, 실행되는 동안 모든 변수와 데이터는 메모리에 저장되어 유지되기 때문에 특정 셀에서 에러가 발생하더라도 전체 프로그램을 다시 실행할 필요가 없다는 점입니다.

Notebooks로 이동해 새로운 노트북을 생성합니다.

![17. create new notebook](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/2ef60bbd-e5d8-4dc3-bc30-7bb09a9b825d)

아래와 같이 CPU, MEM, GPU를 얼마나 사용할지를 결정할 수 있습니다. 여기서는 가벼운 Transformer 모델로 감정 분석을 진행해 볼 것이므로, CPU만 4개를 사용하도록 하겠습니다.

![18. notebook cpu](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/ec6d9579-1622-478b-b700-dcc394d1a1bb)

![19. notebook gpu](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/889a9691-a5af-4ac2-a064-67d9f5b388ef)

여기에서 사용할 주피터 노트북 코드는 다음과 같습니다. IMDB 데이터 세트(영화 리뷰를 모아놓은 데이터 세트)로 Transformer 모델을 train시키고, 새롭게 주어지는 영화 리뷰의 긍정/부정 emotion을 측정하는 모델입니다.

[transformer-emotion-analysis.ipynb](https://drive.google.com/file/d/11jHzhMThlWKHENFildbMkrxQ1bQILX3j/view?usp=sharing)

결과를 보면, 부정적인 영화 리뷰에 88.69% negative 분석을 한 것을 확인할 수 있습니다.

![20. result of notebook](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/fa05446e-8c01-44de-908f-8e484780cd84)

### pipeline UI 띄우기

kubeflow pipeline이란 실험, 작업 및 실행을 관리하고 추적하기 위한 사용자 인터페이스(UI)입니다. 즉, 머신 러닝 파이프라인의 오케스트레이션을 위한 서비스로 파이프라인을 재사용하고 re-run하기 간편해 다양한 시행착오를 시도할 수 있게 합니다.

아래 홈 대시보드에 있는 [Tutorial] Data passing in python components를 사용해서 간단한 파이프라인을 만들어보겠습니다.

![21. make new pipeline](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/14101563-6970-493a-b45f-84873336e5a2)

아래에서 볼 수 있듯이, 각 데이터를 전처리하고 전처리된 데이터를 모델에 전달해 해당 데이터로 train되도록 하는 파이프라인입니다. 해당 파이프라인을 실행하기 위해서 실험을 생성합니다.

![22. graph of new pipeline](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/9c4c31a8-899b-4503-82d8-c402fa1685a5)

![23. new run of pipeline](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/558bf2a6-c70b-4ea6-a210-cb3821126b22)

run하면 다음과 같이 전체 파이프라인이 실행되어, 데이터 전처리부터 모델 train까지 순서대로 진행되는 것을 볼 수 있습니다.

![24. result of pipeline](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/13b17f0b-6d38-468f-8dc7-be7785499d08)

### 참고: 하이퍼파라미터란?

**파라미터**: 모델이 훈련 중에 학습하는 가중치(weight)로, 미리 변경하거나 조정할 수 없습니다.

**하이퍼파라미터**: 모델이 훈련하기 전에 연구자가 설정할 수 있는 변수들(레이어 개수, loss function, 배치 사이즈, epoch 숫자 등)을 의미합니다.

**하이퍼파라미터 튜닝 알고리즘(검색 알고리즘)**: 연구자는 모델에 어떤 하이퍼파라미터 조합을 사용할지를 결정해야 하는데, 적절한 하이퍼파라미터 조합은 선험적이기 보다 경험적으로 알게 되는 경우가 많습니다. 따라서 튜닝 알고리즘을 통해서 하이퍼파라미터를 선택하게 됩니다.

- eg. Grid Search는 모든 하이퍼파라미터 조합을 다 시도해보는 알고리즘으로, A파라미터에 대해 1, 2, 3의 선택지, B 파라미터에 대해서 a, b, c의 선택지가 있다면 [1-a, 1-b, 1-c, 2-a, …, 3-c]를 모두 시도해 봅니다.

### Katib UI 띄우기

Katib은 하이퍼파라미터를 조정하기 위한 시스템으로, 개발자가 작성한 어플리케이션의 프레임워크에 독립적으로 사용 가능합니다. Katib은 Hyperparameter Tuning , Early Stopping 및 Neural Architecture Search 등을 통해서 AI 모델의 성능을 최적화합니다.

- **Experiment**: 실험은 단일 튜닝 실행을 의미하며, 모델의 정확도 값 등 최적화하고 싶은 대상인 Objective, 최적화를 위해서 고려하는 모든 하이퍼파라미터 값의 집합인 Search space, 최적의 하이퍼파라미터 값을 검색할 때 사용하는 알고리즘인 Search algorithm로 이루어져 있습니다.
- **Suggestion**: 제안은 하이퍼파라미터 튜닝 프로세스가 제안한 하이퍼파라미터 값의 집합으로, Katib은 제안된 값 세트를 평가하기 위해 trial을 만듭니다.
- **Trial**: 시도는 실험에서 하이퍼파라미터 튜닝 프로세스의 한 반복을 의미하며, 실험은 목표 또는 구성된 최대 시행 횟수에 도달할 때까지 trial을 시도합니다.
- **Worker job**: 작업은 시행을 평가하고 객관적 가치를 계산하기 위해 실행되는 프로세스입니다. 모든 유형의 Kubernetes 리소스 또는 Kubernetes CRD가 작업이 될 수 있습니다.

다음 깃허브에서 Katib 실험을 진행하기 위한 예시 yaml을 복사합니다.

[raw.githubusercontent.com](https://raw.githubusercontent.com/kubeflow/katib/master/examples/v1beta1/hp-tuning/random.yaml)

yaml 내용을 확인해보면 다음과 같습니다.

```bash
---
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  namespace: kubeflow
  name: random
spec:
  objective:
    type: minimize
    goal: 0.001
    objectiveMetricName: loss
  algorithm:
    algorithmName: random
  parallelTrialCount: 3
  maxTrialCount: 12
  maxFailedTrialCount: 3
  parameters:
    - name: lr
      parameterType: double
      feasibleSpace:
        min: "0.01"
        max: "0.05"
    - name: momentum
      parameterType: double
      feasibleSpace:
        min: "0.5"
        max: "0.9"
  trialTemplate:
    primaryContainerName: training-container
    trialParameters:
      - name: learningRate
        description: Learning rate for the training model
        reference: lr
      - name: momentum
        description: Momentum for the training model
        reference: momentum
    trialSpec:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          spec:
            containers:
              - name: training-container
                image: docker.io/kubeflowkatib/pytorch-mnist-cpu:latest
                command:
                  - "python3"
                  - "/opt/pytorch-mnist/mnist.py"
                  - "--epochs=1"
                  - "--batch-size=16"
                  - "--lr=${trialParameters.learningRate}"
                  - "--momentum=${trialParameters.momentum}"
                resources:
                  limits:
                    memory: "1Gi"
                    cpu: "0.5"
            restartPolicy: Never
```

아래 줄만 편집해서 위에서 생성했던 사용자 개인 네임스페이스를 사용하도록 변경합니다.

```bash
namespace: haeseungjeon
```

해당 예제를 배포합니다.

```bash
kubectl --context=kubeflow apply -f random.yaml
```

위의 실험은 다음과 같은 조건에서 진행되고 있습니다.

**튜닝하는 하이퍼파라미터**

- **lr (learning rate):** 가중치가 업데이트되는 크기를 결정하는 하이퍼파라미터입니다.
- **momentum:** 이전의 가중치 업데이트의 방향을 고려하여 현재의 업데이트를 가속화하는 하이퍼파라미터입니다.

**검색 알고리즘**

- **random:** 랜덤하게 하이퍼파라미터 조합을 생성합니다.

**실험 설정 세부 정보**

- **objective**: 목표는 `loss`라는 메트릭을 최소화하는 것입니다(goal: 0.001).
- **parallelTrialCount**: 동시에 실행되는 실험의 수(3개)를 의미합니다.
- **maxTrialCount**: 최대 실험 횟수(12회)를 의미합니다.
- **maxFailedTrialCount**: 실패할 수 있는 최대 실험 횟수(3회)를 의미합니다.

참고로, Kubeflow 대시보드에서 Experiments(AutoML)에서 Katib을 GUI를 통해서 편리하게 사용할 수도 있습니다. 그러나 여기서는 yaml을 직접 배포하는 것으로 테스트해보도록 하겠습니다.

![25. new experiment](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/bb1ca342-b822-49c8-bf73-36cf51a433a4)

이제 Experiments(AutoML)에서 실험이 진행되는 것을 관찰하고, 실험 결과를 확인할 수 있습니다.

![26. result of new experiment](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/d924ab00-2abb-4d6f-95e1-f49b4616fde6)

lr와 momentum이 둘 다 높을 수록 loss가 작다는 점을 알 수 있습니다.

![27. graph of new experiment](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/6e3c3475-2524-4259-b064-afd89e515998)

## 📜관련 자료

**GCP에 kubeflow 배포를 위한 세팅**

- https://medium.com/@sunwoopark/google-cloud-platform에서-kubeflow-구축하기-상세-가이드-1bf6c4f77c5a
- https://cloud.google.com/iap/docs/concepts-overview?hl=ko
- https://googlecloudplatform.github.io/kubeflow-gke-docs/dev/docs/

**Istio, KNative**

- https://manhyuk.github.io/istio/
- https://gruuuuu.github.io/cloud/knative-hands-on/
- https://devocean.sk.com/blog/techBoardDetail.do?ID=164093
- https://bcho.tistory.com/1322
- https://www.alibabacloud.com/blog/simplifying-knative-application-deployment-and-access_597430

**KServe**

- https://kserve.github.io/website/0.9/modelserving/control_plane/
- https://kserve.github.io/website/0.9/developer/debug/#debug-kserve-request-flow
- https://devocean.sk.com/blog/techBoardDetail.do?ID=163645
- https://docs.kakaocloud.com/tutorial/machine-learning-ai/kubeflow-model-serving

**pipeline**

- https://www.kubeflow.org/docs/components/pipelines/legacy-v1/introduction/

**Katib**

- https://v1-5-branch.kubeflow.org/docs/components/katib/overview/#experiment
- https://v1-5-branch.kubeflow.org/docs/components/katib/hyperparameter/
