# Kubernetes Manifests 

## Sandbox Clusters

These are a collection of manifests that define Kubernetes services. They are
monitored and managed by [Flux](https://fluxcd.io). Flux detects new images 
deployed to DockerHub published by GitHub Actions, automatically updates appropriate 
manifests based on the new image and applies the changes to the Kubernetes 
cluster.


## Setup

### Install [Flux V2 CLI](https://fluxcd.io/docs/cmd)

```bash
brew install fluxcd/tap/flux
```

or

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

### Setup Flux V2

Install Flux manifests in the directory `flux-system`.

*Note*: Be sure and install the components for image automation and reconciliation.

```bash
flux bootstrap github \
    --verbose \
    --owner surveyplanet-training \
    --repository manifests \
    --branch main \
    --components-extra=image-reflector-controller,image-automation-controller \
    --read-write-key 
```

#### Create DockerHub secret

Flux needs permision to poll dockerhub to check for new images. 

```bash
kubectl create secret docker-registry \
    dockerhub \
    --namespace=flux-system \
    --docker-server=https://index.docker.io/v2/ \
    --docker-username=<YOUR USERNAME> \
    --docker-email=<YOUR EMAIL> \
    --docker-password=<DOCKERHUB_ACCESS_TOKEN>
```

This secret is used inside deployment manifests under the `imagePullSecrets` property e.g.:

```yaml
spec.teampte.spec.imagePullSecrets:
    -
        name: dockerhub
```

#### Configure image polling and version policy [...](https://fluxcd.io/docs/guides/image-update/)

Repeat this process for each deployment (both red and blue).

Create an `ImageRepository` to tell Flux which container registry to scan for new tags:

```bash
flux create image repository red \
    --image=docker.io/lukamit/red \
    --secret-ref dockerhub \
    --export >> ./red/image-repository.yaml
```

Create an `ImagePolicy` to tell Flux which semver range to use when filtering tags:

```bash
flux create image policy red \
    --image-ref=red \
    --select-semver=^1.x-0 \
    --export >> ./red/image-repository.yaml
```

At this point your Flux should be aware of your image. Run the following command to 
try it out.

```bash
# commit and push the updates
git add -A && git commit -m "updated red image scan" && git push origin master
# Tell Flux to pull and apply changes:
flux reconcile kustomization flux-system --with-source
```

Check the available images:

```bash
flux get image repository
```

Check the available image policies:

```bash
flux get image policy
```

Edit the deployment.yaml and add a marker to tell Flux which policy to use when updating the container image:

```yaml
spec:
    containers:
    - image: lukamit/red:1.0.0 # {"$imagepolicy": "flux-system:docs"}
      name: red
```


#### Configure image updates

Create an `ImageUpdateAutomation` to tell Flux which Git repository to write image updates to:

```bash
flux create image update flux-system \
    --git-repo-ref=flux-system \
    --git-repo-path="/" \
    --checkout-branch=main \
    --push-branch=main \
    --interval=5m \
    --author-name=<NAME> \
    --author-email=<EMAIl> \
    --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
    --export > ./image-update-automation.yaml
```

*Note:* get the value for `git-repo-ref` with:
```bash
kubectl get gitrepository flux-system -n flux-system -o jsonpath={.spec.secretRef.name}
```

Assuming you're not aleady at the latest image, after a few minutes the image 
should update automatically. Check that you're at the latest image with:

```bash
kubectl get deployment red -n sandbox -o json | jq '.spec.template.spec.containers[0].image'
```

## Working with Flux

### Syncing

Flux will poll this manifest repoistory every 5 minutes and make nessisary 
updates automatically.

### Release

Releases are made automatically when a new image is pushed to
[DockerHub](https://hub.docker.com).
The only time a release may be necessary in the case of a rollback.

### Rollback

To rollback a deployment to previous release you must first suspend image updates
to prevent Flux from automatically updating to latest versions.


#### 1. Supsend image udpates
```bash
flux suspend image update flux-system
```

#### 2. Update semver in deployment maninfest
```yaml
...
spec:
  imagePullSecrets:
    - name: dockerhub
  containers:
    - image: docker.io/lukamit/red:<PREVIOUS_version> # {"$imagepolicy": "flux-system:app"}
      name: app
      ports:
        - containerPort: 9001
...
```

#### 3. Push the update to Github

```bash
git commit -am "rollback" && git push origin master
```

#### 4. Reconcile the manifests

```bash
flux reconcile source git flux-system
```

#### Resume

After you've solved the issue and you're ready to resume image scanning run the
following command:

```bash
flux resume image update flux-system
```

#### Trouble shooting Flux

##### Ensure local envirnomet is running

The `check` command will perform a series of checks to validate that the local 
environment is configured correctly and if the installed components are healthy.

```
flux check
```

##### Check source (github) status

```
flux get sources git
```

##### Check error logs

```
flux logs --level=error
```

##### Check Flux deployment

```
kubectl describe gitrepository flux-system -n flux-system
```

##### Images

To display the images that are being polled and updated by Flux execute:

```bash
flux get image repository

flux get image policy
```


## Replica limits

There is a limit to the amount of pods that can run on a single node. Staging
cluster contains 2 t2.medium nodes that allow 17 pods each (34 total) while the
production cluster has 6 t2.large nodes which allow 35 pods each (210 total).
When increasing replica count and pods get stuck in the 'Pending' state check
node capacity and ensure there isn't more pods than the max capacity.

### Max pods for cluster

The following will output the max number of pods for the entire cluster.

```bash
kubectl get nodes --all-namespaces -o json | jq -r '.items[] | .status.capacity.pods' | awk '{s+=$1} END {print s}'
```

### Total pods in cluster

To get the number of pods currently running in the cluster enter the following:

```bash
kubectl get pods --all-namespaces -o json | jq ' .items | length'
```

[Here](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt)
is a list of ec2 instance types along with the maximum number of pods available
for each.
