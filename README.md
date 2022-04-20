
# CatalogContoller 설치 가이드

## 구성 요소 및 버전
- latest image: quay.io/kubernetes-service-catalog/service-catalog:v0.3.0
- latest version: 0.3.0

## Prerequisites
- TemplateServiceBroker를 사용하고자 하는 경우, install-tsb 프로젝트를 참고하여 선 설치 해주시기 바랍니다.

## 폐쇄망 구축 가이드

1. 사용하는 image를 다운받고 저장합니다.
   - [install-registry 이미지 푸시하기 참조](https://github.com/learncloud/install-registry-docker-ce/blob/main/readme.md)


## 원스탭 설치 가이드
- 아래Step 1 ~ 5를 무시하고 한번에 설치
```bash
cd ~/install-Catalog-Controller-5.0/manifest
chmod 777 install-catalog.sh
./install-catalog.sh install
# ./install-catalog.sh {Fold}

```


## 설치 가이드
1. [설치에 필요한 crd 생성](#Step-1-설치에-필요한-crd-생성)
2. [catalog controller namespace 및 servcice account 생성](#Step-2-catalog-controller-namespace-및-servcice-account-생성)
3. [catalog manager 생성](#Step-3-catalog-manager-생성)
4. [webhook 인증 키 생성](#Step-4-webhook-인증-키-생성)
5. [catalog-webhook 생성](#Step-5-catalog-webhook-생성)
- 비고: install script를 통해 설치하고자 하는 경우에는 /manifest/README.md를 참고 해주세요. 

## Step 1. 설치에 필요한 crd 생성
- 목적 : `CatalogController crd 생성`
- 생성 순서 : 아래 command로 yaml 적용
   1. 설치에 필요한 파일 다운로드
   ```bash
   git clone https://github.com/learncloud/install-Catalog-Controller-5.0.git
   
   ```
  
   2. CatalogController crd 생성
   ```bash
   # 폴더 위치 : ./manifest/yaml/crds/key-mapping
   kubectl apply -f yaml/crds/key-mapping/ 
   
   ```


## Step 2. catalog controller namespace 및 servcice account 생성
- 목적 : `catalog controller namespace 및 servcice account 생성`
- 생성 순서 : 아래 command로 yaml 적용
   ```bash
   #네임스페이스 생성
   kubectl create namespace catalog
   
   # RBAC란 ? 역할 기반 액세스 제어를 말함
   # 폴더 위치 : install-Catalog-Controller-5.0/manifest/yaml
   kubectl apply -f serviceaccounts.yaml
   kubectl apply -f rbac.yaml 
   
   ```
   
## Step 3. catalog manager 생성
- 목적 : `catalog manager 생성`
- 생성 순서 : 아래 command로 yaml 적용
    ```bash
    # 반드시 각 파일에 image 항목의 {imageRegistry}와 {catalogVersion}은  사용자 환경에 맞게 수정
    # {imageRegistry}/kubernetes-service-catalog/service-catalog:v{catalogVersion} 
    # → 192.168.178.17:5000/kubernetes-service-catalog/service-catalog:v0.3.0 
    # → or quay.io/kubernetes-service-catalog/service-catalog:v0.3.0
    
    # 파일 위치 : ./manifest/yaml/controller-manager-deployment.yaml
    kubectl apply -f controller-manager-deployment.yaml
    
    # 파일위치 : ./manifest/yaml/controller-manager-service.yaml
    # service ip를 달아주는 yaml파일
    kubectl apply -f controller-manager-service.yaml
    
    ```
    
 - 참고 : local docker내에 image가 없을 경우 업로드 하는방법
   ```bash
   docker pull quay.io/kubernetes-service-catalog/service-catalog:v0.3.0
   docker tag quay.io/kubernetes-service-catalog/service-catalog:v0.3.0 192.168.178.17:5000/quay.io/kubernetes-service-catalog/service-catalog:v0.3.0
   docker push 192.168.178.17:5000/quay.io/kubernetes-service-catalog/service-catalog:v0.3.0

   ```
   
## Step 4. webhook 인증 키 생성
- 목적 : `webhook 인증에 필요한 키를 생성`
- 생성 순서 : 아래 command 적용
    
    ```bash
    cd ~/install-Catalog-Controller-5.0
    openssl genrsa -out rootca.key 2048
    
    # 비고: .rnd 파일이 없어서 해당 명령어 실행이 안되는 경우 /etc/ssl/openssl.cnf 파일의  "RANDFILE = $ENV::HOME/.rnd" 부분을 주석처리 합니다.
    openssl req -x509 -new -nodes -key rootca.key -sha256 -days 3650 -subj /C=KO/ST=None/L=None/O=None/CN=catalog-catalog-webhook -out rootca.crt
    
    # v3.ext 생성
    cp manifest/ca/v3.ext ~/install-Catalog-Controller-5.0
    
    openssl req -new -newkey rsa:2048 -sha256 -nodes -keyout server.key -subj /C=KO/ST=None/L=None/O=None/CN=catalog-catalog-webhook -out server.csr
    openssl x509 -req -in server.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial -out server.crt -days 3650 -sha256 -extfile ./v3.ext
    openssl base64 -in rootca.crt -out key0
    openssl base64 -in server.crt -out cert
    openssl base64 -in server.key -out key
    
    ```
    
## Step 5. catalog-webhook 생성
- 목적 : `catalog-webhook 생성`
- 생성 순서 : 아래 command 적용
    ```bash
    cd ~/install-Catalog-Controller-5.0/manifest/yaml
    kubectl apply -f webhook-register.yaml
    #비고: 해당 파일의 변수를 7번에서 생성한 키로 대체 합니다. (단, 생성된 키 공백은 모두 지우셔야 합니다.)
    {{ b64enc $ca.Cert }} --> key0 내부 값으로 대체
    # ex) key0=$(awk 'NF {sub(/\r/, ""); printf "%s",$0;}' key0)
    {{ b64enc $cert.Cert }} --> cert 내부 값으로 대체
    # ex) cert=$(awk 'NF {sub(/\r/, ""); printf "%s",$0;}' cert)
    {{ b64enc $cert.Key }} --> key 내부 값으로 대체
    # ex) key=$(awk 'NF {sub(/\r/, ""); printf "%s",$0;}' key)
    
    # 비고: 반드시!!, 각 파일에 image 항목의 {imageRegistry}와 {catalogVersion}은  사용자 환경에 맞게 수정해야 합니다. 
    # ({imageRegistry}/kubernetes-service-catalog/service-catalog:v{catalogVersion})
    
    kubectl apply -f webhook-deployment.yaml ([파일위치](./manifest/yaml/webhook-deployment.yaml))
    
    kubectl apply -f webhook-service.yaml ([파일위치](./manifest/yaml/webhook-service.yaml))

    ```
    

## 삭제 가이드
1. [사용중인 리소스 제거](#Step-1-사용중인-리소스-제거)
2. [설치 리소스 제거](#Step-2-설치-리소스-제거)
3. [CRD 제거](#Step-3-CRD-제거)

## Step 1~3을 무시하고 진행

```bash
cd ~/install-Catalog-Controller-5.0/manifest
chmod 777 install-catalog.sh
./install-catalog.sh uninstall
# ./install-catalog.sh {install or uninstall }
# {Fold}부분에 뭘 쓰느냐에 따라 설치될지 제거될지 결정됨. 스크립트에install  or uninstall 조건문 걸려있음

```




--------------------------------------------------------

## Step 1. 사용중인 리소스 제거
- 목적 : `사용중인 리소스 차례로 제거`
- 삭제 순서 : 아래 command 순서대로 적용
    - kubectl delete servicebinding --all --all-namespaces
    - kubectl delete serviceinstance --all --all-namespaces
    - kubectl delete servicebinding --all --all-namespaces
    - kubectl delete clusterservicebroker --all --all-namespaces
    - kubectl delete servicebroker --all --all-namespaces

## Step 2. 설치 리소스 제거
- 목적 : `설치에 필요한 pod, service 등의 리소스 제거`
- 삭제 순서 : 설치에 진행했던 yaml 파일 역순으로 적용
    - kubectl delete -f webhook-service.yaml ([파일](./manifest/yaml/webhook-service.yaml))
    - kubectl delete -f webhook-deployment.yaml ([파일](./manifest/yaml/webhook-deployment.yaml))
    - kubectl delete -f webhook-register.yaml ([파일](./manifest/yaml/webhook-register.yaml))
    - kubectl delete -f controller-manager-service.yaml ([파일](./manifest/yaml/controller-manager-service.yaml))
    - kubectl delete -f controller-manager-deployment.yaml ([파일](./manifest/yaml/controller-manager-deployment.yaml))
    - kubectl delete -f rbac.yaml ([파일](./manifest/yaml/rbac.yaml))
    - kubectl delete -f serviceaccounts.yaml ([파일](./manifest/yaml/serviceaccounts.yaml))
    - kubectl delete namespace catalog

## Step 3. CRD 제거
- 목적 : `CRD 제거`
- 삭제 순서 : 아래 command로 yaml 적용
    - kubectl delete -f crds/key-mapping/ ([폴더](./manifest/yaml/crds/key-mapping)) 
