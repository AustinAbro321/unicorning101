# Unicorning 101 

You are going to create a new Hello World project using all the concepts you learned in sparkle academy and more! 

## Table of contents 
- Create a Hello World application, any language
- Containerize that application
- harden that container and make a multi-stage build if applicable
- Publish your image to a container registry
- k8ify that application to make it run in a k3d cluster
- add isitio to the application so it's being sent through by a virtual service
- wrap our application in a helm chart
- zarfify your deployment so it works easily in the air gap

# Hello world app

Before we can run our application in Kubernetes we need an application to run. We are going to create a new application for this exercise. It can be in any language you want! Your goal is to run an application locally and have it start. You have many options for languages
- go https://go.dev/doc/articles/wiki/ 
  - Efficient language often used for tools in the Kubernetes landscape, like zarf(link)
- javascript (node) https://nodejs.org/en/docs/guides/getting-started-guide/
  - Common language in the industry. Pepr is written in typescript (typed javascript)
- java spring https://spring.io/guides/gs/spring-boot/
  - You'll run across lots of apps in the government using Java. 3 billion apps use Java or something like that

You can choose any language for this. A quick Google search of Hello World X language will bring you one of the many tutorials. You know you are finished with this when you can go to localhost:*port-number* and see "hello world" or whatever text you decided to display.

# Containerization

