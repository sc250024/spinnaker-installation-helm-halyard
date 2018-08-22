# Spinnaker installation on Kubernetes using (new!) Halyard-based Helm chart

![Spinnaker](https://upload.wikimedia.org/wikipedia/commons/b/b1/C%26C_99_Sailboat_99095_Spinnaker_Run.jpg)

At [ParkBee](https://parkbee.com) we're experimenting with [Spinnaker](https://www.spinnaker.io/), Netflix's open-source continuous deployment (CD) solution. One aspect of Spinnaker became very clear from the beginning: just installing it can be a bit tricky. Most of the day-to-day work that DevOps / Infrastructure engineers do is related to linking things / systems together using tools available; with that said, we believe installing the tool shouldn't be the hard part of that process, and that most time should be spent on creating and improving pipelines for your development teams.

Spinnaker is different. Unlike most modern cloud-native apps that run as a single process (usually in a Go binary), Spinnaker is actually a composite application of individual microservices. The architecture diagram listed on the website reveals the complexity of this app: https://www.spinnaker.io/reference/architecture/

Luckily, there are great tools like [Helm](https://helm.sh/) available to us to make the installation, and release of new software on Kubernetes very simple. We started experimenting with the ["pure" YAML version](https://github.com/helm/charts/tree/9f790ac4644387f2856bdbc4ab5f44feb52ddeee/stable/spinnaker) of the Spinnaker chart only to realize that dealing with the litany of configuration options to simply get Spinnaker to run was overwhelming. [My post on the r/devops sub-Reddit](https://www.reddit.com/r/devops/comments/90nomd/productiongrade_spinnaker_using_helm/) revealed the following from one of the chart's owners, Vic Iglesias [(@viglesiasce)](https://github.com/viglesiasce):

> The chart was intended to be a Quick Start for an arbitrary cluster to get people familiar with Spinnaker. As the Spinnaker feature set has expanded it has become unfeasible to keep up with all the functionality by leveraging static/templated config files as the current chart does.
>
> We have been working on a new version of the chart which provides the same functionality as the original (ie helm install gets you a working Spinnaker in any cluster). The main difference in this iteration is that it provisions and uses Halyard for all the config under the hood.

Vic was nice enough to reference a PR he had been working on ([helm/charts#6407](https://github.com/helm/charts/pull/6407)) that utilizes Halyard instead of the pure, YAML based approach.

## Enter Halyard

This post isn't a deep dive into Halyard itself, but as a quick introduction, the [Halyard GitHub repo](https://github.com/spinnaker/halyard) states:

> [Halyard is] a tool for configuring, installing, and updating Spinnaker.

Lars Wander ([@lwander](https://github.com/lwander)), the other maintainer of the Spinnaker Helm chart, gives a great talk on YouTube ([Halyard Deep Dive](https://www.youtube.com/watch?v=6oHRe3-zflo)) describing why Halyard is necessary for a large tool like Spinnaker. Essentially it boils down to this: *Halyard helps you spend less time on configuring Spinnaker for your cloud environment, and more time on actually using the product for your CD needs.*

## How the new Halyard-based chart works

Most Helm charts are essentially YAML with some Mustache templating built in to make the generation of Kubernetes YAML manifests easier. Rather than deploy YAML configurations directly, the new Halyard-based Helm chart deploys the following on installation:

* A `Job` pod which runs the initial installation.
* A `StatefulSet` pod which runs the actual Halyard daemon (used by the aforementioned pod on initial installation), and is used by a cluster administrator to administer Spinnaker.

Other pods that are installed will depend based on your own needs. For example, upon the initial installation, a Redis pod or Minio pod might be spun up as well if you choose to use those services.

## Installation

Now let's get to the installation. Once this PR ([helm/charts#6407](https://github.com/helm/charts/pull/6407)) is merged the new chart will be available in the `stable` channel from the default Kubernetes chart repository. If it's not merged, you can still use the chart, but you should do a `git clone` of Vic's repository here: [https://github.com/viglesiasce/charts](https://github.com/viglesiasce/charts). The new Halyard-based chart is located in the [spin-v1.0.0](https://github.com/viglesiasce/charts/tree/spin-v1.0.0) branch. Anytime you see `stable/spinnaker`, you can just directly point to the Git repository on your local computer, e.g. `/some/path/to/viglesiasce-charts/stable/spinnaker`.

This article assumes you already have the following set up:

* Kubernetes cluster with enough resources. **NOTE**: This probably should not be run locally, although you're free to try; just keep in mind that this is a large deployment of around 10 pods.
* Helm
* Ingress controller

As with most Helm chart, you'll need a `values.yaml` file to populate with your own settings. Here's an example which will get you set up with the following:

* DockerHub
* Ingress with TLS support
* Kubernetes integration
* S3 bucket persistence
* Service account for re-deploying Spinnaker via the `StatefulSet` pod

```yaml
ingress:
  enabled: true
  host: "spinnaker.mycompany.com"
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "spinnaker.mycompany.com"
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ingress.class: "nginx"
  tls:
   - secretName: my-company-com-tls
     hosts:
       - spinnaker.mycompany.com

dockerRegistries:
- address: "index.docker.io"
  email: "mydockeraccount@mycompany.com"
  name: "company-dockerhub"
  password: "<dockerhub-password>"
  username: "<dockerhub-username>"
  repositories:
    - company/dockerrepo1
    - company/dockerrepo2
    - personal/dockerrepo3

kubeConfig:
  enabled: false
  contexts:
  - my-kubernetes-cluster
  deploymentContext: my-kubernetes-cluster

rbac:
  create: true

minio:
  enabled: false

s3:
  accessKey: "<aws-access-key>"
  bucket: "<aws-spinnaker-bucket>"
  enabled: true
  region: "<aws-region>"
  secretKey: "<aws-secret-key>"

serviceAccount:
  create: true

spinnakerFeatureFlags:
  - artifacts
  - infrastructure-stages
  - jobs
  - pipeline-templates
```

If you're not using AWS, and just want to try things out, you can set `s3.enabled` to `false` and `minio.enabled` to `true` to enable local persistence using Minio.

First create a dedicated namespace for Spinnaker:

```shell
$ kubectl create ns spinnaker
```

Then run the following Helm command to install:

```shell
$ helm upgrade \
--install \
--namespace spinnaker \
--timeout 1200 \
--values example-values.yaml \
--wait \
spinnaker \
stable/spinnaker
```

The long timeout period is needed for this deployment as it takes a while for Halyard to configure everything, and then for the pods to spin up.

You will see the initial pods being created like so:

![Initial installation pods](https://raw.githubusercontent.com/sc250024/spinnaker-installation-helm-halyard/master/images/spinnaker-initial-install-pods.png)

Once the pod with prefix `spinnaker-install-using-hal*` is running, you can monitor the installation using:

```shell
$ kubectl logs -f spinnaker-install-using-hal-x2gk9
```

After some time, you will see that all of the new pods created by Halyard (with a base name of `spin`) have been created:

![Installation finished](https://raw.githubusercontent.com/sc250024/spinnaker-installation-helm-halyard/master/images/spinnaker-install-finished.png)

Go to the Ingress URL created, and you should something like the following:

![Installation finished](https://raw.githubusercontent.com/sc250024/spinnaker-installation-helm-halyard/master/images/spinnaker-post-install-screen.png)

Since the Kubernetes provider is installed by default, you should see the **spinnaker** and **spin** applications created by the Helm installer.

## Additional configuration

At this point, assuming that the installation went well, there's still some things to configure. I'll use the new `StatefulSet` pod create to demonstrate how to make changes to your Spinnaker deployment from inside the pod.

Start a shell session inside the pod by running the following command:

```shell
$ kubectl exec -it spinnaker-spinnaker-halyard-0 bash
```

The following sections will assume you're inside the shell of this StatefulSet container.

### Change timezone

By default, Spinnaker uses `America/Los_Angeles` as the default time zone. To change that, run the following command:

```shell
$ hal config edit --timezone "Etc/UTC"
```

And then deploy:

```shell
$ hal deploy apply
```

### Cloud provider

Since Spinnaker can create cloud resources for you as part of your deployments, let's configure Spinnaker for AWS.

The pod already has your AWS credentials mounted at `/opt/s3/accessKey` and `/opt/s3/secretKey`. Configure your Spinnaker account ([instructions on how to do that here](https://www.spinnaker.io/setup/install/providers/aws/)) using the following command:

```shell
$ cat /opt/s3/secretKey | hal config provider aws edit \
  --access-key-id $(cat /opt/s3/accessKey) \
  --secret-access-key
```

Enable Spinnaker to assume the IAM role you've created:

```shell
hal config provider aws account add company-aws \
  --account-id ZZZXXXXXXXXX \
  --assume-role role/spinnakerManaged
```

Configure your region(s):

```shell
hal config provider aws account edit company-aws \
  --regions eu-west-1
```

Enable the AWS provider:

```shell
$ hal config provider aws enable
```

And finally deploy:

```shell
$ hal deploy apply
```

### Enable GitHub OAuth login

Out of the box, Spinnaker does not come with a default authentication mechanism, and is wide open to anyone with the URL. Our preferred method was to have Spinnaker behind *one* ingress resource that handled both the UI (Deck), and API (Gate) services; this turned out to be more difficult than we thought, and currently, we couldn't get the Deck and Gate services to work behind one ingress resource.

Many example guides for Spinnaker, including [How to deploy Spinnaker on Kubernetes: a quick and dirty guide](https://www.mirantis.com/blog/how-to-deploy-spinnaker-on-kubernetes-a-quick-and-dirty-guide/), solve the problem of by exposing ports directly; we thought though it's much nicer though to point to https://spinnaker.mycompany.com rather than http://52.XX.XX.XXX:9000 and http://52.XX.XX.XXX:8084.

Spinnaker's own post ([Try out public Spinnaker on GKE](https://www.spinnaker.io/setup/quickstart/halyard-gke-public/)) reveals the solution that's recommended: create two ingress resources for Deck and Gate separately. For this example, we'll use the following URLs:

* spinnaker.mycompany.com => Deck
* spinnaker-api.mycompany.com => Gate

#### Client tokens

We'll use GitHub to configure OAuth, but first we need client tokens.

1. Go to https://github.com/settings/developers and click **New OAuth App**.
2. Under **Application name**, type *Spinnaker*.
3. Under **Homepage URL**, type *https://spinnaker.mycompany.com*.
4. Under **Authorization callback URL**, type *https://spinnaker-api.mycompany.com/login*
5. Click **Register application**.
6. Make note of the generated **Client ID** and **Client Secret** given by GitHub.

#### Edit ingress resource

We'll need to make a change to the ingress resource to create an endpoint for the Gate service, which handles OAuth.

Edit the ingress resource:

```shell
$ kubectl edit ingresses.extensions -n spinnaker -o=jsonpath='{.items[?(@.metadata.labels.release=="spinnaker")].metadata.name}'
```

Under `spec.rules`, ensure that both of the `host` entries are present:

```yaml
spec:
  rules:
  - host: spinnaker.mycompany.com
    http:
      paths:
      - backend:
          serviceName: spin-deck
          servicePort: 9000
        path: /
  - host: spinnaker-api.mycompany.com
    http:
      paths:
      - backend:
          serviceName: spin-gate
          servicePort: 8084
        path: /
```

And if you've configured TLS certificates, ensure that those entries are present also:

```yaml
spec:
(...)
  tls:
  - hosts:
    - spinnaker.mycompany.com
    secretName: my-company-com-tls
  - hosts:
    - spinnaker-api.mycompany.com
    secretName: my-company-com-tls
```

#### Apply Hal config

On the Hal `StatefulSet` container, first export the OAuth information as variables:

```shell
$ export CLIENT_ID="<github-client-id>"
$ export CLIENT_SECRET="<github-client-secret>"
$ export OAUTH_PROVIDER="github"
$ export SPINNAKER_API_BASE_URL="https://spinnaker-api.mycompany.com/"
$ export SPINNAKER_REDIRECT_URI="https://spinnaker-api.mycompany.com/login"
$ export SPINNAKER_UI_BASE_URL="https://spinnaker.mycompany.com/"
```

Configure the GitHub OAuth provider:

```shell
$ hal config security authn oauth2 edit \
  --client-id ${CLIENT_ID} \
  --client-secret ${CLIENT_SECRET} \
  --provider ${OAUTH_PROVIDER}
```

Update Spinnaker with the new URLs:

```shell
$ hal config security authn oauth2 edit --pre-established-redirect-uri ${SPINNAKER_REDIRECT_URI}
$ hal config security ui edit --override-base-url ${SPINNAKER_UI_BASE_URL}
$ hal config security api edit --override-base-url ${SPINNAKER_API_BASE_URL}
```

Enable OAuth login:

```shell
$ hal config security authn oauth2 enable
```

And finally deploy:

```shell
$ hal deploy apply
```

Ensure your environment varibles aren't persisted in the BASH history by exiting using this command:

```shell
$ kill -9 $$
```

### Additional config on initial installation

With the new chart, you can run all of the commands above without having to login to the Halyard pod. In the `values.yaml` file, there's a section under `halyard.additionalConfig` which enables you to specify a Kubernetes `ConfigMap` with your additional commands:

```yaml
halyard:
  spinnakerVersion: 1.8.5
  image:
    repository: gcr.io/spinnaker-marketplace/halyard
    tag: stable
  additionalConfig:
    enabled: false
    configMapName: my-halyard-config
    configMapKey: config.sh
```

## Teardown

After running through the demo, tear-down the installation:

```shell
$ helm delete --purge spinnaker
$ kubectl delete ns spinnaker --grace-period=0
```

## Conclusion

Spinnaker is a well-supported open-source product, and hopefully with this guide, you can evaluate and get it running for your own CD needs. ⎈ Happy Helming! ⎈

I work for Parkbee. We develop smart tech. Our Mobility Management Solution optimizes the use of underutilized parking space to get cars off the street and [KEEP YOUR CITY MOVING](https://keepyourcitymoving.com/).

If you're in the Netherlands or the UK, check out [Parkbee's website](https://parkbee.com/) (we're hiring!)

![ParkBee - KEEP YOUR CITY MOVING](https://keepyourcitymoving.com/wp-content/themes/kycm/build/img/logo/logo-parkbee@2x.png)
