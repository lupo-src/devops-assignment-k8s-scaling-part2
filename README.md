# devops-assignment-k8s-scaling-part2
## Case assumptions
For the sake of simplicity and due to unknown exact use case I assume load increases and decreases not very rapidly. Also I do not consider much cost-optimization at this point. Some considarations aroung cost-optimization and rapid load peaks I'm going to write about at the end as a way to improve incrementally over time.

In my case to spin up all resources and manage Kubernetes in AWS I'm going to use KOPS. 

For scaling AWS resources I'm going to use cluser-autoscaler (available for KOPS as an addon) and HPA with metric-server.
Kubernetes/KOPS configuration:
* k8s version: 1.14.6
* kubectl version: 1.16.2
* KOPS version: 1.14.0
* assumptions for k8s resources requests and limits on pod level (based on best practices and problems you might face on production if set otherwise):
  * Requests/limits set on every deployment, very important to set it properly so we won't face issue with "slack" (gap between requested resources and actually used - causes resources to be not efficiently allocated) and with stranded resources (some capacity of a node is not available because eg. CPU requests allocate full node but Mem request just let's say 70% - not possible to schedule new pods anyway)
  * Memory overcommit forbidden > set requests=limits
  * CPU CSF Quota disabled on the cluster, no CPU throttling (which may significantly impact latencies in multi microservice environment that scales)
* It would be a good idea to restrict Requests and Limits at Namespace level, at least on DEV and PREPROD environment to not trigger scaling (costs) due to incorrect setting in Deployment configuration (ResourceQuotas and LimitRange).
* Cluster size should be determined by resource request (+ some overhead)

## Technical walkthrough

#### Requirements: KOPS, GIT, kubectl, AWSCLI installed
#### Preparing for KOPS:
```
aws iam create-group --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
aws iam create-access-key --user-name kops
```

#### Configure the aws client to use your new IAM user
```
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here
```

#### Because "aws configure" doesn't export these vars for kops to use, we export them now
```
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

#### We use following version of k8s
```
kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.2", GitCommit:"c97fe5036ef3df2967d086711e6c0c405941e14b", GitTreeState:"clean", BuildDate:"2019-10-15T19:18:23Z", GoVersion:"go1.12.10", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.6", GitCommit:"96fac5cd13a5dc064f7d9f4f23030a6aeface6cc", GitTreeState:"clean", BuildDate:"2019-08-19T11:05:16Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

We can use gossip-based cluster and ommit setting up DNS zones (for the sake of simplicity in this scenario)

The only requirement to trigger this is to have the cluster name end with .k8s.local

#### Creating buckets for KOPS state store
```
aws s3api create-bucket \
    --bucket <<your_bucket_name>> \
    --region eu-north-1
	--create-bucket-configuration LocationConstraint=eu-north-1
```

#### So now setting up env vars for KOPS
```
export NAME=test.k8s.local
export KOPS_STATE_STORE=s3://lupo-kops-state-store
```

#### Creating cluster config and SSH keys
```
kops create cluster --zones=eu-north-1a --name=${NAME} --node-count=1 --node-size=t3.micro --master-size=t3.micro
kops create secret --name test.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub
kops edit cluster ${NAME} #adjust if needed
```

#### Spinning up the cluster in AWS
```kops update cluster ${NAME} --yes```

#### Verifying the setup
```
kops validate cluster
kubectl get nodes
kubectl -n kube-system get pods
```

#### Deploying metric-server
```curl https://raw.githubusercontent.com/kubernetes/kops/master/addons/metrics-server/v1.8.x.yaml > metric-server.yaml```
#### Edit metric server to work with self signed cert
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: metrics-server
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.3
        imagePullPolicy: Always
        args: [ "--kubelet-insecure-tls" ]
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```

#### And the cluster itself
```kops edit cluster --name {cluster_name}```
#### Edit following part to (also to allow self signed certs)
```yaml 
    kubelet:
      anonymousAuth: false
      authenticationTokenWebhook: true
      authorizationMode: Webhook
