Continous delivery for microservices
------------------------

This is a step by step guide on how to set up a GitOps workflow with Istio/flagger and Weave Flux. GitOps is a way to do Continuous Delivery, it works by using Git as a source of truth for declarative infrastructure and workloads. In practice this means using git push instead of kubectl create/apply or helm install/upgrade.
 
Requirements
------------

  * A Kubernetes cluster (You can use managed Kubernetes cluster on GCP, AWS, Azure, OVH) 
  * helm, kubectl, and git binaries.
  * Fork this repo into your github account
  * Access to a public registry (Docker HUB, GCR, Quay)

Prepare the cluster
-----------------------

Commencons par installer les outils nécessaires pour utiliser notre cluster comme infrastruture de continous delivery. Pour suivre ce tutoriel vous devez avoir un cluster K8S fonctionnel et avoir configurer la connexion à celui-ci avec les droits admnistrateur via kubectl.

#### Helm install

Télécharger la dernière version de Helm pour votre système d'exploitation à l'URL suivante: https://github.com/helm/helm/releases

```bash
# Linux install

 wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
 tar zxvf helm-v2.13.1-linux-amd64.tar.gz
 sudo mv linux-amd64/helm /usr/local/bin/
 rm -rf linux-amd64
```

On va maintenant installer Tiller sur le cluster, il va falloir créer un compte pour que tiller puisse déployer nos applications sur un cluster avec RBAC activé.

```yaml
# cluster-admin-tiller.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

```bash
# Install tiller with cluster admin rights

kubectl apply -f cluster-admin-tiller.yaml
helm init --service-account tiller

# Check installation
helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```

#### Weave Flux

Maintenant queHelm est installé, nous allons pouvoir déployer les différentes application s dont nousa vons besoin, commencons par Flux  pour la partie GitOps.

N'oubliez pas de fork ce repo sur votre compte vous aurez besoin de l'URL de votre fork pendant l'installation de flux.

```bash
helm repo add weaveworks https://weaveworks.github.io/flux
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flux/master/deploy-helm/flux-helm-release-crd.yaml

# Install Weave Flux and its Helm Operator by specifying your fork URL (Just change your user)

helm upgrade -i flux \
--set helmOperator.create=true \
--set helmOperator.createCRD=false \
--set git.url=git@github.com:jabberwock888/cd-containers.git \
--set git.path=flux/ \
--namespace flux \
weaveworks/flux
```

Retrieve the public key and add it to your forked repo. If you don't know how to do it, you can follow this documentation https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys

> Do not forget to add write access when you add the public key in github

```bash
# Copy the result of the command in the deploy keys section of your forked repo
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDW+goQDzlTdo...
```

#### Istio

Maintenant que nous avons installés Flux, nous allons pouvoir l'utiliser pour déployer les applications manquantes en mode GitOps. Dans ce repo, il y a déjà la configuration nécessaire pour installer Istio via Flux. 

Il faut juste retirer l'annotation ignore, pour que Flux déploie automatiquement Istio. Commencons par installer le chart Istio-init en charge des CRDs de Istio.

```yaml
# Edit the ini-istio-1.1.x.yaml and change the weave.work/ignore annotation to false

---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: init-istio
  namespace: istio-system
  annotations:
    flux.weave.works/ignore: 'false'
spec:
  releaseName: init-istio
  chart:
    repository: https://storage.googleapis.com/istio-release/releases/1.1.0/charts/
    name: istio-init
    version: 1.1.0
  values:
    global:
      hub: gcr.io/istio-release
      tag: master-latest-daily
      imagePullPolicy: IfNotPresent
    certmanager:
      enabled: false
```

> Do not forget to push your changes !

```bash
kubectl logs -f deployment/flux -n flux
Event(v1.ObjectReference{Kind:"HelmRelease", Namespace:"istio-system", Name:"init-istio", UID:"b620d559-6998-11e9-8381-8e7b409193c5", APIVersion:"flux.weave.works/v1beta1", ResourceVersion:"6823", FieldPath:""}): type: 'Normal' reason: 'ChartSynced' Chart managed by HelmRelease processed successfully

