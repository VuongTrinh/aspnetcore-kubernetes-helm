Run ASP.NET Core 3 on Kubernetes with Helm.
===========================================

In this post I'm going to show you step-by-step how to create, build and host ASP.NET Core 3 web API on Kubernetes with a little help from Helm 3.

==========================================================================================================

I've been using Windows 10 Pro but you should be fine running it on Linux or MacOS.

There's a few things you have to install before going further:

-   [Git](https://git-scm.com/downloads)
-   [Docker Desktop](https://hub.docker.com/?overlay=onboarding) - version 2.1.0.5 or higher
-   [.Net Core 3.0](https://dotnet.microsoft.com/download/dotnet-core/3.0)
-   [Helm](https://v3.helm.sh/docs/intro/install/) - version 3.0.0 or higher

========================================================================================

Create three instances of ASP.NET Core 3 web API behind a 'load balancer' that will route to any of them on Kubernetes local cluster.

[![Demo](https://res.cloudinary.com/practicaldev/image/fetch/s--KIyP8OwB--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://thepracticaldev.s3.amazonaws.com/i/73e1ozkebvb456mlxoqa.gif)](https://res.cloudinary.com/practicaldev/image/fetch/s--KIyP8OwB--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://thepracticaldev.s3.amazonaws.com/i/73e1ozkebvb456mlxoqa.gif)

[](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#create-aspnet-core-3-web-api)Create ASP.NET Core 3 Web API
=========================================================================================================================================

Let's start by creating a new Web API using `dotnet new` command. Open Git Bash as an administrator and type command which will create a project template in `app` directory.

```
dotnet new webapi -o app

```

Now, open the `app` directory and launch the web API

```
cd app/
dotnet run watch

```

Navigate to `localhost:5000/weatherforecast` to make sure the app is running.

```
[{"date":"2019-12-01T18:13:28.0973127+01:00","temperatureC":35,"temperatureF":94,"summary":"Warm"},
{"date":"2019-12-02T18:13:28.0975531+01:00","temperatureC":-14,"temperatureF":7,"summary":"Scorching"},{"date":"2019-12-03T18:13:28.0975584+01:00","temperatureC":4,"temperatureF":39,"summary":"Cool"},
{"date":"2019-12-04T18:13:28.0975588+01:00","temperatureC":-1,"temperatureF":31,"summary":"Mild"},
{"date":"2019-12-05T18:13:28.0975591+01:00","temperatureC":-14,"temperatureF":7,"summary":"Hot"}]

```

[](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#add-environment-variables-to-local-host)Add environment variables to local host
--------------------------------------------------------------------------------------------------------------------------------------------------------------

Weather data are interesting but let's try to output environment variables that will later prove that we're in a different host.

For localhost we'll use below variables:

```
export APPENVIRONMENT="development"
export APPHOST="local"

```

Stop the app and add a new controller that will output these variables. Note the routing attribute which is set to root path.

```
namespace app.Controllers
{
    [ApiController]
    [Route("")]
    public class InfoController : ControllerBase
    {
        private readonly ILogger<InfoController> _logger;

        public InfoController(ILogger<InfoController> logger)
        {
            _logger = logger;
        }

        [HttpGet]
        public InfoModel GetInfo()
        {
            return new InfoModel { AppEnvironment = GetEnvironmentVariable("APPENVIRONMENT"), AppHost = GetEnvironmentVariable("APPHOST") };
        }

        private string GetEnvironmentVariable(string name)
        {
            _logger.LogInformation($"Getting environment variable '{name}'.");
            return Environment.GetEnvironmentVariable(name.ToLower()) ?? Environment.GetEnvironmentVariable(name.ToUpper());
        }
    }
}

```

Run the app again and check the response on `localhost:{port}`.

```
{"appEnvironment":"development","appHost":"local"}

```

We have our web API ready to ship to another host. To do so, we need to create a docker image of it.

[](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#create-a-docker-image)Create a docker image
==========================================================================================================================

First step of containerization is to create a Dockerfile.

```
touch Dockerfile

```

with following content:

```
FROM mcr.microsoft.com/dotnet/core/sdk:3.0-alpine as build
WORKDIR /app

# copy csproj and restore
COPY app/*.csproj ./aspnetapp/
RUN cd ./aspnetapp/ && dotnet restore

# copy all files and build
COPY app/. ./aspnetapp/
WORKDIR /app/aspnetapp
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/aspnet:3.0-alpine as runtime
WORKDIR /app
COPY --from=build /app/aspnetapp/out ./
ENTRYPOINT [ "dotnet", "app.dll" ]

```

Having Dockerfile ready, we can build our image which will be named `aspnet3k8s` and tagged as `v1`. We need to navigate one level up from `app` directory and run following command.

```
docker image build --pull -t aspnet3k8s:v1 .

```

Having an image built we can run it.

```
docker run --rm -it -p 9000:80 aspnet3k8s:v1

```

Browse `localhost:9000` and you should see `null` values for both variables.

```
{"appEnvironment":null,"appHost":null}

```

That's a good sign. After all we're now in a different environment running on a docker host.

Stop the command and try passing new pair of environment variables for the new host.

```
docker run --env APPENVIRONMENT=production --env APPHOST=docker --rm -it -p 9000:80 aspnet3k8s:v1

```

and check the response again on `localhost:9000`

```
{"appEnvironment":"production","appHost":"docker"}

```

Good!

[](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#create-a-helms-chart)Create a Helm's Chart
=========================================================================================================================

Finally we've got to the meat of the matter. We have our ASP.NET app containerized and we can deploy it on a Kubernetes cluster. We won't do it directly but use a popular "package manager" meant for Kubernetes - Helm.

Remember, our goal is to create three instances of our ASP.NET Core 3 web API behind a 'load balancer' that will route to any of them.

Translating it to a Kubernetes nomenclature we're going to create a Deployment that will in turn create a ReplicaSet consisted of 3 pods (instances) and a Service which will serve as a 'load balancer' forwarding requests to any of them from outside world.

If you haven't done it already, it's high time for installing both Docker Desktop and Helm.

Before going further it might be good to read [Helm's three big concepts](https://v3.helm.sh/docs/intro/using_helm/).

### [](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#initialize-chart)Initialize Chart

We could do it using `helm create NAME` command but it would create a lot of files we don't need for the purposes of this guide.

Thus, let's create a following structure on a root folder.

```
chart/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml

```

First let's have a look at `Chart.yaml` which specifies name and version of the Chart. There are other fields you can add too, but only these are required.

```
name: aspnet3-demo
version: 1.0.0

```

### [](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#chart-values)Chart values

Then, there's `values.yaml`, which specifies our default settings that will be parsed and injected into templates inside `/templates` directory.

We define our environment name there. You can also see our image name `aspnet3k8s` and `replicas: 3` field which specifies how many instances of our web API we'd like to have.\
Please note I've added a field `pullPolicy: IfNotPresent` because our image is only on our local machine.

```
environment: development

apphost: k8s

label:
  name: aspnet3core

container:
  name: aspnet3
  pullPolicy: IfNotPresent
  image: aspnet3k8s
  tag: v1
  port: 80
replicas: 3

service:
  port: 8888
  type: ClusterIP

```

So far so good, but you might ask yourself how do these `values` YAMLs bring as closer to our goal? Answer is, these values are referenced in our templates. Let's examine them now.

### [](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#chart-templates)Chart templates

We'll start with `deployment.yaml`. This template will create a Deployment object that will in turn create ReplicaSet consisted of 3 pods each running our web API container image. Have a look at all references to values we defined in `values.yaml`. Note a Helm's built-in object `.Release` that we'll use for naming our deployment.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Values.label.name }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.label.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.label.name }}
        environment: {{ .Values.environment }}
    spec:
      containers:
        - name: {{ .Values.container.name }}
          image: {{ .Values.container.image }}:{{ .Values.container.tag }}
          imagePullPolicy: {{ .Values.container.pullPolicy }}
          ports:
            - containerPort: {{ .Values.container.port }}
          env:
            - name: apphost
              value: {{ .Values.apphost }}
            - name: appenvironment
              value: {{ .Values.environment}}

```

That will make sure 3 instances of our web API will be running on our local cluster. But without a 'load balancer' we won't access any of them. For that we need a Service which template is defined in `service.yaml`.

```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
  labels:
    app: {{ .Values.label.name }}
spec:
  ports:
  - port: {{ .Values.service.port}}
    protocol: TCP
    targetPort: {{ .Values.container.port }}
  selector:
    app: {{ .Values.label.name }}
  type: {{ .Values.service.type }}

```

We're all set and we can finally deploy our Chart on our local cluster!

[](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#the-grand-finale)The grand finale
================================================================================================================

Run the following command to install our Chart as a release named `aspnet3release`.

```
helm install aspnet3release ./chart/

```

You should see a successful response

```
NAME: aspnet3release
LAST DEPLOYED: Sun Dec  1 12:07:53 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

```

Let's see what exactly has been deployed on a Kubernetes cluster using `kubectl` command. Note that we're using a selector to match only those resources which have label `app=aspnet3core`.

```
kubectl get all --selector app=aspnet3core

```

Which should output following.

```
NAME                                            READY   STATUS    RESTARTS   AGE
pod/aspnet3release-deployment-77686884b-fq5lq   1/1     Running   0          3m43s
pod/aspnet3release-deployment-77686884b-llcsv   1/1     Running   0          3m43s
pod/aspnet3release-deployment-77686884b-qswsj   1/1     Running   0          3m43s

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/aspnet3release-service   ClusterIP   10.107.172.66   <none>        8888/TCP   3m43s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/aspnet3release-deployment   3/3     3            3           3m43s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/aspnet3release-deployment-77686884b   3         3         3       3m43s

```

What we have here. Three pods that runs our web API a ReplicaSet and a Service that listens on `8888` port. Great, let's browse then `localhost:8888` to see our API in action!

Nothing happens. Why? Because the Service's port is not on your local machine but inside the cluster. Thus, we have to forward a port on our local machine to Service's port using `kubectl port-forward` command. We could reuse 8888 port, but let's use 9999 instead to make a clear distinction what's where.

From the previous command `kubectl get all` we know that our service can be identified as `service/aspnet3release-service`. Let's try.

```
kubectl port-forward service/aspnet3release-service 9999:8888

```

Browse `localhost:9999` and... Success!

```
{"appEnvironment":"development","appHost":"k8s"}

```

### [](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#updating-our-deployment)Updating our deployment

You might think why I've used Helm instead of creating standard Kuberentes definitions for both Deployment and Service object. It's quite handy that some values are not copied by value but referenced but still - why make such a fuss of it?

Imagine we'd like to go live with our application. We could copy our development YAMLs and change the values here and there but that would be very error-prone and hard to maintain. Instead using Helm we can simply create a new set of values for production and store it in a new file `production-values.yaml`. Something like:

```
environment: production
replicas: 5

```

For simplicity we'll just upgrade our development using `helm upgrade` command and pass `production-values.yaml`.

```
helm upgrade aspnet3release ./chart --values ./chart/production-values.yaml

```

and the response:

```
Release "aspnet3release" has been upgraded. Happy Helming!
NAME: aspnet3release
LAST DEPLOYED: Sun Dec  1 12:40:00 2019
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None

```

Again, let's verify how it looks like on the cluster:

```
kubectl get all --selector app=aspnet3core

```

and you should see 5 instances of our app. That was easy, isn't it?

```
NAME                                            READY   STATUS    RESTARTS   AGE
pod/aspnet3release-deployment-774c478b4-64jgp   1/1     Running   0          12m
pod/aspnet3release-deployment-774c478b4-fx8bs   1/1     Running   0          12m
pod/aspnet3release-deployment-774c478b4-mkplm   1/1     Running   0          12m
pod/aspnet3release-deployment-774c478b4-mnwgb   1/1     Running   0          12m
pod/aspnet3release-deployment-774c478b4-xnd9b   1/1     Running   0          12m

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/aspnet3release-service   ClusterIP   10.107.172.66   <none>        8888/TCP   44m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/aspnet3release-deployment   5/5     5            5           44m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/aspnet3release-deployment-774c478b4   5         5         5       12m
replicaset.apps/aspnet3release-deployment-77686884b   0         0         0       44m

```

We've also changed the environment variable, let's verify that too. Again we have to port-forward.

```
kubectl port-forward service/aspnet3release-service 9999:8888

```

and browse `localhost:9999` to see the enivronment has been changed to `production`. Cool!

```
{"appEnvironment":"production","appHost":"k8s"}

```

That was quite a journey. We've gone through creating a new ASP.NET Core app, containerizing it, creating Helm's chart,deploying it on Kubernetes cluster and finally updating it.

Oh, almost forgot. To clean our cluster use the following command:

```
helm uninstall aspnet3release

```

[](https://dev.to/wolnikmarcin/run-asp-net-core-3-on-kubernetes-with-helm-1o01#credits)Credits
==============================================================================================

The idea to create this guide was inspired by a great book written by John Arundel and Justin Domingus: [Cloud Native DevOps with Kubernetes](http://shop.oreilly.com/product/0636920175131.do)

Thanks for reading. I hope you liked it.
