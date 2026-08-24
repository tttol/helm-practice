# Local Helm Learning Plan with Minikube

This plan teaches Helm entirely on a local machine. It does not require AWS, Amazon EKS, AWS Blocks, Java, or any cloud resources.

ref: https://zenn.dev/tttol/scraps/59a91d342cb04e

## Learning goal

By the end, you should be able to:

- Explain the difference between a Helm chart and a Helm release.
- Create and structure a Helm chart.
- Use `values.yaml` and environment-specific values files.
- Write Helm templates for Kubernetes resources.
- Install, upgrade, inspect, roll back, test, package, and uninstall releases.
- Debug failed template rendering and Kubernetes deployments.

Helm requires a Kubernetes cluster for operations such as `install` and `upgrade`. Chart authoring commands such as `lint` and `template` can be practiced without installing anything into a cluster. See the [Helm Quickstart](https://helm.sh/docs/intro/quickstart/).

## Local architecture

```text
Docker Desktop
      |
      v
Minikube local Kubernetes cluster
      |
      +-- kubectl: inspect Kubernetes resources
      |
      +-- Helm: package and manage Kubernetes applications
```

Minikube is designed for local Kubernetes learning and development. See the [official Minikube guide](https://minikube.sigs.k8s.io/docs/start/).

## Scope boundary

This plan focuses on Helm and the minimum Kubernetes knowledge needed to use Helm effectively.

It intentionally excludes:

- AWS accounts and AWS credentials
- Amazon EKS
- AWS Blocks
- `aws`, `eksctl`, and AWS CDK
- Java or application development
- Production Kubernetes operations

All Kubernetes resources created by this plan run inside Minikube on your computer.

## Phase 1: Install the local tools

This plan assumes macOS and Homebrew. Docker Desktop must be installed and running before Minikube starts.

```bash
brew install minikube kubectl helm
brew install --cask docker
```

Start Docker Desktop and verify the tools:

```bash
docker version
minikube version
kubectl version --client
helm version
```

Use the [official Helm installation guide](https://helm.sh/docs/intro/install/) if you are not using Homebrew.

## Phase 2: Create a Minikube cluster

Start Minikube with the Docker driver:

```bash
minikube start --driver=docker
```

Verify the cluster:

```bash
minikube status
kubectl config use-context minikube
kubectl get nodes
kubectl get pods --all-namespaces
```

Optional dashboard:

```bash
minikube dashboard
```

Create a namespace for the exercises:

```bash
kubectl create namespace helm-lab
kubectl config set-context --current --namespace=helm-lab
```

### Success criteria

You should see one Minikube node in the `Ready` state, and `kubectl` should communicate with it successfully.

## Phase 3: Learn the Kubernetes minimum

Helm creates and manages Kubernetes resources, so learn the purpose of these objects:

- Pod: the basic unit that runs containers.
- Deployment: maintains the desired number of Pods and performs rollouts.
- Service: provides stable network access to Pods.
- ConfigMap: stores non-sensitive configuration.
- Secret: stores sensitive-looking configuration values.
- Namespace: isolates resources within a cluster.
- Labels and selectors: connect related resources.

Create a small deployment manually so that you can recognize what Helm will automate:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
kubectl get pods
kubectl get services
kubectl get all
```

Access it locally:

```bash
kubectl port-forward service/nginx 8080:80
```

Open `http://localhost:8080` in a browser.

Delete the manually created resources:

```bash
kubectl delete deployment nginx
kubectl delete service nginx
```

### Success criteria

You should be able to explain which Kubernetes resource runs the container and which resource provides network access to it.

## Phase 4: Create and inspect your first chart

Generate a starter chart:

```bash
helm create hello-chart
```

Inspect the directory:

```text
hello-chart/
├── Chart.yaml
├── values.yaml
├── templates/
└── charts/
```

Important files:

- `Chart.yaml`: chart metadata and dependencies.
- `values.yaml`: default configuration values.
- `templates/`: Kubernetes manifests containing Helm templates.
- `templates/_helpers.tpl`: reusable template helpers.
- `templates/NOTES.txt`: post-install instructions.
- `templates/tests/`: chart tests.

Validate the chart without installing it:

```bash
helm lint ./hello-chart
helm template hello ./hello-chart
```

Render with debug information:

```bash
helm template hello ./hello-chart --debug
```

The [Helm Chart Template Developer's Guide](https://helm.sh/docs/chart_template_guide/) explains how templates generate Kubernetes YAML manifests.

### Success criteria

You should be able to identify which generated template creates the Deployment and which creates the Service.

## Phase 5: Install and inspect a Helm release

Install the chart into the learning namespace:

```bash
helm install hello ./hello-chart \
  --namespace helm-lab \
  --create-namespace
```

Inspect the release and its Kubernetes resources:

```bash
helm list --namespace helm-lab
helm status hello --namespace helm-lab
helm get manifest hello --namespace helm-lab
kubectl get all
```

Find the generated Service:

```bash
kubectl get services
```

Forward the Service port using the displayed Service name:

```bash
kubectl port-forward service/<service-name> 8080:80
```

Replace `<service-name>` with the actual name shown by `kubectl get services`.

### Concepts to learn

- A chart is a package of templates and metadata.
- A release is one installed instance of a chart.
- One chart can produce multiple releases.
- Helm stores release history so that releases can be upgraded and rolled back.

## Phase 6: Practice values and configuration

Open `hello-chart/values.yaml` and identify values for:

- Replica count
- Container image repository and tag
- Service type and port
- Resource requests and limits
- Liveness and readiness probes

Render different values without editing the file:

```bash
helm template hello ./hello-chart \
  --namespace helm-lab \
  --set replicaCount=2
```

Upgrade the running release:

```bash
helm upgrade hello ./hello-chart \
  --namespace helm-lab \
  --set replicaCount=2
```

Verify the result:

```bash
kubectl get deployment
kubectl get pods
```

Create a development values file:

```text
hello-chart/
├── values.yaml
├── values-dev.yaml
└── templates/
```

Use it during an upgrade:

```bash
helm upgrade hello ./hello-chart \
  --namespace helm-lab \
  --values ./hello-chart/values-dev.yaml
```

### Exercises

1. Change the replica count from one to two.
2. Change the container image tag.
3. Change the Service port.
4. Add an environment variable.
5. Add a resource request and limit.
6. Render the chart after every change and inspect the generated YAML.

### Success criteria

You should understand the difference between changing a template and changing a value.

## Phase 7: Learn the Helm release lifecycle

Inspect the current release:

```bash
helm list --namespace helm-lab
helm status hello --namespace helm-lab
helm history hello --namespace helm-lab
```

Create a new revision:

```bash
helm upgrade hello ./hello-chart \
  --namespace helm-lab \
  --set replicaCount=3
```

Review the revision history:

```bash
helm history hello --namespace helm-lab
```

Roll back to the first revision:

```bash
helm rollback hello 1 \
  --namespace helm-lab
```

Inspect the result:

```bash
helm status hello --namespace helm-lab
kubectl get deployment
kubectl get pods
```

### Advanced upgrade pattern

Later, practice the idempotent install-or-upgrade pattern:

```bash
helm upgrade --install hello ./hello-chart \
  --namespace helm-lab \
  --create-namespace
```

For safer automated upgrades, experiment with `--atomic` and a timeout after you understand normal upgrades:

```bash
helm upgrade --install hello ./hello-chart \
  --namespace helm-lab \
  --atomic \
  --timeout 2m
```

## Phase 8: Learn Helm templating

Study and practice these concepts:

- `.Values`: user-provided chart configuration.
- `.Release`: information about the current release.
- `.Chart`: chart metadata.
- `if`: conditional resources or fields.
- `with`: change the current object context.
- `range`: iterate over lists and maps.
- Named templates: reusable template fragments.
- `include`: call a named template inside a pipeline.
- `toYaml`: convert values into YAML.
- `nindent`: indent generated YAML correctly.
- `required`: fail rendering when an important value is missing.
- `default`: provide a fallback value.

Useful exercises:

1. Make an Ingress resource optional with a Boolean value.
2. Add a configurable environment-variable map.
3. Add a ConfigMap generated from values.
4. Add a dummy Secret generated from values.
5. Add a required value and observe the rendering failure.
6. Add a helpful message to `NOTES.txt`.

The [Helm Chart Template Guide](https://helm.sh/docs/chart_template_guide/) and [Chart Development Tips](https://helm.sh/docs/howto/charts_tips_and_tricks/) cover these techniques.

## Phase 9: Debug and validate charts

Use this validation sequence before installing a chart:

```bash
helm lint ./hello-chart
helm template hello ./hello-chart \
  --namespace helm-lab
helm template hello ./hello-chart \
  --namespace helm-lab \
  --debug
```

After installation, inspect both Helm and Kubernetes:

```bash
helm status hello --namespace helm-lab
helm get values hello --namespace helm-lab
helm get manifest hello --namespace helm-lab
kubectl get all
kubectl describe pods
kubectl get events --sort-by=.lastTimestamp
```

For a server-side validation without applying changes:

```bash
helm template hello ./hello-chart \
  --namespace helm-lab \
  --output-dir /tmp/helm-rendered
kubectl apply --dry-run=server \
  -f /tmp/helm-rendered/hello-chart/templates
```

Do not put real credentials in chart values or source control. Use dummy values for this local lab.

## Phase 10: Package and test a chart

Package the chart:

```bash
helm package ./hello-chart
```

Inspect chart metadata:

```bash
helm show chart ./hello-chart
helm show values ./hello-chart
```

Run chart tests after adding or reviewing the test template:

```bash
helm test hello --namespace helm-lab
```

Learn how to declare chart dependencies in `Chart.yaml`. Start with local or well-understood dependencies rather than adding many external charts.

## Phase 11: Build the capstone chart

Create a new chart named `web-lab` without relying on the generated chart after you understand it:

```bash
helm create web-lab
```

The capstone should contain:

- Deployment
- Service
- ConfigMap
- Dummy Secret
- Configurable replica count
- Configurable image repository and tag
- Configurable service port
- Resource requests and limits
- Readiness probe
- Liveness probe
- Optional Ingress
- `NOTES.txt`
- Development and production values files

Recommended layout:

```text
web-lab/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── ingress.yaml
    ├── _helpers.tpl
    ├── NOTES.txt
    └── tests/
```

Validate the capstone:

```bash
helm lint ./web-lab
helm template web-lab ./web-lab
helm install web-lab ./web-lab \
  --namespace helm-lab \
  --create-namespace
```

Then practice:

```bash
helm upgrade web-lab ./web-lab \
  --namespace helm-lab \
  --set replicaCount=2

helm history web-lab --namespace helm-lab

helm rollback web-lab 1 \
  --namespace helm-lab

helm uninstall web-lab \
  --namespace helm-lab
```

## Completion checklist

You are ready to move beyond the basics when you can answer these questions:

- What is the difference between a chart, a release, and a rendered manifest?
- What happens when `helm install` runs?
- How does Helm combine `values.yaml`, `--values`, and `--set`?
- How do labels and selectors connect a Deployment and a Service?
- How do you inspect the exact YAML generated by Helm?
- How do you debug a failed release?
- How do you upgrade and roll back a release?
- How do you make a resource optional?
- How do you package and test a chart?
- How do you avoid storing real secrets in a chart repository?

## Cleanup

Remove Helm releases first:

```bash
helm uninstall hello --namespace helm-lab
helm uninstall web-lab --namespace helm-lab
```

Stop Minikube while preserving the cluster:

```bash
minikube stop
```

Delete the local cluster completely:

```bash
minikube delete
```

This cleanup affects only your local Minikube environment. It does not affect AWS because this plan never creates AWS resources.