#Istio-init chart should be installed now
helm ls | grep istio-init
init-istio      1       DEPLOYED        istio-init-1.1.0        1.1.0           istio-system
```

Now follow the same process for the "istio-1.1.x.yaml file", and finally for "gw-istio-system.yaml" file. Vous devez respecter cette odre car les ressources sont dépendantes les unes des autres. Vérifions que tout soit bien installé avant de continuer.

```bash
# You should see these 3 Helm charts
helm ls
NAME            REVISION             STATUS          CHART                   APP VERSION     NAMESPACE
flux            3                    DEPLOYED        flux-0.9.1              1.12.0          flux
init-istio      1                    DEPLOYED        istio-init-1.1.0        1.1.0           istio-system
istio           1                    DEPLOYED        istio-1.1.0             1.1.0           istio-system

# The Istio GW ("gw-istio-system.yaml" file)
kubectl get gateway -n istio-system
NAME       AGE
cloud-gw   7m

# It shouldn't have any pods in pending otherwise it is that your nodes don't have enough CPU to deploy Istio
kubectl get po -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
istio-citadel-645ffc4999-btqmf            1/1     Running     0          12m
istio-galley-978f9447f-q55f4              1/1     Running     0          12m
istio-ingressgateway-8ccdc79bc-gk2rd      0/1     Running     0          12m
istio-init-crd-10-zjwcp                   0/1     Completed   0          20m
istio-init-crd-11-lpx69                   0/1     Completed   0          20m
istio-pilot-5d5dc955dd-rdc6z              0/2     Running     0          12m
istio-policy-c9469c4df-g8x88              2/2     Running     1          12m
istio-sidecar-injector-6dcc9d5c64-wsnc6   1/1     Running     0          12m
istio-telemetry-55d5b7d4dc-z4hwc          0/2     Running     0          12m
prometheus-66c9f5694-tvstz                1/1     Running     0          12m
```

#### Flagger

Dernière application Flagger pour l'automatisation du déploiement canary avec Istio. Nous allons utilser flux pour le déployer comme nous avons fait avec Istio.

Même procédé éditez simplement le fichier flagger-0.9.0.yaml et changer l'annotation flux.weave.works/ignore en false.

```yaml
---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: flagger
  namespace: "istio-system"
  annotations:
    flux.weave.works/ignore: 'false'
spec:
  releaseName: flagger
  chart:
    repository: https://flagger.app/
    name: flagger
    version: 0.9.0
  values:
    namespace: "myapp"
```

```bash
# Verify the installation with helm
helm ls | grep flagger
flagger         1                 DEPLOYED        flagger-0.9.0           0.9.0           istio-system
```

Build and deploy our app
-------------------------

Maintenant nous allons pouvoir déployer notre application sur le cluster, et profiter des focntions de déploiement canary, ainsi que le déploiement via Gitops.

D'abord on va build notre application via un container docker, et publier sa première version sur la registry docker hub. Pour la simplicité de l'exemple nous allons simplement build une image nginx avec un index.html modifié

```bash
cd demo-gitops/app
# Change REGISTRY/REPOSITORY with the correct values for your registry.
docker build -t REGISTRY/REPOSITORY/myapp:1.0.0 .
docker push REGISTRY/REPOSITORY/myapp:1.0.0
```

Ensuite editez le fichier de déploiement flux pour votre application.

```yaml
---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: myapp
  namespace: myapp
  annotations:
    # change flux.weave.works/ignore to false
    flux.weave.works/ignore: 'false'
    flux.weave.works/automated: "true"
    flux.weave.works/tag.chart-image: semver:~1.0
spec:
  releaseName: myapp
  chart:
    git: git@github.com:skhedim/demo-gitops.git
    path: charts/myapp
    ref: master
  values:
    image:
      # change with your correct image path
      repository: REGISTRY/REPOSITORY/myapp
      tag: 1.0.0
    canary:
      enabled: true
      istioIngress:
        enabled: true
        gateway: cloud-gw.istio-system.svc.cluster.local
        # Change the MY_SVC_IPwith the result of following command:
        # kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
        host: MY_SVC_IP.xip.io
