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

### Requirements: KOPS, GIT, kubectl, AWSCLI installed
### Preparing for KOPS:
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

### Configure the aws client to use your new IAM user
```
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here
```

### Because "aws configure" doesn't export these vars for kops to use, we export them now
```
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

### We use following version of k8s
```
kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.2", GitCommit:"c97fe5036ef3df2967d086711e6c0c405941e14b", GitTreeState:"clean", BuildDate:"2019-10-15T19:18:23Z", GoVersion:"go1.12.10", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.6", GitCommit:"96fac5cd13a5dc064f7d9f4f23030a6aeface6cc", GitTreeState:"clean", BuildDate:"2019-08-19T11:05:16Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

We can use gossip-based cluster and ommit setting up DNS zones (for the sake of simplicity in this scenario
The only requirement to trigger this is to have the cluster name end with .k8s.local

### Creating buckets for KOPS state store
```
aws s3api create-bucket \
    --bucket <<your_bucket_name>> \
    --region eu-north-1
	--create-bucket-configuration LocationConstraint=eu-north-1
```

### Creating cluster config and SSH keys
```
kops create cluster --zones=eu-north-1a --name=${NAME} --node-count=1 --node-size=t3.micro --master-size=t3.micro
kops create secret --name test.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub
kops edit cluster ${NAME} #adjust if needed
```

### So now setting up env vars for KOPS
```
export NAME=test.k8s.local
export KOPS_STATE_STORE=s3://lupo-kops-state-store
```

### Spinning up the cluster in AWS
```kops update cluster ${NAME} --yes```

### Verifying the setup
```
kops validate cluster
kubectl get nodes
kubectl -n kube-system get pods
```

### Deploying metric-server
```curl https://raw.githubusercontent.com/kubernetes/kops/master/addons/metrics-server/v1.8.x.yaml > metric-server.yaml```
### Edit metric server to work with self signed cert
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

### And the cluster itself
```kops edit cluster --name {cluster_name}```
### Edit following part to (also to allow self signed certs)
```yaml 
    kubelet:
      anonymousAuth: false
      authenticationTokenWebhook: true
      authorizationMode: Webhook
```

### Then run following commands
```
kops update cluster --yes
kops rolling-update cluster--yes

kubectl apply -f metric-server.yaml
```
### Deploy dashboard
```kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml```
### Launch proxy to be able to connect to dashboard from the browser
```kubectl proxy```

### Deploy load testing application 
```
git clone https://gitlab.com/wuestkamp/kubernetes-scale-that-app.git
cd kubernetes-scale-that-app/
kubectl kustomize i | kubectl apply -f -
```

### Veryfing the deployment
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

### Hit the page http://<<elb_dns>>:5000 from few diffrent terminals so ELB will dispatch to all pods (of course adjust 
### Watch how it utilizes all available resources on node and pods
```
kubectl top node
kubectl top pod
```

### Now let set up limits and requests (with a rule limit=request, explanation why this way described later on)
```yaml
spec:
  containers:
  - image: registry.gitlab.com/wuestkamp/kubernetes-scale-that-app:part1
    imagePullPolicy: Always
    name: app
    resources:
      requests:
        cpu: "400m"
      limits:
        cpu: "400m"
```

### Let see how k8s handles this, one hit on the page http://<<elb_dns>>:5000 
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

### Second hit on page above
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

### Now let's set up HPA to see how k8s scales pods
### This should more or less maintain an average cpu usage across all pods of 50% - just for testing purposes, in real case scenario that would need to be researched in order to apply proper threshold
```
kubectl autoscale deployment app --cpu-percent=50 --min=3 --max=10
kubectl get hpa
NAME   REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app    Deployment/app   34%/50%   3         10        3          47s
```

### After configuring HPA and hitting page many times following problem occurs:
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

### So we see already although scaling of pods is happening as intended we need additional resources to scale furhter. 
### Also it often causes API to be unavailable:
```
$ kubectl top nodes
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)
$ kubectl top pods
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get pods.metrics.k8s.io)
```

### And now it comes to cluster-autoscaler


## Considerations around improvements - latencies and costs
### Rapid load increases
Consider using Autoscaling buffer pods, so pod doing nothing with very low priority so that it can be evicted anytime but with requests in place. This reduces time to trigger new node in case of lack of resources, normally this mechanism is triggered when there is pod in pending state which causes latencies.

### Cost optimization
If application is fully stateless and k8s is properly configured to distribute pods across the cluster with resiliency then we might consider using spot instances in AWS to handle peak load. So scenario would be > core stable load on ec2 reserved instances, slow increase of resource consumption handled by EC2 On-Demand, rapid peak load covered by Spot-instances.
We could consider also using downscaling during off-hours, example project that could be levarage here: github.com/hjacobs/kube-downscaler
