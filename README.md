# AI Scan Repos installer

One Docker image that deploys AI Scan Repos with its full platform (image
registry, Jenkins with the three pipelines, GitHub webhooks, Argo CD) to:

- a cluster on your computer (minikube, Docker Desktop, any kube context),
- a server you can SSH into (K3s or kubeadm),
- Amazon EKS, created with Terraform (VPC, EKS, EFS, ECR).

A web wizard asks for what it cannot know, and tells you the few things only
you can do (a GitHub token, a GitHub OAuth App, `minikube tunnel`).

## Before you start

- Docker.
- A GitHub token that can read the six project repositories, add webhooks to the three code
  repositories and push to this one, created at
  [github.com/settings/tokens](https://github.com/settings/tokens/new?scopes=repo,admin:repo_hook&description=ai-scan-repos-installer)
  with `repo` and `admin:repo_hook`.
  This creates a personal access token under the GitHub account currently signed in.
  > That account must have the required repository access, and
  organization SSO or token-approval policies may require additional authorization.
  
- For minikube: a cluster with at least 8 GB of memory, or the Jenkins builds starve it. A separate
  profile leaves your other clusters alone: 
  ```bash
  minikube start -p ai-scan --cpus 6 --memory 10g --container-runtime=docker
  ```
  (then run the installer with `--network ai-scan`, pick the context `ai-scan`, and use `minikube tunnel -p ai-scan`).
- For EKS: an AWS account; either a `~/.aws` profile or access keys.

## Build and publish the installer image

Run these commands from the repository root. Authenticate to Docker Hub first:

```bash
docker buildx create --name ai-scan-builder --use
docker buildx inspect --bootstrap
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t docker.io/aviel770/ai-scanner-installer:latest \
  --push .
```

The `--platform` option builds one image for both 64-bit Intel/AMD (`amd64`)
and ARM (`arm64`) Linux systems; `--push` publishes the multi-platform image
and its manifest to the selected registry. You can replace `latest` with a
version tag and use a registry-specific image name, such as an ECR URI:
`<account-id>.dkr.ecr.<region>.amazonaws.com/ai-scanner-installer:<tag>`.
Create the named builder only once; if it already exists, select it with
`docker buildx use ai-scan-builder` before building.

To build a single platform and load the result into the local Docker image
store for testing, use `--load` instead. Omitting `--platform` targets the
builder's native platform; you can specify `linux/arm64` for Apple Silicon or
ARM Linux, or `linux/amd64` for Intel/AMD Linux:

```bash
docker buildx build -t ai-scanner-installer:local --load .
```

`.dockerignore` keeps files that are not needed for the image build out of the
Docker build context, including Git metadata, tests, QA materials, Markdown
documentation, Python caches, Terraform state/cache, and generated UI files.
These files remain in the repository for development and debugging. The
Dockerfile builds the UI in its own stage and copies only the built UI and
installer runtime files into the final image.

## Run it

- Leave out `--network minikube` for any target other than minikube (always for EKS).
- Start minikube **before** the installer. If the installer joins a stopped minikube's network,
  it takes the node's address and minikube then fails with "Address already in use".
- Leave out the `.aws` mount if you will type AWS keys in the wizard, or are not using EKS.
- The volume `ai-scan-installer` keeps your encrypted answers and the Terraform
  state. Keep it until you have pressed Destroy, or EKS keeps costing money.

### MacOS / Linux (bash):

```bash
docker run --rm -it \
  -p 127.0.0.1:8088:8088 \
  -p 127.0.0.1:8081-8083:8081-8083 \
  --network minikube \
  --add-host host.docker.internal:host-gateway \
  -v ai-scan-installer:/data \
  -v "$HOME/.kube:/host/kube:ro" \
  -v "$HOME/.minikube:/host/minikube:ro" \
  -v "$HOME/.aws:/root/.aws:ro" \
  docker.io/aviel770/ai-scanner-installer:latest
```

The Docker Hub image is
`docker.io/aviel770/ai-scanner-installer:latest`. For ECR, replace the image URI with
`<account-id>.dkr.ecr.<region>.amazonaws.com/ai-scanner-installer:latest`.
The image must already be published there, and Docker must be authenticated
to that registry. Use the same image URI in the Windows example below.


### Windows (PowerShell), minikube:

```powershell
docker run --rm -it -p 127.0.0.1:8088:8088 -p 127.0.0.1:8081-8083:8081-8083 --network minikube `
  -v ai-scan-installer:/data `
  -v "$env:USERPROFILE\.kube:/host/kube:ro" -v "$env:USERPROFILE\.minikube:/host/minikube:ro" `
  -v "$env:USERPROFILE\.aws:/root/.aws:ro" `
  docker.io/aviel770/ai-scanner-installer:latest
```

Then open <http://localhost:8088>

## What it does

The wizard commits generated deployment files to the Installer repository
(default: `ai-scan-repo/ai-scanner-quick-installer`) on a new branch `env/local` (for local running / Debug) or
`env/eks` branch (When running on AWS).  
This branch is the GitOps source: Argo CD deploys from it,
and Jenkins updates image tags there. Keeping it separate avoids mixing
generated deployment files with the repository's main branch. You can choose a
different Installer repository in the wizard. The six code and deployment
repositories are only **read**, except that webhooks are added to the three
code repositories:

```
env/<env>
└── environments/<env>/
    ├── backend/   base/ (copy of ai-scanner-backend-deployment) + overlay patches
    ├── gateway/   base/ (copy of ai-scanner-gateway-deployment) + overlay patches
    ├── frontend/  base/ (copy of ai-scanner-frontend-deployment) + overlay patches
    └── ci/        <repo>.Jenkinsfile + <repo>.pod.yaml, derived from each code repo
```

| Step | Local / server | EKS |
|---|---|---|
| Cluster | your kube context, or the server's admin kubeconfig over SSH | Terraform: VPC, EKS (2 x t3.large), EBS + EFS CSI, EFS, 4 ECR repos, IAM role for Jenkins |
| Images | a registry inside the cluster | ECR |
| Ingress | Traefik on K3s, the existing nginx class, or ingress-nginx | ingress-nginx on an AWS NLB |
| CI | Jenkins: one job per code repo, using the derived Jenkinsfile; webhook through smee.io | same, webhook straight to Jenkins's load balancer |
| CD | Argo CD syncs `environments/<env>/<app>` from the `env/<env>` branch | same |
| AI model | saved once through the gateway's own Settings code | same |

The derived Jenkinsfiles (`installer/pipelines.py`) keep the original Test,
Build, Scan and Push stages. They differ only in checking out the code repo
themselves and committing image tags to `env/<env>` here. The first images come
from the first Jenkins run.

The only thing an install adds to the project repositories is a push webhook
on the three code repositories (a repository setting, not a commit). Remove
deletes it again.

## Jenkins, Argo CD and Grafana

Press **Jenkins, Argo CD, Grafana** at the top right of the wizard, then click on the **Open all three**.
The installer keeps a port-forward to each one, for any target (minikube, a server,
EKS), on ports the `docker run` line publishes on 127.0.0.1 only:

> The wizard generates the password. Use **Show** to view it or **Copy** to copy it.

| Service | Address | Login |
|---|---|---|
| Jenkins | http://localhost:8081 | `admin` + the install's admin password |
| Argo CD | http://localhost:8082 | Same Credentials |
| Grafana (with Prometheus) | http://localhost:8083 | Same Credentials |

On **EKS** each of the three gets its own (ALB) AWS load balancer instead, and the panel
links to those domain `...elb.amazonaws.com` addresses (Jenkins on port 8080).

The wizard shows the password (Show / Copy). It is generated once per install and
kept in the encrypted answers; Argo CD stores only its bcrypt hash.

## HTTPS with your own certificate (optional)

On "Where it runs", open **Add an HTTPS certificate** and choose the certificate
(PEM, with its chain) and its unencrypted private key. Set the app address to
`https://<a name the certificate covers>` (on EKS the domain field works too).
**Check certificate** verifies that the key matches, the name is covered and it
has not expired. The Ingress then serves HTTPS with it, on minikube, a server or
EKS (no ACM certificate is created then). Point the name at the cluster: a DNS
record, or the hosts file for a local minikube.

### Generate a certificate for local testing

For a private test deployment, OpenSSL can create a self-signed certificate.
Choose a DNS name you will use as the app address, and use that exact name in
the certificate's Subject Alternative Name (SAN). For example:

```bash
mkdir -p local-tls
cd local-tls
umask 077
openssl req -x509 -newkey rsa:3072 -sha256 -nodes -days 365 \
  -keyout app.key \
  -out app.crt \
  -subj "/CN=ai-scan.local" \
  -addext "subjectAltName=DNS:ai-scan.local"
```

In the wizard, set the app address to `https://ai-scan.local`, upload
`app.crt` as the certificate and `app.key` as the private key, then select
**Check certificate**. Make `ai-scan.local` resolve to the cluster's ingress
address on the machines that need to open the app (for a local cluster, use a
hosts-file entry). The private key is unencrypted so the wizard can read it;
keep `app.key` private and do not commit or share it. The installer stores
uploaded answers encrypted on its data volume.

> Note - Self-signed certificate encrypts traffic but browsers and other clients
will not trust it by default, so expect a certificate warning unless you
explicitly trust it on each client. Do not use this as a publicly trusted
production certificate.

For an internet-facing deployment, use a certificate issued by a trusted
certificate authority (for example, through your organization's PKI or an
ACME provider such as Let's Encrypt). Obtain the certificate for the exact
hostname and upload the leaf certificate together with its intermediate
chain, plus its matching unencrypted private key. Configure DNS to point that
hostname at the cluster before use. Protect the private key, and never commit
it to Git. This installer option uses the uploaded certificate directly; it
does not request or manage an ACM certificate on EKS.

## Secrets

Everything you type stays in the installer's memory and in `answers.enc` on
the volume, encrypted with your passphrase (scrypt + AES-GCM). Inside the
cluster they become Kubernetes Secrets. The log never shows them. Nothing is
written to git except the overlays, which hold no secrets.

## Clean up a local Minikube node

To remove the installed platform, use **Remove/Destroy** in the wizard while
the installer and cluster are still running. To delete the entire Minikube
cluster afterward, including all data stored in it, run:

```bash
minikube delete -p ai-scan
```

This removes the `ai-scan` Minikube profile, but does not remove the installer
Docker image or the separate Buildx builder. To stop the cluster temporarily
and keep it for later, run `minikube stop -p ai-scan` instead. Verify the
profile was deleted with `minikube profile list`. Use Minikube to manage its
node; do not remove the node container directly with `docker rm`.

## Developing

```bash
pip install -r requirements-dev.txt && pytest -q
uvicorn installer.app:app --port 8088        # the API
cd ui && npm install && npm run dev           # the page, proxied to the API
```