Now we are going to run our application within a container. If you've gone through sparkle academy you've already built an [nginx container](https://360.articulate.com/review/content/9483167d-cad3-4747-8899-897c39e9378b/review). Now we are going to build a container with your application. 

Usually, this is pretty straightforward. As an example, below we are using rust. The general flow of all the docker files should be more or less the same. 

- Start from an image that has the language you need
- Copy the source code into the container 
- Build the application 
- Run the application

P.S. I am using the latest tag here. It's okay to do so as well for now, but it should be known that using "latest" should never happen in production. This article gives a pretty good summary https://vsupalov.com/docker-latest-tag/. TLDR makes it hard to know what's actually running in production, which makes it easy to override something that shouldn't be overwritten and hard to track security vulnerabilities like CVEs.

```dockerfile
FROM rust:latest
WORKDIR /usr/src/hello-world
COPY . .
RUN cargo build --release
CMD ["./target/release/hello-world"]
```

## Assignment: 
Create a docker image for your application. Run this docker image. You are finished with this exercise when you can pull up your application running as a docker container on localhost. 

*Hint* you will need to port forward. Google searching is your friend  


# Hardening containers 

## Multi stage builds

While the above dockerfile may seem pretty slim there is a ton that we don't need within this container. Rust compiles into an executable, so we don't need any of the Rust binaries in the container after it's built. Additionally, the rust default image from dockerhub comes with a bunch of additional tools pre-installed for our convenience. That convenience comes at the price of security. At Defense Unicorns we work with the government so avoiding CVEs is often a top priority. curl is a good example of an often included tool that isn't needed but is there for convenience. curl is useful in some situations as you may want to pull down additional dependencies from the internet. curl and tools like it often have CVEs, whether or not, these CVEs are exploitable in the context of your container is a complex question to answer. Luckily, we often don't need tools like curl in our final image. If we do need extra tools, as long as they're not needed at runtime we can still avoid having them in our final image.

Introducing multi-stage builds. Read this page for a quick overview: https://docs.docker.com/build/building/multi-stage/. multi-stage builds allow you to have separate stages of a dockerfile, each stage can stem from a different base image. Files can be copied from an earlier image to a later image. Often the first stage(s) are used to build/pull the binaries needed to run the application, while the later stages use a smaller image and run the binary. This would allow us to start with an image with curl, download a package then copy that package into our final stage without needing the curl cli in our final stage. If we have a statically typed language like go, rust, java, etc we can build our application and run it in the final stage without needing an image with the language installed.

Below is an example of avoiding having the language in the final image by utilizing multi-stage builds. In addition, we've made some changes to make this dockerfile closer to what you might see in a real repository building production images.

Note:
- This is now a multi-stage build. In the first "builder" stage we start from a rust image and use rust tools to compile our application. In the second stage, we copy our built binary over to Alpine and run it 
- Both images are now using Alpine as a base OS. This is a much smaller OS with much less pre-installed on it. Generally has very few CVEs. Since the OS is smaller there is an additional tool that we need to bring. You can see we need to install musl-dev. In a less slim OS tools like this are pre-installed. In Alpine, we more often explicitly install what we need
- No longer using the latest tag now we are pinning to a specific version. This gives us the benefit that we know our app is not going to randomly take on a new version of Rust, potentially with breaking changes or CVEs. 
- We now only copy the files we need rather than all files. Small change but it is more explicit and readable

```dockerfile
FROM rust:1.73.0-Alpine3.18 as builder
WORKDIR /app
COPY src/ src/
COPY Cargo.lock .
COPY Cargo.toml .
# This package is needed to compile our rust binary 
RUN apk add musl-dev
RUN cargo build --release

FROM Alpine:3.18.4
WORKDIR /app
COPY --from=builder /app/target/release/hello-world /app/hello-world
# Rocket.toml is only needed at run time so add it here
COPY Rocket.toml . 
CMD ["./hello-world"]
```

## Hardening non-compiled applications 

But wait?! If I'm using Nodejs or Python or \[insert non-statically compiled language\] I won't receive the benefits statically compiled languages do. You're correct! Well mostly, there may be situations where there are build-time dependencies that aren't needed at runtime, in this situation we may still consider a multi-stage build. We are creating pretty simple hello world applications though so we aren't likely to run into this case. Still, we want to reduce the surface area of vulnerabilities possible on our container. To do this we want the smallest image possible that has the dependencies we need to build an application. A common distro to use is Alpine a Linux distribution designed to be small, simple, and secure.

In the above example, you can see we switched to Alpine. If you search dockerhub, you should be able to find a base image for your language that is built on top of Alpine

An added benefit of going to multi-stage dockerfiles and smaller OSs for your images can result in huge reductions in image size

When I run `docker images non_hardened_image`, I get 
```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
cargo        latest    22599959abb4   19 hours ago   2.73GB
```

Alternatively when I run `docker images hardened_image` I get 
```
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
cargo-multi   latest    ee92ce7340e8   19 hours ago   17.4MB
```

That's a reduction in size from 2.73GB to 17.4MB. An over 150 times reduction! 

## Assignment 

If the language you choose is statically compiled make a multi-stage build. Whether or not you have a multi stage build switch to using Alpine.

# Container registries

Before we can deploy our image we will need to store it somewhere. The best place to store images is in a container registry. https://360.articulate.com/review/content/9483167d-cad3-4747-8899-897c39e9378b/review - take a look here for a refresher. Often at Defense Unicorns, you will see us use images from Ironbank, a dod run container registry. However, to keep things simple we are going to just publish our image to dockerhub. 

## Assignment 
Publish your image to dockerhub. You will have to make an account if you don't have one already


# Kubernetes deployment

Now it's finally time to deploy our application in Kubernetes. Things are going to amp up this lesson, it's a huge step. Make sure you are ready to google search, and if you are unfamiliar with Kubernetes make sure you've completed this section of the review https://360.articulate.com/review/content/41f8c202-c1c2-4dc9-914b-f03388b3f776/review.

First, we are going to [install k3d](https://k3d.io/v5.6.0/#releases). This is a tool we use often at defense unicorns for local testing. It creates a small k3s cluster within docker. If that's confusing don't worry, while k3d is generally not suitable for production it's a great way to easily set up a cluster to deploy our code. 

## Assignment 
Create a deployment and service for your application. You know you are done when your app is running as a deployment in Kubernetes and you can view it in your browser.

Getting the application to show up on your browser from k3d can be tricky. We recommend following this guide after your service is configured. https://k3d.io/v5.4.6/usage/exposing_services/

# Isitio integration - virtual services
 
Next, we are going to install and use isitio. Isitio is a complex topic, we encourage you to read this https://istio.io/latest/about/service-mesh/. It will be confusing at first, don't falter! Reading through and trying to understand will help the information digest easier in the future.

We are going to start with one of the simpler and more widely used features of isitio, virtual services. A virtual service lets us define a host in Kubernetes. It then lets us define complex routing rules for that host. This shifts DNS routing rules much closer to the cluster rather than to a cloud service provider. With virtual services, you only need a load balancer to point to the cluster. Once the load balancer points to the cluster, when the cluster receives a request it uses virtual services to decide where that request goes. Further reading on virtual services: https://istio.io/latest/docs/reference/config/networking/virtual-service/

Let's take a look at the below virtual service. In this case, we have two microservices but we want them both to resolve at specific prefixes for bookclub-example.com. Using a virtual service we can assign routing rules to do so. 

Let's look at an example with one host and two routing rules
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: example
spec:
  hosts:
  - "bookclub-example.com"
  gateways:
    - "istio-system/main"
  http:
  - match:
    - uri:
        prefix: /authors # <- This means when someone goes to bookclub-example.com/authors they will get the below destination
    route:
    - destination:
        host: author-mircroservice # <- This is the service that we are forwarded to when someone goes to bookclub-example.com/authors. It can be the same type of service that you made earlier. 
        port:
          number: 8080
  - match:
    - uri:
        prefix: /books # < This means when someone goes to bookclub-example.com/books they will get the below destination. 
    route:
    - destination:
        host: book-mircoservice # <- This is the service that we are forwarded to when someone goes to bookclub-example.com/books. It can be the same type of service that you made earlier. 
        port:
          number: 8080
```

## Assignment 
Make a virtual service for your application. We are going to cheat somewhat to get it to work in k3d with a real domain.

Usually, it is hard to get isitio to work as intended with real URLs on a local cluster. Locally we don't have DNS services or a load balancer that points to our cluster. Without this we can't send reqeusts to our cluster that istio can parse through.  

To get around this problem for testing purposes Defense Unciorns created a cert for *.bigbang.dev. *.bigbang.dev is a [CNAME](https://www.cloudflare.com/learning/dns/dns-records/dns-cname-record/) that points back to localhost. We will also set up our k3d cluster to forward requests from localhost 443 and 80 back to the cluster. By combining all of these steps, we can go to a website, for example, rust-hello-world.bigbang.dev and have it point to localhost. Because this is complex and uses an existing solution below is a detailed guide on using the *.bigbang.dev cert and domain in your cluster. 

### Create k3d cluster
You likely have an existing k3d cluster from the previous assignment. However, this k3d cluster will need to be created with different arguments for isitio to work. Delete your previous cluster if necessary
```bash
k3d cluster create --api-port 6445 --k3s-arg "--disable=traefik@server:0" --port 80:80@loadbalancer --port 443:443@loadbalancer
``` 

Disabling traefik is important because trafeik comes by default inside k3 and therefore k3d. There is overlap between isitio and traefik so there will be conflicts if both exist. 

### Install isitio
Next, we are going to install the isitio command line tool *istioctl*. Following instructions here: https://istio.io/latest/docs/setup/getting-started/#download

While we now have the command line tool on our local machine we still do not have istio installed in the cluster. In our case, because we are not in a production environment we are going to use the istio command line tool tool to install isitio onto our cluster. Usually, in a defense unicorn environment, isitio would be installed through big bang or DUBBD. We will cover this in a later section. For now, install istio simply with

`istioctl install -y`

### Create isitio gateway
Next, we need to install a istio gateway. A gateway is a custom resource definition created by istio. We know this because in the apiversion we see networking.**istio**.io/v1alpha3. When a request comes into the cluster it's first going to go to the gateway. From there the gateway will delegate the requests to the virtual services. In our case the gateway is only going to delegate if the request is on port 80 or 443 and the host of the request falls under *.bigbang.dev. For example rust-hello-world.bigbang.dev

Apply this yaml file

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: "main"
  namespace: istio-system
spec: 
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
      - "*.bigbang.dev"
    tls:
      httpsRedirect: true 
  - hosts:
    - '*.bigbang.dev'
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      credentialName: public-cert
      mode: SIMPLE
```

### Create public-cert secret

Notice the field for the https server above tls.credentialName. This is set to public-cert. It is expecting a Kubernetes secret called public cert with the fields tls.crt and tls.key set within the secret. In a production environment the public cert secret values would be dangerous to share, but because *.bigbang.dev is just for testing and points back to localhost anyway we post these certs online. To fill out the value for the below yaml. Get the value for 
- the tls.cert here https://github.com/defenseunicorns/uds-package-dubbd/blob/main/defense-unicorns-distro/bigbang.dev.cert
- the tls.key here https://github.com/defenseunicorns/uds-package-dubbd/blob/main/defense-unicorns-distro/bigbang.dev.key

Then you will need to base64 the values. The easiest way is generally to use the base64 tool in your terminal, you can also find websites to do it for you. This is fine because these aren't real secrets, though you should generally be wary of putting secrets online

Example of using base64 tool 
```bash
echo "cert_or_key_value" | base64
```

Fill in the tls.crt and tls.key with the respective values then apply the yaml file. 

```yaml
apiVersion: v1
data:
  tls.crt: 
  tls.key: 
kind: Secret
metadata:
  name: public-cert
  namespace: istio-system
type: Kubernetes.io/tls
```

### Create your own virtual service

Now that you've applied the gateway and the secret you are finally ready to make your virtual service. This will hopefully be simple now that the complex pieces are knocked out. You know you are finished with this when you can go to your_site_name.bigbang.dev and see your Hello World site

# helm charts

Now we are going to make our helm chart. If you do not already have Helm installed / a baseline understanding of helm go through the exercises here: https://360.articulate.com/review/content/54116b61-7866-494d-acc2-c689ccc26004/review

First, we are going to go to the root of our repository and run
```bash
helm create {your_app_name}
```

This will create a default Nginx helm chart. There's a lot here so let's start by breaking down the Deployment. Check the comments below to get a better understanding of what Helm is doing. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # The name of the deployment is templated using "rust-hello-world.fullname" to 
  # provide flexibility for the user to name the deployment as they prefer.
  # Whenever you see include, you can find how the value is defined in the _helpers.tpl file
  # _helpers.tpl generally holds values that have logic to be used throughout the chart that we don't want 
  # to duplicate in every file. 
  name: {{ include "rust-hello-world.fullname" . }}
  labels:
    # This section allows users to add their custom labels. The "rust-hello-world.labels" 
    # function in _helpers.tpl allows this. The labels are then correctly idented
    {{- include "rust-hello-world.labels" . | nindent 4 }}

spec:
  # The replicas field is set based on the "replicaCount" value from the values file, 
  # but only if autoscaling is not enabled. If autoscaling is enabled, this field will 
  # be omitted from the final YAML.
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}

  selector:
    matchLabels:
      # This will include the labels for the deployment's pod selector.
      # Notice how this matches the template labels below. This is because
      # we always want these two values to be the same for our deployment. Another advantage of helm
      # is the limited duplication
      {{- include "rust-hello-world.selectorLabels" . | nindent 6 }}

  template:
    metadata:
      # If "podAnnotations" are specified in the values file, they will be added 
      # to the pod metadata as annotations.
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      labels:
        # These are the labels that will be assigned to the pods created by this deployment.
        {{- include "rust-hello-world.selectorLabels" . | nindent 8 }}

    spec:
      # If "imagePullSecrets" are specified in the values file, they will be added here 
      # to allow the pods to pull images from private container registries.
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "rust-hello-world.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}

      containers:
        # Sets this to the chart name. The container name is just for readability
        - name: {{ .Chart.Name }}

          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          # The image is put together by putting the image.repository and image.tag field from the values file
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            # Resource requests and limits for the container, as defined in the values file.
            {{- toYaml .Values.resources | nindent 12 }}

      # Node selector allows you to specify on which nodes the pods should be scheduled.
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      # Affinity settings provide more advanced rules on pod placement.
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      # Tolerations enable the pods to be scheduled onto nodes with matching taints.
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

```

There is a lot here. Generally, you will see most of these fields specified on every chart you see in the wild because we always want users to be able to label, name, set tolerations, set nodes, etc on their instance of the chart. Situations will come up where if they can't do this the chart becomes unusable for their environment. For example, they already might have a different deployment with the same name, or they might only be able to run this deployment on a certain node and therefore need to set tolerations.

Beyond just the deployment there is a lot in the chart.

## Assignment

This assignment is broken into two parts

Part 1 - Get the chart working. Doesn't have to be as clean as you would want it. You should not include the cert or isitio gateway your chart. They should be kept logically separate from the app. You know you are completed with this section when your app is accessible as it was in the previous section and was stood up using *helm install*.

Part 2 - We are going to simply the chart. Delete the hpa.yaml, ingress.yaml, and serviceaccount.yaml. Generally, these files will be included in a helm chart you create, but we're going to delete them in our Hello World app for simplicity. It also gives us a good opportunity to practice editing our chart. Make sure you also delete the service account, autoscaling, and ingress fields where they exist throughout the chart. You will encounter errors if you don't fully delete all the references to the values of these files. This is to be expected, the helm cli should guide you toward what you need to do to fix it. Again you are completed with this section when your app runs from a successful helm install/update

## Zarf 

Finally, we get to the moment we've all been waiting for. Zarf! 

Zarf is a free open-source tool that enables continuous software delivery on disconnected networks. We, Defense Unicorns, created and now maintain zarf.

First, [install zarf](https://zarf.dev/install)

## Assignment 
Package your helm chart so that it runs as a zarf package. You know you are completed with this section once your app is successfully running after being deployed as a zarf package. 

This section is intentially left brief. Use the zarf docs and examples in the docs to guide you on creating a solution. If the docs are insufficient please give us feedback to improve! 