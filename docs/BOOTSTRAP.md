# Bootstrapping hlc.marks.dev

The following is a record of the deployment of Happy Little Cloud, my Raspberry-Pi-based Kubernetes cluster designed to host [marks.dev](https://marks.dev) and other personal projects.

## Approach and Goals

- I am using [this guide by Alexander Sniffin](https://alexsniffin.medium.com/a-guide-to-building-a-kubernetes-cluster-with-raspberry-pis-23fa4938d420) as a starting template
- Build in additional production-grade security (e.g. no unauthenticated logins, port restrictions)
- Leverage ArgoCD as the primary orchestration system for all services
- Create Helm charts for all internal services to be managed by ArgoCD
- Services for the cluster to manage:
  - `HTTP/80` - Internal websites and local network traffic
  - `HTTPS/443` - External site traffic using letsencerypt.org as the CA
  - `PiHole/53` - [DNS-based ad blocking](https://pi-hole.net/) for running on Raspberry Pi
  - `DNS/53` - BIND internal DNS server managing internal domains with PiHole as its root
  - `Prometheus/9090` - Metrics gathering and system monitoring
  - `Grafana/8080` - System analytics and data visualization site
  - `Ansible/22` - Cluster-hosted baremetal server orchestration and configuration
  - `Tekton` - Kubernetes-based cloud-native CI/CD pipeline framework
  - `Upptime/443` - Uptime Monitoring and status page ([status.marks.dev](https://status.marks.dev))
  - `Nextcloud` - [On-prem cloud solution](https://nextcloud.com/) for sharing files, calendars, contacts, and tasks across all devices
  - `Plex/32400` - [On-prem media streaming service](https://plex.tv/) for TV, Movies, and Audio
  - `Home Assistant` - [Home automation](https://www.home-assistant.io/) and IoT device management solution

## Stage 1 - Initial Deployment

- Stage goals:
  - Get a 12-node Kubernetes cluster online in 2023.
  - Abandon Ubuntu configurations for Debian 12 base OS
  - Deploy the cluster via k3s, standard k8s binaries, or alternate method
  - Deploy ArgoCD and set up basic cluster services
  - Provide storage, ingress, and load balancer services
  - Enable Prometheus logging and monitoring with Graphana dashboards
  - Deploy first network services: DNS and hosting `marks.dev`
  - Automate along the way as much as possible

### Stage 1.1 - Baremetal server configuration

- Pull latest [tested-images](https://raspi.debian.net/tested-images/) from [raspi.debian.net](https://raspi.debian.net)
- Image all 12 Raspberry Pi SDs
- Manual SD configuration:
  - `config.txt` - set hostname and authorized key
  - `cmdline.txt` - append `cgroup_enable=memory cgroup_memory=1`
- Install all SDs and boot Raspberry Pis
- Configure DHCP address reservations on Unifi router
- Configure DNS entries in PiHole
- Test: `for i in 401 402 403 404 301 302 303 304 305 306 307 308; do fping hlc-$i.marks.dev; done`

### Stage 1.2 - Cluster setup thru Ansible

- Refresh `ansible` repo and create `marks_dev` role to start configuration
- Reorganize `marks_dev` repo to support future tooling and documentation needs
- `marks_dev` role setup:
  - Create `bob` service user for cluster operations
- Initial run:
  - Need to set `remote_user: root` to plays in `marks.dev.yml`
  - Install python3 & pip3: `for i in 401 402 403 404 301 302 303 304 305 306 307 308; do ssh root@hlc-$i.marks.dev "apt update && apt install -y python3 python3-pip"; done`
  - Get `common` role installed
  - Debug `marks_dev` role for installing `bob` user
  - Add installing Docker to setup
  - Add installing Python3 as well
  - Until I get handlers setup: `for i in 401 402 403 404 301 302 303 304 305 306 307 308; do ssh root@hlc-$i.marks.dev reboot; done`
  - Got Kubernetes installed with `containerd` as the CRI

### Stage 1.3 - Attempted boostrapping with standard Kubernetes

- First bootstrapping
  - log in as `bob` on `hlc-401.marks.dev`
  - copy over `kubeadm-config.yaml` file
  - `sudo kubeadm init --config kubeadm-config.yaml`
  - move `/etc/kubernetes/admin.conf` to `/home/bob/.kube/config`
  - `kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml`
  - `for i in 402 403 404 301 302 303 304 305 306 307 308; do ssh root@hlc-$i.marks.dev kubeadm join 10.23.50.41:6443 --token builda.happylittlecloud --discovery-token-ca-cert-hash sha256:87ce29360abc518d6a0e74fee99bb53e66858ee6da89e8c6015905eeb8616248; done`
- Troubleshooting
  - deployed k8s `1.29.0` and `1.26.12` with constant `CrashLoopBackOff`s
  - issue seems to be due to resource constraints
  - attempt to install k3s with the ansible playbook 
  - distributed setup did not work
  - attempting now with k3s manually after configuring shared storage

### Stage 1.4 - Different approach with k3s

- Configuring NFS
  - install `nfs-kernel-server` and `xfsprogs` on hlc-401
  - add `ntp` and server updating to ansible playbooks
  - create xfs partition on `/dev/sda1`
  - create `/data` directory and mount partition to it
  - xfs caused i/o errors, defaulted to ext4
  - nfs server ready to share
  - looking at `nfs-subdir-external-provisioner` as an on-cluster storage solution
- Starting up k3s
  - `ssh-copy-id bob@hlc-###.marks.dev`
  - already present due to attempted ansible playbook
  - needed to try once and fail, run `k3s-uninstall.sh`, then try command again:
  - `curl -sfL https://get.k3s.io | K3S_TOKEN=<redacted> sh -s - server --cluster-init`
  - `cat /var/lib/rancher/k3s/server/node-token` to get full token
  - Now, create 3 more control plane nodes from the rpi4s:
  - `curl -sfL https://get.k3s.io | K3S_TOKEN=<redacted> sh -s - server --server https://hlc-401.marks.dev:6443`
  - To remove etcd, I had to uninstall k3s on hlc-401 and run:
  - `curl -sfL https://get.k3s.io | K3S_TOKEN=<redacted> sh -s - server --disable-etcd --write-kubeconfig-mode=644 --server https://hlc-402.marks.dev:6443`
  - The rest of the servers (301-308):
  - `curl -sfL https://get.k3s.io | K3S_TOKEN=<redacted> sh -s - agent --server https://hlc-401.marks.dev:6443`
- Labeling:
  - `kubectl label node hlc-401.marks.dev node-role.kubernetes.io/nfs=true`
  - `for i in 301 302 303 304 305 306 307 308; do kubectl label node hlc-$i.marks.dev node-role.kubernetes.io/agent=true; done`
  - `for i in 301 302 303 304 305 306 307 308; do kubectl label node hlc-$i.marks.dev node-role.kubernetes.io/agent-; done`
  - `for i in 301 302 303 304 305 306 307 308; do kubectl label node hlc-$i.marks.dev node-role.kubernetes.io/worker=true; done`

## Stage 2 - Filling out core functionality

- What's left to do:
  - Still need to install `nfs-subdir-external-provisioner`
  - Need to determine where etcd is storing its crap and use a USB key to supplement storage
  - `/var/lib/rancher` seems to be the culprit...worth exploring a solution
    - Use 4 USB3.0 drives for this (all 4 control planes)
  - Need to determine a load-balancing solution
    - MetalLB is an option - [metallb.universe.tf](https://metallb.universe.tf/)
  - ArgoCD
  - All the rest thru Argo
  - Super-basic site for `hlc.marks.dev` to prove functionality
  - Prettify shell (motd and PS1)

### Stage 2.1 - Storage

- New Years First Steps
  - Break out old `hlc-salt-data` repo to get old motd configuration
  - Update ansible playbooks to include motd configuration
  - Verify all logins look prettified for now
  - Format partitions and mount to /var/lib/rancher/k3s, migrating data and etcd stuff
- NFS Setup
  - https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner#with-helm
  - `helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/`
  - `helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=hlc-401.marks.dev --set nfs.path=/data`
  - test: `kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/test-claim.yaml -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/test-pod.yaml`
  - results:

    ```bash
    bob@hlc-401:/$ cd /data                                                                            
    bob@hlc-401:/data$ ls                                                                              
    hlc  kube-system-test-claim-pvc-ded87e89-d0f8-45ac-84b4-2a61cf741f15  lost+found  test
    bob@hlc-401:/data$ cd kube-system-test-claim-pvc-ded87e89-d0f8-45ac-84b4-2a61cf741f15/             
    bob@hlc-401:/data/kube-system-test-claim-pvc-ded87e89-d0f8-45ac-84b4-2a61cf741f15$ ls              
    SUCCESS
    ```

### Stage 2.2 - Getting ArgoCD Online

- ArgoCD setup:
  - `kubectl create namespace argocd`
  - `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml`
  - Tried `kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'`
  - Failed because Traefik is already binding to all `80` and `443` ports for all server nodes
  - Need to do this thru Traefik
- Traefik setup:
  - Built in to k3s, works similarly to `ingress-nginx`
  - Reading up on usage, handles TLS termination!
  - Cannot reach the portal at all due to missing certs
  - Can't go without `cert-manager` because HTTPS is required for `.dev` TLDs
- Cert-Manager setup:
  - helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.13.3 --set installCRDs=true
  - Didn't work as intended due to DNS01 being required for an HSTS domain to validate certs
- GoDaddy DNS01 webhook
  - `git clone git@github.com:snowdrop/godaddy-webhook.git`
  - `cd godaddy-webhook`
  - `helm install -n cert-manager godaddy-webhook ./deploy/charts/godaddy-webhook`
  - Couldn't get it to work no matter what I tried
  - Attempting to migrate public DNS to AWS
  - Uninstalled the godaddy-webhook Helm chart
- AWS Route53 DNS01 setup
  - [Instructions](https://deploy-preview-435--cert-manager.netlify.app/v0.14-docs/configuration/acme/dns01/route53/)
  - Set up IAM role and credentials
  - Cert-manager configuration failed again
  - Credentials tested correctly on the CLI
  - `AWS_ACCESS_KEY_ID=<redacted> AWS_SECRET_ACCESS_KEY=<redacted> aws route53 change-resource-record-sets --hosted-zone-id Z0801011JLVGM02KBXSX --change-batch file://test-resource-record.json`
  - Proved the credentials work, but cert-manager still is not
  - Further tweaking found that using `accessKeyIDSecretRef` does not work
  - Attempted using `accessKeyID` with plaintext string, and that worked!
  - Both a wildcard cert for `*.marks.dev` and `whoami.marks.dev` were validated successfully
  - We are finally ready to start working on Argo!
- Install ArgoCD
  - Create cert first `k apply -f argocd-certs.yaml`
  - Create ingress `k apply -f argocd-ingress.yaml`
  - Add `argo` repo: `helm repo add argo https://argoproj.github.io/argo-helm`
  - Install chart `helm install -n argocd argocd argo/argo-cd -f argocd-values.yaml`
  - Needed to upgrade values multiple times as I figured out Traefik
  - Eventually did not use the `IngressRoute` but used the built-in `Ingress` as part of the ArgoCD Helm Chart
  - Futzed with `argocd-certs.yaml` and `argocd-ingress.yaml` for a while to no avail, going off the ArgoCD docs
  - Worked like a charm as soon as I got the Argo Helm Chart configuration set up with cert-manager!
- First Logins
  - `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
  - Log in with `admin` user with above password to UI
  - `argocd admin initial-password -n argocd`
  - `argocd login argocd.marks.dev`
  - Test deployment:
  - `argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default`
  - Didn't work as expected, but it appears to be the template fault
  - Ready to start deploying!

## Stage 3 - First Public Services

Stage Goals:

- HLC Microsite
- marks.dev website
- marks.dev GitHub Actions pipeline
- mr-poopybutthole Discord bot
- prometheus cluster montioring
- grafana dashboard/observability service
- bind9 internal recursive DNS server
- upptime status website

### Stage 3.1 - Let's build just a happy little cloud

- Realize the quote is "Let's build just a happy little cloud" for a start...
  - It often helps to know the inspiration for this whole damn thing in the first place
- App of Apps design
  - A single app defining an entire cluster of apps
  - [Literally just followed the docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- Happy Little Cloud Helm chart
  - First actual helm chart
  - Inspired by [butt.holdings](http://butt.holdings)
  - Used YouTube video downloader to download episode
  - Used Gifcurry AppImage to edit gif
  - Attempted configMap with binaryData (base64 encoded)
  - Initial attempt failed because configmaps were too large
  - Temporarily set up low-res gif to get it to work
  - Added Docker image to upload high-res assets
  - Got it online finally! Cert-manager works too!
- Mr. PoopyButthole Discord Bot Helm chart
  - Stupid simple single pod deployment
  - Use stripped down Helm chart template
  - Worked way too easily
  - How nice of me to leave a stupidly bad security issue
  - Definitely need to fix that soon
- marks.dev
  - Doing very well, I'm just going right for the gold
  - It worked way too easily to be honest
  - I guess I am just used to way more difficult problems
  - I do need a way to fix the build pipeline tho
- marks.dev GitHub Actions pipeline
  - I really miss GitHub Actions...it's Azure DevOps without all the bullshit
  - After Jenkins and Azure DevOps Pipelines, this is a piece of cake
  - It took some fighting, but it was not super hard
  - GitHub Secrets are good and easy, variables not so much
  - Once I handled the quirks, it went quite well
  - Only took about 15 builds to get it running!

### Stage 3.2 - Getting serious

- Prometheus and Grafana
  - Easy simple single-file templates in `hlc` chart
  - Prometheus install was very fast and simple, only required an ingress setup
  - Grafana was similarly simple, as well as automatic connection to Prometheus
  - Dashboards installed and configured by default, so far so good!
- Issue with Prometheus
  - Storage was using the wrong `StorageClass`, not using the `nfs-client` by default
  - Required manually patching the default `StorageClass` as follows:
    - `k patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'`
    - `k patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
  - Deleting the PVCs
    - `k delete pvc -n prometheus prometheus-server`
    - `k delete pvc -n prometheus storage-prometheus-alertmanager-0`
  - Restarting services
    - `k rollout restart -n prometheus deploy/prometheus-server`
    - `k rollout restart -n prometheus statefulset/prometheus-alertmanager`
  - Services were not coming up until the nfs-subdir-external-provisioner was restarted:
    - `k rollout restart -n kube-system deploy/nfs-subdir-external-provisioner`
- Grafana login issue
  - Can login with default admin account, but the password keeps changing
  - Attempted removing the Helm metadata, did not help
  - Analyzed the chart and it appears to erroneously force password generation
  - [https://github.com/helm/helm-www/issues/1259](https://github.com/helm/helm-www/issues/1259)
  - Above bug notes this in the chart, and reading the chart corroborates
  - It is supposed to read the existing secret data but does not appear to do so correctly
  - Tried setting `admin.existingSecret: true` based on my reading of the chart
  - Using `immutable: true` for the secret (k8s features since `1.22`) is working so far
- Blocked on Storage Issues
  - `hlc-401` continues to have its filesystems go read-only, both the external SSD and USB drives
  - storage is not reliable for more than a couple of day maximum
  - Grafana and Prometheus were proven out temporarily, but are no longer working due to lack of storage
  - Need to re-approach the plan
  - Moving on to HLC Mark 3

TODOs:

```text
(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

Result: Failure
Reason: Cheapening out on storage and not doing it right in the first place
