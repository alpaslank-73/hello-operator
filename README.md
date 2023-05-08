Hello Operator:

Bu ornek "cluster-wide scope" bir operator ornegi. Namespace scoped olan operatorler ise tek bir namespace'e kurulup oradaki resource'lari yonetiyor. Benim ornek baska namespace yaratabiliyor.

#### Install operator-sdk

$ export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)

$ export OS=$(uname | awk '{print tolower($0)}')

$ export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.28.1

$ curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}

$ chmod +x operator-sdk_linux_amd64

$ sudo ln operator-sdk_linux_amd64 /usr/local/bin/operator-sdk

#### Install ansible-runner: 

Bu RPM sadece operatoru local olarak (test amacli) calistiracaksan gerekli. 

$ subscription-manager repos enable ansible-automation-platform-2.2-for-rhel-8-x86_64-rpms

$ sudo yum install -y ansible-runner

#### Kustomize aslinda sart degil ancak kurmak istersen:

$ curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash    # sonra "sudo ln kuztomize /usr/local/bin/"

-------------------
# Create operator #
-------------------
Bu operator Ansible role ile quay.io/alpaslank/hello:latest imajini kullanarak bir deployment yapiyor.

$ mkdir hello-operator ; cd hello-operator

$ operator-sdk init --domain quay.io --plugins ansible  

$ operator-sdk create api --group mygroup --version v1 --kind Hello --generate-role 

rolu'u update et: (Role ve operator dosyalari github'da mevcut)

$ cat roles/hello/tasks/main
...
$ cat roles/hello/defaults/main.yml

#### Lokal olarak run etmek icin ansible-runner gerekiyor:

$ oc login -u admin -p redhat 
$ oc new-project test    

