# WIP: Kubernetes-Service-Controller-AWS-ELB-Walkthrough
When you create a `kind:Service` resource with

```yaml
apiVersion: v1
kind: Service
metadata:
  name: paymentuxe
  namespace: develop-paymentuxe
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9000
  selector:
    app: paymentuxe
  type: LoadBalancer
```

OR when you expose a deployment resource as a `kind:Service`
```bash
kubectl expose deployment paymentuxe --type=LoadBalancer --port=80 --target-port=9000
```

The kubernetes code creates, *for you*, a AWS Classic Load Balancer fully configured and ready to take on incoming traffic. 

The purpose of this document is help understand *how* exactly kubernetes is doing this for us. Where's the magic? How does it *know* that we are using AWS? Where in the code is it actually making the aws api requests? How does it know *when* to make this request? How is it setting what subnet, security groups or vpcs to use? How can you specify or override options with annotations? How do you configure this with kops? How does kops tell kubernetes what config options to use? Where are these configs stored and injected into the kubneretes binaries (kube-apiserver/kube-controller-manager)??


## Getting Started

### Binaries
```
kops version
Version 1.6.2
...
kubectl
Client Version: v1.8.0
Server Version: v1.7.10
...
go version
go version go1.9.2 darwin/amd64
```
### Setting up a local kubernetes dev environment