```

#### Then run following commands
```
kops update cluster --yes
kops rolling-update cluster --yes

kubectl apply -f metric-server.yaml
```
#### Deploy dashboard (optional, just to observe how nodes/pods scale, review events etc.)
```kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml```
#### Launch proxy to be able to connect to dashboard from the browser
```kubectl proxy```

#### Deploy load testing application 
```
git clone https://gitlab.com/wuestkamp/kubernetes-scale-that-app.git
cd kubernetes-scale-that-app/
kubectl kustomize i | kubectl apply -f -
```

#### Veryfing the deployment
```
$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
app-7f7d49d44f-d9rzp   1/1     Running   0          35s
app-7f7d49d44f-hfkx8   1/1     Running   0          35s
app-7f7d49d44f-kf486   1/1     Running   0          35s
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)          AGE
kubernetes   ClusterIP      100.64.0.1      <none>                                                                     443/TCP          103m
lb           LoadBalancer   100.71.52.108   a8471cffbff1011e9820c06507077d6e-1622315427.eu-north-1.elb.amazonaws.com   5000:32274/TCP   52s
```

#### Hit the page http://<<elb_dns>>:5000 from few diffrent terminals so ELB will dispatch to all pods
#### Watch how it utilizes all available resources on node and pods
```
kubectl top node
kubectl top pod
```

#### Now let set up limits and requests
```yaml
spec:
  containers:
  - image: registry.gitlab.com/wuestkamp/kubernetes-scale-that-app:part1
    imagePullPolicy: Always
    name: app
    resources:
      requests:
        cpu: "200m"
      limits:
        cpu: "400m"
```

#### Let see how k8s handles this, one hit on the page http://<<elb_dns>>:5000 
```
$ kubectl top pods
NAME                   CPU(cores)   MEMORY(bytes)
app-6d7457dc77-55bdp   301m         160Mi
app-6d7457dc77-r76js   4m           34Mi
app-6d7457dc77-rjwfk   4m           34Mi
$ kubectl top nodes
NAME                                           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-20-45-69.eu-north-1.compute.internal    82m          4%     725Mi           83%
ip-172-20-62-197.eu-north-1.compute.internal   246m         12%    564Mi           64%
```

#### Second hit on page above
```
$ kubectl top pods
NAME                   CPU(cores)   MEMORY(bytes)
app-6d7457dc77-55bdp   4m           37Mi
app-6d7457dc77-r76js   300m         243Mi
app-6d7457dc77-rjwfk   300m         192Mi
$ kubectl top nodes
NAME                                           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-20-45-69.eu-north-1.compute.internal    127m         6%     738Mi           85%
ip-172-20-62-197.eu-north-1.compute.internal   1525m        76%    586Mi           67%
```

#### Now let's set up HPA to see how k8s scales pods
#### This should more or less maintain an average cpu usage across all pods of 50% - just for testing purposes, in real case scenario that would need to be researched in order to apply proper threshold
```
kubectl autoscale deployment app --cpu-percent=50 --min=3 --max=10
kubectl get hpa
NAME   REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app    Deployment/app   34%/50%   3         10        3          47s
```

#### After configuring HPA and hitting page many times following problem occurs:
```
$ kubectl get pods
NAME                  READY   STATUS             RESTARTS   AGE
app-776d4cd8f-kjzf2   1/1     Running            2          10m
app-776d4cd8f-s45g4   1/1     Running            1          10m
app-776d4cd8f-th4ct   1/1     Running            0          101s
app-776d4cd8f-ww9cq   1/1     Running            0          101s
app-776d4cd8f-wwqrh   0/1     CrashLoopBackOff   2          10m
$ kubectl top nodes
NAME                                           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-20-45-69.eu-north-1.compute.internal    164m         8%     743Mi           85%
ip-172-20-62-197.eu-north-1.compute.internal   2839m        141%   474Mi           54%

