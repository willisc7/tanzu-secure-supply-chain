# VMware Tanzu Secure Supply Chain

### Maintainer
* Current mainainer is Chris Willis (chwillis@vmware.com)
* Various commits from Tanzu Federal Solutions Engineering team

### Overview
This pipeline works by first performing a static code analysis (e.g. SonarQube scan) when new code arrives in the spring-music sample repo. If the scan passes the quality threshold configured in the static code analysis tool, Tanzu Build Service (TBS) is used to build the application container using image layers and buildpacks that are constantly updated/patched. Once the hardened container is built, a MySQL database helm chart is applied to the target kubernetes cluster and deploys a MySQL service hardened using VMware's Tanzu Application Catalog (TAC) automation. Lastly, a helm chart deploys the application container and injects an agent for a dynamic code scanning tool (e.g. Contrast) to protect against future vulnerabilities and zero days that may surface. The result is a pipeline that ensures security and quality in code deployments from developer commit to production and back again (whenever a vulnerability is discovered).

![alt text](https://github.com/willisc7/images/blob/main/Tanzu%20Secure%20Supply%20Chain.jpg?raw=true)

### TL;DR
0. Whenever developer commits code to the monitored repo, perform a static code analysis
0. If the code passes the quality gate, TBS will trigger and build the app container
0. Deploy MySQL database using Tanzu Application Catalog (TAC) helm chart and container image
0. Deploy TBS-generated application image using helm chart and inject dynamic code analysis agent

### To Do
* Configure docker authentication so pipeline doesnt need all repos to be public
* Change `static-code-scan` task to be more generic
* Figure out why helm resources are yellow sometimes with vSphere TKG clusters
* For vSphere with Tanzu TKG clusters, figure out why `kubectl apply -f ./manifests/svc-acct-rbac-role.yaml` authentication doesnt work
* Have ingress deployed as part of spring-music helm chart

## Getting started

### Credhub Secrets
Seeding specific credentials in Credhub is required prior to the use of this pipeline. For your convenience, a template for the required credentials is stored in `credhub-template.yml`

| Keyword | Explanation |
|---------|:------------|
| `k8s_svc_token_<CLUSTER_NAME>` | the token with proper kubernetes privileges that the pipeline will use to interact with the kubernetes cluster |
| `git.username` | the username that will be used to pull code from git (will be used for more later as pipeline is enhanced) |
| `git.private_key` | private key for the git account |
| `s3_access_key_id` | S3 ID to access buckets that store container image that will be used to run all pipeline jobs |
| `s3_secret_access_key` | S3 secret key to access buckets that store container image that will be used to run all pipeline jobs |

### Deploy an Ingress (Optional)
* On AWS, review and edit `./manifests/alb-rbac-role.yaml` and `./manifests/alb-ingress-controller.yaml` and run `kubectl create -f ./manifests/kube2iam.yml && kubectl create -f ./manifests/alb-rbac-role.yaml && kubectl create -f ./manifests/alb-ingress-controller.yaml`
* On vSphere with Tanzu, install Contour using `helm repo add bitnami https://charts.bitnami.com/bitnami && helm install my-release bitnami/contour --set envoy.service.externalTrafficPolicy="Cluster"`

### If Using Self-Signed Certs (Optional)
If you're using self-signed certs, modify helm concourse resource to trust your CA cert (doesnt appear to be an easy way around this right now)
* `docker pull typositoire/concourse-helm3-resource`
* `docker run -it typositoire/concourse-helm3-resource bash`
* Copy and paste CA cert into `/usr/local/share/ca-certificates/ca.crt` and run`update-ca-certificates`
* Open a new shell and run `docker ps` to get the <CONTAINER-ID>
* `docker commit <CONTAINER-ID> <IMG_REPO_FQDN>/concourse-helm3-resource`
* `docker push <IMG_REPO_FQDN>/concourse-helm3-resource`

### Create TBS Image Config
This pipeline assumes TBS already exists and that you have access to it.
* `kp secret create image-repo --registry https://<IMG_REPO_FQDN> --registry-user <IMG_REPO_USERNAME>` and type in your password when prompted
* `kp secret create git --git-url git@<GIT_FQDN> --git-ssh-key <GIT_SSH_KEY>`
* `kp image create spring-music --tag <IMG_TAG> --git git@<GIT_FQDN>:<GIT_REPO>.git --git-revision master --wait`

### Setup Environment
* Create svc account for Concourse to use to interact w/ k8s cluster 
  * For vSphere with Tanzu TKG clusters, just use auth credentials from your kube config and skip these steps
  * `kubectl apply -f ./manifests/svc-acct-rbac-role.yaml`
  * Copy value of secret for svc account `kubectl get secret $(kubectl get sa concourse -o json | jq -r '.secrets[0].name') -o json | jq -r '.data.token' | base64 -d`
  * Edit the svc token secret in `credhub-template.yml`, paste the value of the secret, delete anything thats not needed, and run `credhub import -f ./credhub-template.yml`
* `cp ./params-template.yml ./params.yml` and edit the appropriate values
  * For **k8s.cluster_ca**, you can get the value by running the following on a mac `kubectl get secret $(kubectl get sa concourse -o json | jq -r '.secrets[0].name') -o json | jq -r '.data."ca.crt"' | pbcopy`
* Edit `./manifests/spring-music-ingress.yaml` appropriately and create it using `kubectl create -f ./manifests/spring-music-ingress.yaml`
* Log into Concourse `fly login -t <CONCOURSE_TARGET> -c <CONCOURSE_URL> -n <CONCOURSE_TEAM_NAME> --insecure`
* Create the pipeline using `fly -t <CONCOURSE_TARGET> sp -p "tanzu-secure-supply-chain" -c ./pipeline.yml -l ./params.yml`
* Unpause the pipeline using `fly -t <CONCOURSE_TARGET> up -p "tanzu-secure-supply-chain"`
* Bump spring-music to trigger pipeline `git commit --allow-empty -m "bump" && git push`