Followed the kubernetes dev community [readme](https://github.com/kubernetes/community/blob/master/contributors/devel/development.md#1-fork-in-the-cloud) to get a clone of the repo in my $GOPATH, set up remote upstream, create branch and rebase.

To match the code lines from the kubernetes logs, I did a checkout of the working server version
```git
git checkout v1.7.10
```

### Setting a kubernetes cluster with kops
```
export NAME=k8-internal.dev.cloud.eguevara.net

kops create cluster \
--master-count 1 --master-zones us-west-2a \
--node-count 1 --zones us-west-2a \
--topology private --dns-zone dev.cloud.eguevara.net --networking calico \
--target=terraform --out=. \
--authorization RBAC \
--cloud aws \
--bastion \
--cloud-labels="Project=Shared Services,Product=DevOps" \
$NAME
```
Since I'm  using the `--target` and `--out` to output the terraform templates, I still need to run `terraform plan` and `terraform apply`.  I prefer this option so that I can review all of the AWS resources that will be created. 

Once all of this is done, you can validate by
```bash
kops validate cluster
...
k cluster-info
```

At this point you should have all of the kubernetes components up and running.
```bash
k get cs
...
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

Now lets jump into how, through kops, we configure aws global options to tell kubernetes code to behave differently with options like `disableSecurityGroupIngress`, `ElbSecurityGroup` from the [CloudConfig](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go#L400) type.

## AWS Configure Options
#### How do you add AWS configs so that the kubernetes can build the AWS provider?

Through the kops cluster specs, you can add additional cloud config settings to drive specific AWS behavior.  For example, you can add the `disableSecurityGroupIngress` flag so that kubernetes does not add the ELB sg to the node sg.  To do this, edit the cluster with kops and add:
```yaml
spec:
  cloudConfig:
    disableSecurityGroupIngress: true
```

When the kube-apiserver binary is started on the master node, it uses these kops cluster spec changes to add them as flags when running the binary.

```bash
 /usr/local/bin/kube-apiserver 
 ...
 --cloud-provider=aws 
 --cloud-config=/etc/kubernetes/cloud.config 1>>/var/log/kube-apiserver.log 2>&1
```

The `cloudConfig: disableSecurityGroupIngress` settings are stored in the `cloud.config` file and then mounted to the kube-apiserver container.

```bash
Mounts:
      /etc/kubernetes/cloud.config
```

To find out how the binary was started with all of the flags and mounts you can describe the resource with 
```bash
k describe pod kube-apiserver-ip-100-68-14-xxx.us-west-2.compute.internal -n kube-system
```

At this point, kubernetes has all of the initial aws configs needed to build an aws cloud provider.

## Building the AWS Provider
#### How does the kubernetes build the AWS cloud provider?

Lets first look at the kube-apiserver logs to see if there is anything that can indicate any cloud provider movement.  

You can view the kube-apiserver logs by running
```bash
kubectl exec -it kube-apiserver-ip-100-68-14-xxx.us-west-2.compute.internal /bin/sh -n kube-system
...
cat /var/log/kube-apiserver
...
aws.go:806] Building AWS cloudprovider
```

ahhh... The logs spit out *aws.go:806* with a *Building AWS cloudprovier* message.

[aws.go:806](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go#L806) is part of the of the `newAWSCloud` function that creates a new instance (awsCloud) of the `Cloud` type setting up things like the aws metadata, elb, ec2 and asg services and returns it. It also importantly reads the `cloud-config` file and converts the file into a [CloudConfig](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go#L813) type.

Once you have the aws global configs read from `cloud-config`, other `awsCloud` fields are set.  However our use case does not set any other options so it uses the [metadata](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go#L880) service to figure out the instance and vpc info based on the actual (self) node thats its running on.

`awsCloud.tagging` is also set based on the nodes tags (tag: KubernetesCluster). This tag is used to filter resources based on clusterID (k8-dev.dev.cloud.eguevara.net)

After all this setting up, the func returns the `awsCloud` instance.

#### So... what calls the `newAwsCloud` func?  
Tracing back the `newAWSCloud` function, you'll find that its called as part of the [init()](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go#L738) function that gets called when the `aws` package is imported.

#### So then... what imports the `aws` package?

It all starts with the main package from `cmd/kube-apiserver` command in [apiserver.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-apiserver/apiserver.go)

The main package imports `k8s.io/kubernetes/cmd/kube-apiserver/app`

Importing the `app` package then imports a blank identifer for the cloud providers as part of the [plugins.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-apiserver/app/plugins.go) file.

```go
_ "k8s.io/kubernetes/pkg/cloudprovider/providers"
```

Following the `providers` package then leads to yet another blank identifer for the `aws` package as part of the [providers.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/providers.go) file.

```go
_ "k8s.io/kubernetes/pkg/cloudprovider/providers/aws"
```

And there you have it, importing this aws package from [aws.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go) then calls the `func init()` on load which ties it all together by returning `newAWSCloud` which is an implementation of the cloudprovider interface.

cool!!  

At this point now, the kube-apiserver is now able to build a aws provider given the aws configs from `cloud-configs` flag. 

#### So now... How does kubernetes watch/listen to when it needs to create an external load balancer on a service resource change?

## Controllers

So what I realized at this point is that creating the actual `kind:Service` resource does not actually create the elb.  This just writes to the api server and stores the manifest information to its datastore (etcd).  Once the resource has be stored and made available to the kube-apiserver a controller reacts and attempts to reconcile a desired state.

### Wait what? What the hell is a controller?
A [kubernetes controller](https://kubernetes.io/docs/reference/generated/kube-controller-manager/) will watch a resource and implement the behavior (code)
that is needed to maintain a stable state for that resource.  Kubernetes has
a list of built-in controllers (nodeController, volumeController,
serviceController etc) to work with.  All of these controllers are managed by the kube-controller-manager binary that is started on the master node.  

Just reading the list of controllers available, the serviceController looks like it's exactly what we are looking for. It's responsible for creating and deleting LoadBalancers for a given cloud provider.

That said, we should be able to look at the logs to see if there is anything
relevant to cloud providers that maps back to the serviceController. Lets dig into the kube-controller-manager. 

```bash
k exec -n kube-system kube-controller-manager-ip-100-68-14-xxx.us-west-2.compute.internal /bin/sh
...
cat /var/log/kube-controller-manager.log
...
servicecontroller.go:176] Starting service controller
```

aha.. soo.. The controller-manager is looping through each registered controller and starting them as background threads within the the kube-controller-manager binary (process). 

The serviceController, indicated by the logs, gets started by the [servicecontroller.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/controller/service/servicecontroller.go#L176) file 

## Service Controller
The serviceController is implemented like any other controller where it waits for the cache to sync, then fires off some workers as separate goroutines to read the key from a queue then sets up the watch ([syncService](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/controller/service/servicecontroller.go#L724:29)) to listen for events to occur. In our case for the service controller, the watch listens to changes to service resources that have `LoadBalancer=true`.

Based on the lister service, which fetches the latest service info from the apiserver for that key, the code will [process](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/controller/service/servicecontroller.go#L738) the delete or update code of the service or again add the key back to the queue.

The [processServiceDeletion](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/controller/service/servicecontroller.go#L771) and [processServiceUpdate](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/controller/service/servicecontroller.go#L226) then leads to exactly the aws code we are looking for by *ensuring* the load balancer state is fulfilled.  

cool...

At this point, we've traced back how the serviceController is called by the manager to set up the watches on the service resource.  Before we jump into the aws logic that is processed by the worker on events, lets go back a bit to figure out how the manager loads the serviceController in the first place. 

#### How does the kube-controller-manager know about the serviceController? How does it know how many workers to kick off? How does it know cloud provider info to hand off to the controller to manage the elb?



## Tips
#### What are some of things that helped trace through the code?
- Run the tests on the aws package to step through the cloud provider code.  For example, using vscode you can easily run `debug test` to step into/through the code from aws.go
```bash
Running tool: /usr/local/go/bin/go test -timeout 30s k8s.io/kubernetes/pkg/cloudprovider/providers/aws -run ^TestDescribeLoadBalancerOnDelete$

ok  	k8s.io/kubernetes/pkg/cloudprovider/providers/aws	0.077s
Success: Tests passed.
```

- Take a look at the CloudTrail event history from the AWS console.  With CloudTrail, you can filter by the userName and use the master node which is running the kube-apiserver to filter the results (i-035be74d9d4f68).  This gives you a sequential list of the apis called by the kubernetes code to configure the AWS elb.  For example, you can see events like CreateSecurityGroup, ModifyLoadBalancerAttributes, CreateLoadBalancer and AuthorizeSecurityGroupIngress.  This helped out identifying what apis were called from the code.
