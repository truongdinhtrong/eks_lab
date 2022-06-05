
## EKS - CICD - Security - Quota - Istio:

  - [01. Install EKS:](#01-install-eks)
    - [a. EKS with on-demand instance:](#a-eks-with-on-demand-instance)
    - [b. EKs with spot instance:](#b-eks-with-spot-instance)
    - [c. Scale out or in a single nodegroup:](#c-scale-out-or-in-a-single-nodegroup)
    - [d. EKS with On-demand and Spot instance:](#d-eks-with-on-demand-and-spot-instance)
    - [e. EKS with AWS Graviton2 processors:](#e-eks-with-aws-graviton2-processors)
  - [02. Observability:](#02-observability)
    - [a. eBPF - Pixie:](#a-ebpf-pixie)
    - [b. Deploy Grafana-Prometheus with helm:](#b-deploy-grafana-prometheus-with-helm)
  - [03. Persistence volume:](#03-persistence-volume)
    - [a. Amazon EBS CIS driver:](#a-amazon-ebs-cis-driver)
    - [b. Amazon EFS CIS driver:](#b-amazon-efs-cis-driver)
    - [c. Backup and restore data:](#c-backup-and-restore-data)
  - [04. Install Application: Wordpress](#04-install-application-wordpress)
    - [a. install MariaDB:](#a-install-mariadb)
    - [b. install Wordpress:](#b-install-wordpress)
    - [c. install Memcached:](#c-install-memcached)[](vscode-webview://1i8i6vpk77s4a3uvfcgtr7rhivira5r57u99b31hphn3ko44qd1b/Users/macbook/DATA/workspace/devops_lab/eks/README.md#a-ebpf-pixie)
  - [05. Install cert-manager:](#05-install-cert-manager)
    - [a. install cert-manager:](#a-install-cert-manager)
    - [b. install Cluster Issuer - Let’s Encrypt:](#b-install-cluster-issuer-lets-encrypt)
  - [06. Install Ingress and SSL:](#06-install-ingress-and-ssl)
    - [a. install ingress controller:](#a-install-ingress-controller)
    - [b. install ingress and configure SSL Let's Encrypt for vhost:](#b-install-ingress-and-configure-ssl-lets-encrypt-for-vhost)
    - [c. configure public certificate:](#c-configure-public-certificate)
  - [07. Install CI/CD:](#07-install-cicd)
    - [a. CI - install jenkins master-slave:](#a-ci-install-jenkins-master-slave)
    - [b. CD - install ArgoCD:](#b-cd-install-argocd)
  - [08. Security k8s:](#08-security-k8s)
    - [a. RBAC:](#a-rbac)
    - [b. Network policy:](#b-network-policy)
    - [c. Limit requests from LoadBalancer:](#c-limit-requests-from-loadbalancer)
  - [09. Quota resource:](#09-quota-resource)
  - [10. Service Mesh - Istio:](#10-service-mesh-istio)
    - [a. Install Istio:](#a-install-istio)
    - [b. Configure istio-gateway:](#b-configure-istio-gateway)

  

---
#### 01. install EKS:

- install eksctl on MacOS:

```
brew tap weaveworks/tap

brew install weaveworks/tap/eksctl
```

##### a. EKS with on-demand instance:

- install EKS:

```
eksctl create cluster \
  --name cluster-test \
  --version 1.21 \
  --region us-east-1 \
  --nodegroup-name worker-nodes \
  --node-type t2.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 10 \
  --ssh-access --ssh-public-key=~/.ssh/trong_online_new.pub
```

Note: dùng t2.medium cho lab vì cài Grafana-Prometheus khá nặng.


- delete EKS cluster:
```
eksctl delete cluster --region=us-east-1 --name=cluster-test --wait
```

##### b. EKs with spot instance:

```
eksctl create cluster \
  --name cluster-test \
  --spot \
  --instance-types t2.medium \
  --version 1.21 \
  --region us-east-1 \
  --nodegroup-name workers-spot \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 10 \
  --asg-access \
  --ssh-access --ssh-public-key=~/.ssh/trong_online_new.pub
```

- delete EKS cluster:
```
eksctl delete cluster --region=us-east-1 --name=cluster-test --wait
```

##### c. Scale out or in a single nodegroup:

- Scale out a single nodegroup:
```
eksctl scale nodegroup \
  --cluster=cluster-test \
  --nodes=4 workers-on-demand \
  --region us-east-1
```

- Scale in a single nodegroup:
```
eksctl scale nodegroup \
  --cluster=cluster-test \
  --nodes=2 workers-on-demand \
  --region us-east-1
```

```
kubectl get nodes

NAME                             STATUS                     ROLES    AGE   VERSION
ip-192-168-33-231.ec2.internal   Ready                      <none>   20m   v1.21.12-eks-5308cf7
ip-192-168-50-1.ec2.internal     Ready,SchedulingDisabled   <none>   59m   v1.21.12-eks-5308cf7
ip-192-168-9-161.ec2.internal    Ready                      <none>   20m   v1.21.12-eks-5308cf7
ip-192-168-9-17.ec2.internal     Ready,SchedulingDisabled   <none>   60m   v1.21.12-eks-5308cf7
```

Note: đợi 1 khoản thời gian thì spot instance mới tự delete và đồng bộ lại các POD trong cluster.

##### d. EKS with On-demand and Spot instance:

- Create cluster with On-demand and Spot instance:

B1> create cluster with on-demand instance:

```
eksctl create cluster \
  --name cluster-test \
  --version 1.21 \
  --region us-east-1 \
  --nodegroup-name workers-one-demand \
  --node-type t2.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 10 \
  --asg-access \
  --ssh-access --ssh-public-key=~/.ssh/trong_online_new.pub
```

B2> create nodegroup with spot instance:

```
eksctl create nodegroup \
  --spot \
  --name workers-spot \
  --cluster cluster-test \
  --region us-east-1 \
  --managed \
  --instance-types=t2.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 10 \
  --asg-access \
  --ssh-access --ssh-public-key=~/.ssh/trong_online_new.pub
```

```
kubectl get nodes \
  --label-columns=eks.amazonaws.com/capacityType \
  --selector=eks.amazonaws.com/capacityType=ON_DEMAND
NAME                            STATUS   ROLES    AGE     VERSION                CAPACITYTYPE
ip-192-168-0-9.ec2.internal     Ready    <none>   2m57s   v1.21.12-eks-5308cf7   ON_DEMAND
ip-192-168-62-99.ec2.internal   Ready    <none>   2m59s   v1.21.12-eks-5308cf7   ON_DEMAND
```

```
kubectl get nodes \
  --label-columns=eks.amazonaws.com/capacityType \
  --selector=eks.amazonaws.com/capacityType=SPOT

NAME                             STATUS   ROLES    AGE   VERSION                CAPACITYTYPE
ip-192-168-27-207.ec2.internal   Ready    <none>   20m   v1.21.12-eks-5308cf7   SPOT
ip-192-168-41-92.ec2.internal    Ready    <none>   20m   v1.21.12-eks-5308cf7   SPOT
```
```
kubectl get nodes --sort-by=.metadata.creationTimestamp

NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-62-99.ec2.internal    Ready    <none>   26m   v1.21.12-eks-5308cf7
ip-192-168-0-9.ec2.internal      Ready    <none>   26m   v1.21.12-eks-5308cf7
ip-192-168-27-207.ec2.internal   Ready    <none>   19m   v1.21.12-eks-5308cf7
ip-192-168-41-92.ec2.internal    Ready    <none>   19m   v1.21.12-eks-5308cf7
```

- scale out spot instance:

```
eksctl scale nodegroup \
  --name workers-spot \
  --cluster cluster-test \
  --nodes 4 \
  --region us-east-1 
```

```
kubectl get nodes \
  --label-columns=eks.amazonaws.com/capacityType \
  --selector=eks.amazonaws.com/capacityType=SPOT

NAME                             STATUS   ROLES    AGE   VERSION                CAPACITYTYPE
ip-192-168-27-207.ec2.internal   Ready    <none>   67m   v1.21.12-eks-5308cf7   SPOT
ip-192-168-30-130.ec2.internal   Ready    <none>   78s   v1.21.12-eks-5308cf7   SPOT
ip-192-168-32-56.ec2.internal    Ready    <none>   98s   v1.21.12-eks-5308cf7   SPOT
ip-192-168-41-92.ec2.internal    Ready    <none>   67m   v1.21.12-eks-5308cf7   SPOT
```

- scale in sport instance:
```
eksctl scale nodegroup \
  --name workers-spot \
  --cluster cluster-test \
  --nodes 2 \
  --region us-east-1 
```

- delete nodegroup:

```
eksctl delete nodegroup \
  --cluster cluster-test \
  --region us-east-1 \
  --name workers-on-demand
```
```
kubectl get nodes

NAME                             STATUS                     ROLES    AGE    VERSION
ip-192-168-0-9.ec2.internal      Ready,SchedulingDisabled   <none>   110m   v1.21.12-eks-5308cf7
ip-192-168-30-130.ec2.internal   Ready                      <none>   37m    v1.21.12-eks-5308cf7
ip-192-168-32-56.ec2.internal    Ready                      <none>   37m    v1.21.12-eks-5308cf7
ip-192-168-62-99.ec2.internal    Ready,SchedulingDisabled   <none>   110m   v1.21.12-eks-5308cf7
```

- delete cluster:
```
eksctl delete cluster --region=us-east-1 --name=cluster-test --wait
```

##### e. EKS with AWS Graviton2 processors:

```
https://florent-brosse.medium.com/how-to-optimize-cost-with-aws-graviton-and-spot-in-amazon-elastic-kubernetes-service-eks-85287d736f48
```

```
eksctl create cluster \
  --version 1.21 \
  --name cluster-test \
  --instance-types t2.micro \
  --managed \
  --nodes-max 10 \
  --nodes-min 1 \
  --nodes 1 \
  --asg-access \
  --nodegroup-name workers-on-demand \
  --region=us-east-1
```

```
eksctl create nodegroup \
  --cluster cluster-test \
  --instance-types m6g.medium \
  --managed \
  --spot \
  --name workers-spot \
  --asg-access \
  --nodes-max 10 \
  --nodes-min 1 \
  --nodes 1 \
  --region=us-east-1
```


---
#### 02. Observability:

##### a. eBPF - Pixie:

- install Pixie CLI on macos terminal:

```
bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"
```

- deploy Pixie on K8s:
```
px deploy

Pixie CLI
Running Cluster Checks:
 ✔    Kernel version > 4.14.0 
 ✔    Cluster type is supported 
 ✔    K8s version > 1.16.0 
 ✔    Kubectl > 1.10.0 is present 
 ✔    User can create namespace 
 ✔    Cluster type is in list of known supported types 
Installing Vizier version: 0.11.2
Generating YAMLs for Pixie
Deploying Pixie to the following cluster: admin@cluster-test.us-east-1.eksctl.io
Is the cluster correct? (y/n) [y] : y
Found 2 nodes
 ✔    Installing OLM CRDs 
 ✔    Deploying OLM 
 ✔    Deploying Pixie OLM Namespace 
 ✔    Installing Vizier CRD 
 ✔    Deploying OLM Catalog 
 ✔    Deploying OLM Subscription 
 ✔    Creating namespace 
 ✔    Deploying Vizier 
 ✔    Waiting for Cloud Connector to come online 
Waiting for Pixie to pass healthcheck
 ✔    Wait for PEMs/Kelvin 
 ✔    Wait for healthcheck 

==> Next Steps:

Run some scripts using the px cli. For example: 
- px script list : to show pre-installed scripts.
- px run px/service_stats : to run service info for sock-shop demo application (service selection coming soon!).

Check out our docs: https://docs.withpixie.ai:443.

Visit : https://work.withpixie.ai:443 to use Pixie's UI.
```

##### b. Deploy Grafana-Prometheus with helm:

- create monitoring namespace:

```
k create ns monitoring
```

- get file values:
```
helm show values prometheus-community/kube-prometheus-stack >  grafana-prometheus/kube-prometheus-stack.yaml
```

- install Grafana Prometheus:

```
helm install monitor-prod prometheus-community/kube-prometheus-stack -f grafana-prometheus/kube-prometheus-stack.yaml -n monitoring
```

- edit svc dùng ELB:

đổi ClusterIP thành LoadBalancer:

```
k edit svc gra-pro-kube-prometheus-st-prometheus
```
```
k edit svc gra-pro-grafana    
```

```
k get svc                 
NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                      AGE
alertmanager-operated                     ClusterIP      None             <none>                                                                         9093/TCP,9094/TCP,9094/UDP   3m59s
gra-pro-grafana                           LoadBalancer   10.100.62.187    ac93357813d00405f8410bc91bc5c8d1-935614249.ap-southeast-1.elb.amazonaws.com    80:31070/TCP                 4m11s
gra-pro-kube-prometheus-st-alertmanager   ClusterIP      10.100.182.252   <none>                                                                         9093/TCP                     4m11s
gra-pro-kube-prometheus-st-operator       ClusterIP      10.100.56.156    <none>                                                                         443/TCP                      4m11s
gra-pro-kube-prometheus-st-prometheus     LoadBalancer   10.100.134.156   a3a98a7dc61354cd68c17def666ae9a7-1255035348.ap-southeast-1.elb.amazonaws.com   9090:30707/TCP               4m11s
gra-pro-kube-state-metrics                ClusterIP      10.100.240.67    <none>                                                                         8080/TCP                     4m11s
gra-pro-prometheus-node-exporter          ClusterIP      10.100.172.96    <none>                                                                         9100/TCP                     4m11s
kubernetes                                ClusterIP      10.100.0.1       <none>                                                                         443/TCP                      19m
prometheus-operated                       ClusterIP      None             <none>                                                                         9090/TCP                     3m59s
```

- password Grafana:
```
admin
prom-operator
```

- uninstall Grafana-Prometheus:
```
helm list --all-namespaces

NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
monitor-prod    monitoring      1               2022-05-23 14:12:38.110992 +0700 +07    deployed        kube-prometheus-stack-35.2.0    0.56.2     
```

```
helm uninstall monitor-prod -n monitoring
```

---
#### 03. Persistence volume:

##### a. Amazon EBS CIS driver:

```
https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
```

- tạo IAM OIDC:
```
eksctl utils associate-iam-oidc-provider --cluster cluster-test --region us-east-1 --approve

2022-05-23 17:43:15 [ℹ]  eksctl version 0.98.0
2022-05-23 17:43:15 [ℹ]  using region ap-southeast-1
2022-05-23 17:43:16 [ℹ]  will create IAM Open ID Connect provider for cluster "cluster-test" in "ap-southeast-1"
2022-05-23 17:43:17 [✔]  created IAM Open ID Connect provider for cluster "cluster-test" in "ap-southeast-1"
```

- show kết quả tạo IAM OIDC:
```
aws eks describe-cluster --name cluster-test --query "cluster.identity.oidc.issuer" --output text
```

```
aws iam list-open-id-connect-providers | grep AEFE4D85BF2C37882AB57EA042994EXP

            "Arn": "arn:aws:iam::302380193111:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/AEFE4D85BF2C37882AB57EA042994EXP""
```

- create Amazon EBS CSI driver IAM role for service accounts:
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster cluster-test \
  --region us-east-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

- Adding the Amazon EBS CSI add-on:

dùng để tăng security và giảm deplay mount của EBS.

```
eksctl create addon --name aws-ebs-csi-driver --cluster cluster-test --region us-east-1 --service-account-role-arn arn:aws:iam::302380193245:role/AmazonEKS_EBS_CSI_DriverRole --force
```

- test thử EBS xem có tạo đc pv,pvc không:

làm theo hướng dẫn demo để xem có attach được ebs vào cluster không:
```
https://docs.aws.amazon.com/eks/latest/userguide/ebs-sample-app.html
```


- tạo pv,pvc cho MariaDB từ EBS:

vi storage/pv-pvc-mariadb-ebs-storage-class.yaml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: pv-mariadb-ebs
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp3
  # encrypted: 'true' # EBS volumes will always be encrypted by default
volumeBindingMode: WaitForFirstConsumer # EBS volumes are AZ specific
reclaimPolicy: Delete
mountOptions:
- debug

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-mariadb-ebs
spec:
  storageClassName: pv-mariadb-ebs
  accessModes:
    # - ReadWriteMany
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

- tạo pv,pvc cho Wordpress từ EBS:

vi storage/pv-pvc-wordpress-ebs-storage-class.yaml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: pv-wordpress-ebs
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp3
  # encrypted: 'true' # EBS volumes will always be encrypted by default
volumeBindingMode: WaitForFirstConsumer # EBS volumes are AZ specific
reclaimPolicy: Delete
mountOptions:
- debug

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-wordpress-ebs
spec:
  storageClassName: pv-wordpress-ebs
  accessModes:
    # - ReadWriteMany
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

```
k apply -f storage/
```

##### b. Amazon EFS CIS driver:

```
https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
```


##### c. Backup and restore data:

---
#### 04. Install Application: Wordpress

##### a. install MariaDB:


```
helm show values bitnami/mariadb > mariadb-values.yaml 
```
thay đổi các tham số: 

```
...
architecture: standalone
auth:
  rootPassword: "root@456"
  database: wordpress
  username: "wordpress_admin"
  password: "wordpress_admin"
...
  persistence:
    enabled: true
    existingClaim: "pvc-mariadb-ebs"
...
  service:
    type: ClusterIP
```
Note: PVC cài ở namespace nào thì install mariadb ở namespace đó nếu không sẽ không claim được volume.

- install mariadb with helm:

```
helm install mariadb-prod bitnami/mariadb -f mariadb/mariadb-values.yaml 
```


- Note:

```
** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace database -l app.kubernetes.io/instance=mariadb-prod

Services:

  echo Primary: mariadb-prod.database.svc.cluster.local:3306

Administrator credentials:

  Username: root
  Password : $(kubectl get secret --namespace database mariadb-prod -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mariadb-prod-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.5.15-debian-10-r52 --namespace database --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mariadb-prod.database.svc.cluster.local -uroot -p wordpress

To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'auth.rootPassword' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace database mariadb-prod -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)
      helm upgrade --namespace database mariadb-prod bitnami/mariadb --set auth.rootPassword=$ROOT_PASSWORD
```

##### b. install Wordpress:

```
helm show values bitnami/wordpress > wordpress/wordpress-values.yaml
```

thay các tham số như: user password, connect service mariadb, ...

```
...
existingClaim: "pvc-wordpress-ebs"
...
externalDatabase:
  host: mariadb-prod
  port: 3306
  user: wordpress_admin
  password: "wordpress_admin"
  database: wordpress
...
service:
  type: ClusterIP
...
```

- install wordpress with helm:
```
helm install wordpress-prod bitnami/wordpress -f wordpress/wordpress-values.yaml
```

- Note:

```
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    wordpress-prod.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

   kubectl port-forward --namespace default svc/wordpress-prod 80:80 &
   echo "WordPress URL: http://127.0.0.1//"
   echo "WordPress Admin URL: http://127.0.0.1//admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: admin
  echo Password: $(kubectl get secret --namespace default wordpress-prod -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

- public wordpress with service type LoadBalancer:

vi wordpress/wordpress-values.yaml
```
service:
  type: LoadBalancer
```
```
helm upgrade wordpress-prod bitnami/wordpress -f wordpress/wordpress-values.yaml
```

- check service k8s:
```
k get svc               
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP                                                                   PORT(S)                      AGE
kubernetes       ClusterIP      10.100.0.1     <none>                                                                        443/TCP                      5h12m
mariadb-prod     NodePort       10.100.73.32   <none>                                                                        3306:30725/TCP               17m
wordpress-prod   LoadBalancer   10.100.86.99   a0caee15fe6204f6e8148c098eabf085-173044325.ap-southeast-1.elb.amazonaws.com   80:30730/TCP,443:32399/TCP   17m
```


##### c. install Memcached:


---
#### 05. Install cert-manager:

##### a. install cert-manager:

```
wget https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml

kubectl apply -f cert-manager/cert-manager.yaml
```

##### b. install Cluster Issuer - Let’s Encrypt:

vi cert-manager/cluster-issuer.yaml

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer # ClusterIssuer work với all namespace, Issuer work với 1 namespace
metadata:
  # name: letsencrypt-staging # server test
  name: letsencrypt-prod

spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    # email: user@example.com 
    email: truongdinhtrongctim@gmail.com
    # server: https://acme-staging-v02.api.letsencrypt.org/directory
    server: https://acme-v02.api.letsencrypt.org/directory # chạy production
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      # name: example-issuer-account-key
      name: letsencrypt-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```
```
kubectl apply -f cert-manager/cluster-issuer.yaml 
```
---
#### 06. Install Ingress and SSL:

##### a. install ingress controller:

```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/cloud/deploy.yaml -O ingress/ingress-controller.yaml
```

```
kubectl apply -f ingress/ingress-controller.yaml
```

##### b. install ingress and configure SSL Let's Encrypt for vhost:

- install ingress vhost:

vi ingress/ingress-vhost-wordpress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-wordpress-truongdinhtrong-com
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      location /admin {
        allow 172.16.94.1;
        deny all;
      }
spec:
  tls:
  - hosts:
    - wordpress.truongdinhtrong.com
    secretName: wordpress-truongdinhtrong-com-tls
  rules:
  - host: "wordpress.truongdinhtrong.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: wordpress-prod
            port:
              number: 80

```

```
k apply -f ingress/ingress-vhost-wordpress.yaml 
```
- Note: service của wordpress phải dùng ClusterIP (không dùng LoadBalancer) để POD acme connect qua chứng thực HTTP01 bằng link /.well-known/acme-challenge/

- create CNAME wordpress.truongdinhtrong.com trỏ về LoadBalancer của AWS: 

```
 k get svc -n ingress-nginx       
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.210.23    a1cbe4987b9584c85b2af55541a94e68-1431810916.ap-southeast-1.elb.amazonaws.com   80:32322/TCP,443:30091/TCP   23m
ingress-nginx-controller-admission   ClusterIP      10.100.219.204   <none>                                                                         443/TCP                      23m
```

Note: tạo CNAME wordpress.truongdinhtrong.com trỏ về a1cbe4987b9584c85b2af55541a94e68-1431810916.ap-southeast-1.elb.amazonaws.com

##### c. Using public certificate:

- tạo secret chứa cert và key:
```
kubectl create secret tls star-truongdinhtrong-com-tls --namespace default --key comodo_ssl/truongdinhtrong_com.key --cert comodo_ssl/truongdinhtrong_com.crt
```

- add secret đó vào ingress:

vi ingress-prod.yaml
```

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-star-truongdinhtrong-com
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  tls:
  - hosts:
    - *.truongdinhtrong.com
    secretName: star-truongdinhtrong-com-tls
  rules:
  - host: "*.truongdinhtrong.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: wordpress-prod
            port:
              number: 80
```


---
#### 07. Install CI/CD:

##### a. CI - install jenkins master-slave:

- build docker:

vi jenkins/Dockerfile
```
FROM jenkins/jenkins:latest-jdk8
USER root
RUN apt-get update && apt-get install -y lsb-release curl net-tools wget unzip ansible awscli software-properties-common
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add -
RUN apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
 signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
 https://download.docker.com/linux/debian \
 $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
# install docker
RUN apt-get update && apt-get install -y docker-ce-cli terraform
#ensures that /var/run/docker.sock exists
RUN touch /var/run/docker.sock && chown root:jenkins /var/run/docker.sock 
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.2 docker-workflow:1.26"
```

```
cd jenkins
docker build -t jenkins_cicd:1.2 -f Dockerfile .
```

```
docker login
```

```
docker push truongdinhtrongctim/jenkins_cicd:1.2
```

- deploy jenkins master lên k8s:

```
k create ns ci-cd
```

vi storage/pv-pvc-jenkins-ebs-storage-class.yaml
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: pv-data-jenkins-master-ebs
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp3
  # encrypted: 'true' # EBS volumes will always be encrypted by default
volumeBindingMode: WaitForFirstConsumer # EBS volumes are AZ specific
reclaimPolicy: Delete
mountOptions:
- debug

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data-jenkins-master-ebs
  namespace: ci-cd
spec:
  storageClassName: pv-data-jenkins-master-ebs
  accessModes:
    # - ReadWriteMany
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

```
k apply -f storage/pv-pvc-jenkins-ebs-storage-class.yaml
```

vi jenkins/deploy_jenkins-master.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-jenkins-master
  namespace: ci-cd
spec:
  selector:
    matchLabels:
      app: jenkins-master
  template:
    metadata:
      labels:
        app: jenkins-master
    spec:
      volumes:
      - name: volume-jenkins-master
        persistentVolumeClaim:
          claimName: pvc-data-jenkins-master-ebs
      containers:
      - name: jenkins-master
        image: truongdinhtrongctim/jenkins_cicd:1.2
        ports:
        - containerPort: 8080
        volumeMounts:
          - mountPath: "/var/jenkins_home"
            name: volume-jenkins-master
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-master
  namespace: ci-cd
spec:
  selector:
    app: jenkins-master
  ports:
  - port: 8282
    targetPort: 8080
```


- install jenkins with helm:

```
helm show values bitnami/jenkins > jenkins/jenkins-values.yaml
```

```
...
jenkinsUser: admin
jenkinsPassword: "f70f6cb2937348a89cd71e0e76d2dec5"

  existingClaim: "pvc-data-jenkins-master-ebs"
service:
  type: ClusterIP
...
```

```
helm install jenkins bitnami/jenkins -f jenkins/jenkins-values.yaml -n ci-cd
```

```
** Please be patient while the chart is being deployed **

1. Get the Jenkins URL by running:

  echo "Jenkins URL: http://127.0.0.1:8080/"
  kubectl port-forward --namespace ci-cd svc/jenkins 8080:80

2. Login with the following credentials

  echo Username: admin
  echo Password: $(kubectl get secret --namespace ci-cd jenkins -o jsonpath="{.data.jenkins-password}" | base64 --decode)              
```

##### b. CD - install ArgoCD:


```
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -O argo-cd/install-argo-cd.yaml
```

```
k apply -f argo-cd/install-argo-cd.yaml -n ci-cd
```
```
kubectl -n ci-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

```
kubectl port-forward svc/argocd-server -n ci-cd 8081:443
```


- test argoCD:
```
brew install argocd
argocd login localhost:8081
argocd app create helm-guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path helm-guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app get helm-guestbook
argocd app sync helm-guestbook
argocd app delete helm-guestbook
```

- error:

```
failed to sync cluster https://10.100.0.1:443: failed to load initial state of resource Endpoints: endpoints is forbidden: User "system:serviceaccount:ci-cd:argocd-application-controller" cannot list resource "endpoints" in API group "" at the cluster scope
```
- fixed:
```
```


---
#### 08. Security k8s:

##### a. RBAC:

##### b. Network policy:

##### c. Limit requests from LoadBalancer:

```
https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting
```


---
#### 09. Quota Resource:


#### 10. Service Mesh - Istio:

##### a. Install Istio:

- install Istio:

```
wget https://github.com/istio/istio/releases/download/1.13.4/istio-1.13.4-osx.tar.gz
```

```
tar -zxvf istio-1.13.4-osx.tar.gz
mkdir istio-installation
mv istio-1.13.4 istio-installation
```

```
export PATH=$PATH:/Users/macbook/Downloads/istio-installation/istio-1.13.4/bin
```

```
istioctl install

This will install the Istio 1.13.4 default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed                                                                                   
✔ Istiod installed                                                                                       
✔ Ingress gateways installed                                                                             
✔ Installation complete   

Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/pzWZpAvMVBecaQ9h9
```



- inject proxy to POD:

```
kubectl label namespace default istio-injection=enabled
```
```
kubectl get ns default --show-labels                   
NAME      STATUS   AGE   LABELS
default   Active   67m   istio-injection=enabled,kubernetes.io/metadata.name=default
```

- refresh POD's mariadb,wordpress to apply Istio injection :

```
k delete pod mariadb-prod-0 -n default
k delete pod wordpress-prod-8464f44667-knkds -n default
```

```
kubectl get pods

NAME                              READY   STATUS    RESTARTS   AGE
mariadb-prod-0                    2/2     Running   0          32m
wordpress-prod-8464f44667-knkds   2/2     Running   0          18m
```

Note: các pod lúc này là 2/2 container trong mỗi POD vì mỗi POD sẽ inject vào 1 init container của Istio để monitor POD đó.

- check label:

```
kubectl get namespace -L istio-injection

NAME              STATUS   AGE     ISTIO-INJECTION
cert-manager      Active   3h7m    enabled
ci-cd             Active   3h17m   enabled
default           Active   3h43m   enabled
ingress-nginx     Active   3h5m    
istio-system      Active   156m    
kube-node-lease   Active   3h43m   
kube-public       Active   3h43m   
kube-system       Active   3h43m   
monitoring        Active   3h31m   disabled
```
- apply all addons Istio:

```
kubectl apply -f ~/Downloads/istio-installation/istio-1.13.4/samples/addons/

serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

```
k get pods -n istio-system      

NAME                                    READY   STATUS    RESTARTS   AGE
grafana-6c5dc6df7c-kxnf5                1/1     Running   0          2m9s
istio-ingressgateway-76dcc86449-b94dn   1/1     Running   0          23m
istiod-7664dfcb67-hzqtv                 1/1     Running   0          23m
jaeger-9dd685668-qb4x9                  1/1     Running   0          2m1s
kiali-699f98c497-wssrc                  1/1     Running   0          102s
prometheus-699b7cc575-pn8wt             2/2     Running   0          92s
```

```
k get svc -n istio-system 

NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                                      AGE
grafana                ClusterIP      10.100.142.64    <none>                                                                   3000/TCP                                     2m57s
istio-ingressgateway   LoadBalancer   10.100.54.251    a5e070a85a4294bec99dd97ad9a25324-926677944.us-east-1.elb.amazonaws.com   15021:31734/TCP,80:32296/TCP,443:31358/TCP   66m
istiod                 ClusterIP      10.100.217.127   <none>                                                                   15010/TCP,15012/TCP,443/TCP,15014/TCP        66m
jaeger-collector       ClusterIP      10.100.12.79     <none>                                                                   14268/TCP,14250/TCP,9411/TCP                 2m40s
kiali                  ClusterIP      10.100.254.67    <none>                                                                   20001/TCP,9090/TCP                           2m28s
prometheus             ClusterIP      10.100.235.197   <none>                                                                   9090/TCP                                     2m18s
tracing                ClusterIP      10.100.242.232   <none>                                                                   80/TCP,16685/TCP                             2m44s
zipkin                 ClusterIP      10.100.57.205    <none>                                                                   9411/TCP                                     2m42s
```

```
k port-forward svc/kiali -n istio-system 20001:20001
```

- Disable Istio-injection:

```
kubectl label namespace monitoring istio-injection=disabled --overwrite

kubectl get namespace -L istio-injection                               
NAME              STATUS   AGE     ISTIO-INJECTION
cert-manager      Active   3h7m    enabled
ci-cd             Active   3h17m   enabled
default           Active   3h43m   enabled
ingress-nginx     Active   3h5m    
istio-system      Active   156m    
kube-node-lease   Active   3h43m   
kube-public       Active   3h43m   
kube-system       Active   3h43m   
monitoring        Active   3h31m   disabled
```

##### b. Configure istio-gateway:




