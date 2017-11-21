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
#### How does the Kubernetes talk to AWS?

Lets first look at the kube-apiserver logs to see if there is anything that can indicate any cloud provider movement. 

_Note:_  Picking to start with the kube-apiserver since the kubectl client goes through the api server to store its resources.  *I later discover, as you'll see below that this is the wrong path ;)*

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

And there you have it, importing this aws package from [aws.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/providers/aws/aws.go) then calls the `func init()` on load which [registers](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/pkg/cloudprovider/plugins.go#L44) the `newAWSCloud` instance to memory by its `cloud-provider` (aws).  So when the kube-apiserver needs a cloud context, it does a map lookup on the `cloud-provider` to its registry and gets the concrete implementation of the cloud provider interface. Something to keep in mind is that all of this cloud provider set up happens before we actually run the kub-apiserver `main()` function.  

So.. at this point, the kube-apiserver is now able to build a aws provider given the aws configs from `cloud-configs` flag. It stores all of the cloud providers in its registry and it's ready for the kube-apiserver code to use when needed.  


cool!!  

Aaaannd... here is the point where I found myself in a dead end, not being able to tie in the kube-apiserver command to any of the load balancer code from the aws provider... hmmm. 

Seems like the kube-apiserver was the wrong path for my purpose of figuring out how elbs are created as part of a new service resource.
#### So then.. what actually calls the code that updates the aws load balancers?

## Controllers

So what I realized then is that creating the actual `kind:Service` resource does not actually create the elb.  This just writes to the api server and stores the manifest information to its datastore (etcd).  Once the resource has be stored and made available to the kube-apiserver a *controller* reacts and attempts to reconcile a desired state.

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

## Controller Manager

As mentioned above, the manager is responsible for starting the built-in control loops. Similar to the kube-apiserver, we can take a look at how the kube-controller-manager is started and with which flags.


```bash
k describe pod kube-controller-manager-ip-100-68-14-xxx.us-west-2.compute.internal -n kube-system
```

Inspecting the output we can find the start command used and the mounts used.

```bash
/usr/local/bin/kube-controller-manager i
--cloud-provider=aws 
--cluster-name=k8-dev.dev.cloud.motorola.net 
--v=2 
--kubeconfig=/var/lib/kube-controller-manager/kubeconfig 
--cloud-config=/etc/kubernetes/cloud.config 
 1>>/var/log/kube-controller-manager.log 2>&1
...
Mounts:
      /etc/kubernetes/cloud.config
```

cool...

Lets take a look at the kube-controller-manager command. 

Before main() is even called, the `main` package handles its imports.  In particular, it imports the `k8s.io/kubernetes/cmd/kube-controller-manager/app` package.  The app does its import dance and gets to [plugins.go](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/plugins.go#L28) where it then imports the cloud providers package as a blank identifier. The `providers` package then loads all of the cloud providers available, again with [blank identifiers](https://github.com/kubernetes/kubernetes/blob/v1.7.10/pkg/cloudprovider/providers/providers.go).  Each package registers itself as part of their `init()` function when they are imported. At this point before main(), the `kube-controller-manager` already has in memory all of the cloud providers built, registered and ready for use.

Lets look at the main() now.

[controller-manager.go](https://github.com/kubernetes/kubernetes/blob/v1.7.10/cmd/kube-controller-manager/controller-manager.go) starts in main() by creating a new cm server() which sets up default configs.  Then, it reads the flags passed into the binary and sets them as options. These are the arguments passed into the kube-controller-manager binary (--cloud-provider, --cluster, --cloud-config) from above. 

Once its has the flags, logs and metrics set up, it calls `app.Run(s)` to *run* the controller manager server. Here the code begins by validating the KnownControllers, setting up the kube configs to talk to master by reading from the `--kubeconfig` flag and creates a [http server](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/controllermanager.go#L152) debuging/metrics.

At this point you should be able to validate the handler endpoints like /metrics for example which is used by prometheus scraping.
```bash
wget -S -q localhost:10252/metrics
  HTTP/1.1 200 OK
  Content-Length: 43944
  Content-Type: text/plain; version=0.0.4
  Date: Tue, 21 Nov 2017 19:50:42 GMT
  Connection: close
```

Then it locks the kube-controller-manager using the LeaderElection resource type so that it can elect a leader.  Once it has a leader, it spawns the [run](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/controllermanager.go#L160) goroutine to iterate and *start* each built-in controller.

Looking at the `run` goroutine, [CreateControllerContext](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/controllermanager.go#L175) will create the cloud context based on the `--cloud-provider` flag to do a map lookup on the registered cloud providers initialized earlier.  The controller context, along with its cloud provider instance ([ctx.Cloud](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/controllermanager.go#L411)),  is then used to start the list of controllers. 

[StartControllers](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/controllermanager.go#L181) takes in as a parameter a map of controllers to process.  Looking at the [NewControllerInitializers()](https://github.com/kubernetes/kubernetes/blob/bebdeb749f1fa3da9e1312c4b08e439c404b3136/cmd/kube-controller-manager/app/controllermanager.go#L307), we see the list of controllers to start and listed is `controllers["service"] = startServiceController`. Bingo!

_Note:_ The `controllers["service"]` value is of type `InitFunc` which takes in as a parameter a ControllerContext and returns (bool,error) 

So now the StartControllers function iterates through controllers map calling each value of each key.  When it gets to the `"service"` key, since the value is of type func, calling `initFun(ctx)` is like calling `startServiceController(ctx ControllerContext)` from [core.go](https://github.com/kubernetes/kubernetes/blob/v1.7.10/cmd/kube-controller-manager/app/core.go)

The startServiceController function creates a New serviceController instance and finally calls `serviceController.Run` as a goroutine.

Bam!  We've fully traced how a controller manager is started, how it builds its cloud providers, how it iterates through its built-in controllers and finally how it starts with `run` each controller.  This circles us back to the kube-controller-manager log we started with where the service controller continues to set up the watch to handle deletes, creates or adding back to the queue.

Now lets deep dive into the aws logic...

## Tips
#### What are some of things that helped trace through the code?
- Run the tests on the aws package to step through the cloud provider code.  For example, using vscode you can easily run `debug test` to step into/through the code from aws.go
```bash
Running tool: /usr/local/go/bin/go test -timeout 30s k8s.io/kubernetes/pkg/cloudprovider/providers/aws -run ^TestDescribeLoadBalancerOnDelete$

ok  	k8s.io/kubernetes/pkg/cloudprovider/providers/aws	0.077s
Success: Tests passed.
```

- Take a look at the CloudTrail event history from the AWS console.  With CloudTrail, you can filter by the userName and use the master node which is running the kube-apiserver to filter the results (i-035be74d9d4f68).  This gives you a sequential list of the apis called by the kubernetes code to configure the AWS elb.  For example, you can see events like CreateSecurityGroup, ModifyLoadBalancerAttributes, CreateLoadBalancer and AuthorizeSecurityGroupIngress.  This helped out identifying what apis were called from the code.
- You can use kubectl edit to look at the kube-controller-manager and kube-apiserver manifest files to inspect what flags are set and mounts will be created. 
```bash
k edit -n kube-system pod kube-apiserver-ip-100-68-14-xxx.us-west-2.compute.internal 
```
