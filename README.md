# devops-assignment-k8s-scaling-part2
## Case assumptions
For the sake of simplicity and due to unknown exact use case I assume load increases and decreases not very rapidly. Also I do not consider much cost-optimization at this point. Some considarations aroung cost-optimization and rapid load peaks I'll write about at the end as a way to improve incrementally over time.
In my case to spin up all resources and manage Kubernetes in AWS I'm going to use KOPS. 
For scaling AWS resources I'm going to use cluser-autoscaler (available for KOPS as an addon).
Kubernetes/KOPS configuration:
* k8s version:
* kubectl version:
* KOPS version:
* assumptions for k8s resources requests and limits on pod level (based on best practices and problems you might face on production if set otherwise):
  * Requests/limits set on every deployment, very important to set it properly so we won't face issue with "slack" (gap between requested resources and actually used - causes resources to be not efficiently allocated) and with stranded resources (some capacity of a node is not available because eg. CPU requests allocate full node but Mem request just let's say 70% - not possible to schedule new pods anyway)
  * Memory overcommit forbidden > set requests=limits
  * CPU CSF Quota disabled on the cluster, no CPU throttling (which may significantly impact latencies in multi microservice environment that scales)
* It would be a good idea to restrict Requests and Limits at Namespace level, at least on DEV and PREPROD environment to not trigger scaling (costs) due to incorrect setting in Deployment configuration (ResourceQuotas and LimitRange).
* Cluster size should be determined by resource request (+ some overhead)

## Technical walkthrough

## Considerations around improvements - latencies and costs
### Rapid load increases
Consider using Autoscaling buffer pods, so pod doing nothing with very low priority so that it can be evicted anytime but with requests in place. This reduces time to trigger new node in case of lack of resources, normally this mechanism is triggered when there is pod in pending state which causes latencies.

### Cost optimization
If application is fully stateless and k8s is properly configured to distribute pods across the cluster with resiliency then we might consider using spot instances in AWS to handle peak load. So scenario would be > core stable load on ec2 reserved instances, slow increase of resource consumption handled by EC2 On-Demand, rapid peak load covered by Spot-instances.
We could consider also using downscaling during off-hours, example project that could be levarage here: github.com/hjacobs/kube-downscaler