$ kubectl get hpa
NAME   REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app    Deployment/app   55%/50%   3         10        5          10m
```

#### So we see already although scaling of pods is happening as intended we need additional resources to scale furhter. 
#### And now it comes to cluster-autoscaler
#### Let's configure Autoscaller Addon
```
wget https://github.com/kubernetes/kops/blob/master/addons/cluster-autoscaler/cluster-autoscaler.sh
```
#### Adjust the script
```
#Set all the variables in this section
CLUSTER_NAME="test.k8s.local"
CLOUD_PROVIDER=aws
IMAGE=k8s.gcr.io/cluster-autoscaler:v1.1.0
MIN_NODES=1
MAX_NODES=10
AWS_REGION=eu-north-1
INSTANCE_GROUP_NAME="nodes"
ASG_NAME="${INSTANCE_GROUP_NAME}.${CLUSTER_NAME}"   #ASG_NAME should be the name of ASG as seen on AWS console.
IAM_ROLE="masters.${CLUSTER_NAME}"                  #Where will the cluster-autoscaler process run? Currently on the master node.
SSL_CERT_PATH="/etc/ssl/certs/ca-certificates.crt"  #(/etc/ssl/certs for gce, /etc/ssl/certs/ca-bundle.crt for RHEL7.X)
```

#### Launch above script
```
./cluster-autoscaler.sh
```

#### Then let's hit page many times with Apache Bench 
```ab -s 600 -n 100 -c 5 http://<<elb_dns>>:5000/```

#### We can already notice that one node has been added:
```
kubectl top nodes
NAME                                           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-20-56-7.eu-north-1.compute.internal     1954m        97%    719Mi           82%
ip-172-20-57-213.eu-north-1.compute.internal   883m         44%    690Mi           79%
ip-172-20-63-134.eu-north-1.compute.internal   118m         5%     723Mi           83%
```