```

Push your changes, et vous devriez être capable d'accéder à votre application en utilisant l'url http://MY_SVC_IP.xip.io.

Canary deployment with Istio/Flagger
------------------------------------

Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA) and creates a series of objects (Kubernetes deployments, ClusterIP services and Istio virtual services) to drive the canary analysis and promotion.

### Canary Custom Resource

For a deployment named _myapp_, a canary promotion can be defined using Flagger's custom resource:

```yaml
apiVersion: flagger.app/v1alpha3
kind: Canary
metadata:
  name: myapp
  namespace: test
spec:
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  # the maximum time in seconds for the canary deployment
  # to make progress before it is rollback (default 600s)
  progressDeadlineSeconds: 60
  # HPA reference (optional)
  autoscalerRef:
    apiVersion: autoscaling/v2beta1
    kind: HorizontalPodAutoscaler
    name: myapp
  service:
    # container port
    port: 9898
    # service port name (optional, will default to "http")
    portName: http-myapp
    # Istio gateways (optional)
    gateways:
    - cloud-gw.istio-system.svc.cluster.local
    - mesh
    # Istio virtual service host names (optional)
    hosts:
    - myapp.example.com
  # promote the canary without analysing it (default false)
  skipAnalysis: false
  # define the canary analysis timing and KPIs
  canaryAnalysis:
    # schedule interval (default 60s)
    interval: 1m
    # max number of failed metric checks before rollback
    threshold: 10
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    # Istio Prometheus checks
    metrics:
    - name: istio_requests_total
      # minimum req success rate (non 5xx responses)
      # percentage (0-100)
      threshold: 99
      interval: 1m
    - name: istio_request_duration_seconds_bucket
      # maximum req duration P99
      # milliseconds
      threshold: 500
      interval: 30s
    # external checks (optional)
    webhooks:
      - name: integration-tests
        url: http://myapp.test:9898/echo
        timeout: 1m
        # key-value pairs (optional)
        metadata:
          test: "all"
          token: "16688eb5e9f289f1991c"
```

### Istio Gateways and Virtual services

#### Gateways

A Gateway configures a load balancer for HTTP/TCP traffic operating at the edge of the mesh, most commonly to enable ingress traffic for an application.

Unlike Kubernetes Ingress, Istio Gateway only configures the L4-L6 functions (for example, ports to expose, TLS configuration). Users can then use standard Istio rules to control HTTP requests as well as TCP traffic entering a Gateway by binding a VirtualService to it.

#### Virtual services

A VirtualService defines the rules that control how requests for a service are routed within an Istio service mesh. For example, a virtual service could route requests to different versions of a service or to a completely different service than was requested. Requests can be routed based on the request source and destination, HTTP paths and header fields, and weights associated with individual service versions.

### Check the configuration

The istio gateway was deployed by flux via the istio-gw.yaml file of the repo. The Virtual service part will be automatically created by Flagger. We will now have to deploy our application and create the Canary file to Flagger

```bash
# Check the Istio GW configuration
kubectl get gateway -n istio-system  -o yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cloud-gw
  namespace: istio-system
  annotations:
    flux.weave.works/ignore: 'false'
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

# Check if you can see the canary configuration
kubectl get canary -n myapp
NAME    STATUS        WEIGHT   LASTTRANSITIONTIME
myapp   Initialized   0        2019-04-01T08:33:42Z
```

Deploy a new version
--------------------

Now everything is in place to test a canary deployment. Flagger monitors the myapp deployment,and waits for the image to be changed to create a second deployment and a virtual service.

In parallel Flux monitors the registry and waits for a new tag to be pushed into the registry to update the deployment. Modify the index.html file and push the new image to the registry

```html
<!--Change the version H2 title-->
<!DOCTYPE html>
<html>
<body>

<h2>Welcome to myapp v1.0.1</h2>

</body>
</html>
```

```bash
# Change REGISTRY/REPOSITORY with the correct values for your registry.
docker build -t REGISTRY/REPOSITORY/myapp:1.0.1 .
docker push REGISTRY/REPOSITORY/myapp:1.0.1
```

# Wait for flux to detect the new image in the registry
kubectl -n flux logs -f $(kubectl -n flux get pods -l app=flux -o jsonpath='{.items[0].metadata.name}')
```