$ make install (bu adim custom resource'u (customresourcedefinition.apiextensions.k8s.io/hellos.mygroup.quay.io) yaratiyor)

$ make run   

#### "make run" calisiyorken test icin asagidaki gibi bir "hello" resource yarat:

*** Dikkat asagidaki islem ilk denemelerinde hata verebilir; CR'un tanimlanmasi zaman alabiliyor)

$ oc create -f config/samples/mygroup_v1_hello.yaml# "hello-sample" isimli "hello" resource'u test namespace'inde ancak yaratilan deployment vs "mynamespace" namespace'inde oluyor (mynamespace default ancak override edilebilir)

$ oc get hello -n NAMESPACE ; oc get deploy,svc,route -n DEST_NAMESPACE

Istersen scale etmeyi de goster:

$ oc edit hello -n test ==> replicas: 2

Silmek icin:

$ oc delete -n hello myhello 
$ oc delete project mynamespace

*** Local operatoru durdur ==> CTRL+C (yukarida baslatmistin)

#### Operatoru cluster uzerine kurmak icin (OLM'den bagimsiz olarak):

Cluster uzerine deploy edebilmek icin operator imajina ihtiyacimiz var var 

Bunun icin Makefile'i duzelt (duzeltilen 2 satirin biri operator imaji digeri bundle imaji ile ilgili):

$ vim Makefile
...
IMAGE_TAG_BASE ?= quay.io/hello-operator  ==> IMAGE_TAG_BASE ?= quay.io/alpaslank/hello-operator
...
IMG ?= controller:latest ==> IMG ?= $(IMAGE_TAG_BASE):$(VERSION)

#### community.okd modullerini kullanabilmek icin:

$ vim requirements.yaml   
...
  - name: community.okd
    version: "2.3.0"

#### Default olarak "controller-manager" serviceaccount'u "manager-role" cluster-role'une sahip ve bu rolde proje yaratma yetkisi yok. Bunu duzeltmek icin (biraz abartip cluster-admin yetkisi tanimladin):

$ vim config/rbac/role.yaml
...
# Full yetki - Alpaslan
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'


*** Asagidaki islem onesinde quay.io'da "hello-operator" reposunu yarat ve "public" yap! (hatta "hello-operator-bundle"i da yarat ve public yap)

$ podman login quay.io 

$ sudo yum install -y docker

$ make docker-build docker-push

$ make deploy ==> hello-operator-system isminde bir namespace'e kuruyor (serviceaccount, service, deployment, role vs de yaratiyor)

#### Eger birseyler duzelteceksen Makefile'daki 
...
"VERSION ?= 0.0.3"  ==> satirini update et ve:

$ make docker-build docker-push

$ make undeploy ==> Bu adima gerek yok aslinda; 

$ make deploy IMG=quay.io/alpaslank/hello-operator:0.0.3

$ oc create -f config/samples/mygroup_v1_hello.yaml

#### Deploy your Operator with OLM

*** Bu islem oncesinde de "hello-operator-bundle" reposunu quay.io'da yarat ve public yap!

$ operator-sdk olm install ==> Bu adim "zaten kurulu" diyerek hata veriyor (DO380 setup'inda)

## Yetki konusunda sorun olmamasi icin rolu update et (aslinda en basta yaptiysan buna gerek olmamasi lazim)

$ vim bundle/manifests/hello-operator-manager-role_rbac.authorization.k8s.io_v1_clusterrole.yaml
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'

$ make bundle   ==> Bundle ile ilgili dizini yaratacak

# Ayrica asagidaki dosyayi da su sekilde duzelt:

$ vim ./bundle/manifests/hello-operator.clusterserviceversion.yaml

  installModes:
  - supported: true
    type: OwnNamespace
  - supported: true
    type: SingleNamespace
  - supported: true                  # Buna gerek olmayabilir
    type: MultiNamespace
  - supported: true                  # Buna gerek olmayabilir
    type: AllNamespaces

$ vim Makefile ==> increase version

$ make docker-build docker-push bundle-build bundle-push

(gerekirse "make bundle-build BUNDLE_IMG=quay.io/alpaslank/hello-operator-bundle..." da mumkun)

$ oc project openshift-marketplace  # Bu adim onemli yoksa operator install ekraninda namespace secemiyorsun

$ operator-sdk run bundle quay.io/alpaslank/hello-operator-bundle:v0.0.x ( versiyon ne ise)

#### Catalog yaratma

*** Bu islem oncesinde "hello-operator-catalog" reposunu quay.io'da yarat ve public yap!

$ make catalog-build catalog-push

# Otomatik update istiyorsak (dikkat; 

$ podman tag quay.io/alpaslank/hello-operator:0.1.3 quay.io/alpaslank/hello-operator:latest (bu adim sart degil ama paralellik acisindan yapmak iyi olur)
$ podman push quay.io/alpaslank/hello-operator:latest 

$ podman tag quay.io/alpaslank/hello-operator-bundle:v0.1.3 quay.io/alpaslank/hello-operator-bundle:latest
$ podman push quay.io/alpaslank/hello-operator-bundle:latest

$ podman tag quay.io/alpaslank/hello-operator-catalog:v0.1.3 quay.io/alpaslank/hello-operator-catalog:latest 
$ podman push quay.io/alpaslank/hello-operator-catalog:latest

$ oc edit 
...
spec:
  image: quay.io/alpaslank/hello-operator-catalog:latest
...
=======================================00
operator-sdk'yi pass gecerek quay.io'daki repo'lar uzerinden direkt operator kurmak icin sirasiyla "namespace, catalogsource, operatorgroup ve subscription" yarat (silmek icin de "sub, csv & og" u sil):

1) oc create namespace hello-operator

2) oc apply -f catalogsource.yaml

3) oc apply -f operatorgroup.yaml

4) oc apply -f subscription.yaml

Dosyalar:

$ cat catalogsource.yaml 
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: hello-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: Hello catalog
  icon:
    base64data: ""
    mediatype: ""
  image: quay.io/alpaslank/hello-operator-catalog:latest
  publisher: Perception
  sourceType: grpc
----------------------------------
$ cat operatorgroup.yaml 
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: hello-operator
  namespace: hello-operator
spec:
  targetNamespaces:
  - hello-operator
----------------------------------
$ cat subscription.yaml 
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hello-operator
  namespace: hello-operator
spec:
  channel: alpha
  name: hello-operator
  source: hello-operator-catalog
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
  #startingCSV: hello-operator.v0.0.2