#### Let's go one step further and try to request even more computation resources 
#### To trigger resources more quickly I'm going to add more resources on pod level.
```yaml
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 500m
```
#### And let's benchmark this using just one request at a time but many times in a raw (I noticed that this apps is not handling well concurent requests)
```ab -s 600 -n 200 -c 1 http://a51e60c22014b11eabcba0e2fcc39146-271971402.eu-north-1.elb.amazonaws.com:5000/ ```
#### As a result pending pods have appeared and yet another new node has been initiated
```
$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
app-6d45d686cd-2ndx4   0/1     Pending   0          105s
app-6d45d686cd-gv7dm   1/1     Running   5          17m
app-6d45d686cd-kv95d   0/1     Pending   0          105s
app-6d45d686cd-lnb4n   1/1     Running   0          105s
app-6d45d686cd-lx5l2   1/1     Running   5          17m
app-6d45d686cd-lz4nc   0/1     Pending   0          105s
app-6d45d686cd-qvs9x   1/1     Running   0          2m45s
app-6d45d686cd-t69kn   1/1     Running   5          17m
$ kubectl get nodes
NAME                                           STATUS   ROLES    AGE     VERSION
ip-172-20-39-12.eu-north-1.compute.internal    Ready    node     4h21m   v1.14.6
ip-172-20-50-177.eu-north-1.compute.internal   Ready    node     51m     v1.14.6
ip-172-20-59-137.eu-north-1.compute.internal   Ready    master   4h26m   v1.14.6
ip-172-20-59-90.eu-north-1.compute.internal    Ready    node     14s     v1.14.6
```
#### Eventually I reached the limit of scaling cluster which was set to 10 nodes. It's been achieved quite quictly (few minutes of scaling up)
```
kubectl get pods
NAME                   READY   STATUS     RESTARTS   AGE
app-6d45d686cd-27p58   0/1     OutOfcpu   0          30s
app-6d45d686cd-2b7gh   0/1     OutOfcpu   0          27s
app-6d45d686cd-2n9b5   0/1     OutOfcpu   0          35s
app-6d45d686cd-2ndx4   1/1     Running    0          6m17s
app-6d45d686cd-2wrjx   1/1     Running    0          58s
app-6d45d686cd-4pg2k   0/1     Pending    0          37s
app-6d45d686cd-568qb   0/1     OutOfcpu   0          31s
[...]
Many lines here
[...]
app-6d45d686cd-t69kn   1/1     Running    6          21m
app-6d45d686cd-thscn   0/1     OutOfcpu   0          30s
app-6d45d686cd-tq7jp   0/1     OutOfcpu   0          26s
app-6d45d686cd-tvf72   0/1     OutOfcpu   0          2m15s
app-6d45d686cd-txpwk   0/1     OutOfcpu   0          34s
app-6d45d686cd-vf42s   0/1     OutOfcpu   0          34s
app-6d45d686cd-w4474   0/1     OutOfcpu   0          30s
app-6d45d686cd-wg2zg   0/1     OutOfcpu   0          36s
app-6d45d686cd-whjcn   0/1     Pending    0          58s
app-6d45d686cd-whtgd   0/1     OutOfcpu   0          30s
app-6d45d686cd-whtmq   0/1     OutOfcpu   0          26s
app-6d45d686cd-whw64   0/1     OutOfcpu   0          32s
app-6d45d686cd-wj4mv   0/1     OutOfcpu   0          58s
app-6d45d686cd-wqnrc   0/1     OutOfcpu   0          35s
app-6d45d686cd-x474p   0/1     Pending    0          37s
app-6d45d686cd-zgx98   0/1     OutOfcpu   0          33s
app-6d45d686cd-zkh4m   0/1     OutOfcpu   0          2m15s
app-6d45d686cd-zmmc7   0/1     OutOfcpu   0          30s
kubectl get nodes
NAME                                           STATUS   ROLES    AGE     VERSION
ip-172-20-35-33.eu-north-1.compute.internal    Ready    node     4m54s   v1.14.6
ip-172-20-35-98.eu-north-1.compute.internal    Ready    node     6m4s    v1.14.6
ip-172-20-39-12.eu-north-1.compute.internal    Ready    node     4h30m   v1.14.6
ip-172-20-40-78.eu-north-1.compute.internal    Ready    node     6m5s    v1.14.6
ip-172-20-40-92.eu-north-1.compute.internal    Ready    node     5m6s    v1.14.6
ip-172-20-41-238.eu-north-1.compute.internal   Ready    node     5m36s   v1.14.6
ip-172-20-50-177.eu-north-1.compute.internal   Ready    node     61m     v1.14.6
ip-172-20-59-137.eu-north-1.compute.internal   Ready    master   4h36m   v1.14.6
ip-172-20-59-90.eu-north-1.compute.internal    Ready    node     9m58s   v1.14.6
ip-172-20-62-71.eu-north-1.compute.internal    Ready    node     5m7s    v1.14.6
[...]
```
#### Then 20-30 minutes nodes were scaled down and back to initial setup

### Summary of the setup and outcomes
Above described setup is not perfect and was meant to just show what approach I'd take. Definitely one the most important things to focus on regarding scaling is to properly fine tune thresholds and requests/limits. The scenario that I went through only considers CPU utilization, I did not focused on memory. Moreover to properly scale up and down the application itself has to be properly designed so that, for instance, if there is a delay to spawn new resources then app and overall architecture of infastructure can handle retries etc. Topic around scaling is very broad and to properly address requirements and issues one need to know application design and architecture very well. Also a lot o improvements can be made within AWS infra, like for instance tunning ELB etc.

## Considerations around improvements - latencies and costs
### Rapid load increases
Consider using Autoscaling buffer pods, so pod doing nothing with very low priority so that it can be evicted anytime but with requests in place. This reduces time to trigger new node in case of lack of resources, normally this mechanism is triggered when there is pod in pending state which causes latencies.

### Cost optimization
If application is fully stateless and k8s is properly configured to distribute pods across the cluster with resiliency then we might consider using spot instances in AWS to handle peak load. So scenario would be > core stable load on ec2 reserved instances, slow increase of resource consumption handled by EC2 On-Demand, rapid peak load covered by Spot-instances.
We could consider also using downscaling during off-hours, example project that could be levarage here: github.com/hjacobs/kube-downscaler
