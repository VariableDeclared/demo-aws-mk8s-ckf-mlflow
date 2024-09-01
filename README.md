# Multicloud E2E RAG demo

## TODO
- Implement notebook which:
    - takes data from S3 (creates if does not exist)
    - create embeddings 
    - stores values in Opensearch 
- Create a pipeline from notebook, add:
    - 3 steps - create DB, download, create/recreate index, create embeddings, save in opensearch
- Deploy the LLM using vLLM & KServe
- Create Chat UI using embedder & KServe endpoint
    - simple chat application with history

**Enhancements**
- remove jumphost.sh & apps.sh
- Add Opensearch COS monitoring
- use Embeddings service as KServe endpoint
- multicloud:
    - juju cloud config
    - jumphost create command
- NVidia NIM as LLM & Embeddings

## Infrastructure Installation

### Create the jumphost

Skip if you already have a jumphost or decide to use your local machine.

For **AWS**, go to the aws-jumphost folder and use terraform:

```bash
cd ./aws-jumphost
terraform init
terraform apply
```

Export the public ip of the jumphost and SSH using defined key

```bash
export JUMPHOST_IP=$(terraform output -raw instance_public_ip)
export PATH_TO_KEY=...

ssh -i PATH_TO_KEY ubuntu@$JUMPHOST_IP
```

Install common packages for jumphost from project root directory

```bash
cd ./..
bash jumphost-common.sh
```

### Juju cloud configuration

Juju cloud credentials need to be configured separately for each of the clouds, more info can be found [here](https://juju.is/docs/juju/juju-add-credential)

For **AWS**, configure the aws cloud config file based on [template](`./aws-jumphost/aws-credentials.tmp`) with your AWS IAM user credentails and add them to the cloud:

```bash
juju add-credential aws -f ./aws-jumphost/aws-credentials.yaml
```

### Deploy Kubernetes cluster

Bootstrap juju controller

```bash
juju bootstrap aws/eu-west-1 aws-controller --bootstrap-constraints 'cores=2 mem=4G'
```

Deploy kubernetes cluster with Juju and Microk8s

```bash
juju add-model mk8s aws

juju deploy ./k8s/k8s-bundle.yaml --model mk8s

juju ssh -m mk8s microk8s/leader -- sudo microk8s status
```

We are using hostpath storage to eliminate the dependency on the external cloud. The root disk is 100GB to acomodate both Kubernetes hostpath storage and Docker Image caching.

Configure additional microk8s plugins

```bash
juju ssh -m mk8s microk8s/leader -- sudo microk8s enable gpu ingress metallb:10.64.140.43-10.64.140.49

juju expose microk8s
```

Save kubeconfig into the kube config default, if you do not use jumphost consider using different location.

```bash
juju ssh -m mk8s microk8s/leader -- sudo microk8s config > ~/.kube/config
```

Taint GPU nodes with PreferNoSchedule:
```bash
kubectl get nodes -l "nvidia.com/gpu.present=true" -o jsonpath='{.items[*].metadata.name}' \
    | xargs -I{} kubectl taint nodes {} node-preference=gpu:PreferNoSchedule --overwrite
```

Optionally, install volcano scheduler if you need more advanced scheduling policies for your workloads.

```bash
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/master/installer/volcano-development.yaml
```

### Deploy COS

Add deployed K8s as a cloud

```bash
juju add-k8s mk8s --cluster-name=microk8s-cluster --client --controller aws-controller
```

Deploy the Observability stack

```bash
juju add-model cos mk8s

juju deploy cos-lite --model cos \
  --trust \
  --overlay ./cos/offers-overlay.yaml \
  --overlay ./cos/storage-small-overlay.yaml
```

To access the COS, go to the section "Access the UIs"

Add the self monitoring the deployed Kuberentes cluster

```bash
juju consume aws-controller:admin/cos.alertmanager-karma-dashboard cos-alertmanager -m mk8s
juju consume aws-controller:admin/cos.grafana-dashboards cos-grafana -m mk8s
juju consume aws-controller:admin/cos.loki-logging cos-loki -m mk8s
juju consume aws-controller:admin/cos.prometheus-receive-remote-write cos-prometheus -m mk8s

juju deploy grafana-agent grafana-agent-cos --channel latest/stable -m mk8s

juju relate grafana-agent-cos:cos-agent microk8s:cos-agent -m mk8s
#juju relate grafana-agent-cos:cos-agent microk8s-gpu:cos-agent -m mk8s
juju relate cos-loki:logging grafana-agent-cos:logging-consumer -m mk8s
juju relate cos-prometheus:receive-remote-write grafana-agent-cos:send-remote-write -m mk8s
juju relate cos-grafana:grafana-dashboard grafana-agent-cos:grafana-dashboards-provider -m mk8s
```

### Deploy Kubeflow and MLflow

Deploy Kubeflow, MLflow and integrate with COS

```bash
juju add-model kubeflow mk8s

juju deploy -m kubeflow --debug ./ckf/bundle.yaml \
    --overlay ./ckf/authentication-overlay.yaml \
    --overlay ./ckf/cos-integration.yaml \
    --overlay ./ckf/mlflow-integration.yaml \
    --trust
```

Log into the UI for the first time using credentials:
User: admin
Password: admin

### Deploy Opensearch

Create a new model and set cloudinit-userdata for it.

```bash
juju add-model os aws

juju model-config --model os --file=./opensearch/cloudinit-userdata.yaml
```

Deploy Opensearch.

```bash
juju deploy ./opensearch/bundle.yaml
```

Ignore the error with replicas, or deploy Opensearch in HA mode.

When deployment is GREEN, get the access information:

```bash
# juju run data-integrator/leader get-credentials > ./opensearch/os-creds.yaml
juju run opensearch/leader get-password > ./opensearch/os-creds.yaml

# export OS_IP=$(cat ./opensearch/os-creds.yaml | yq -C ".opensearch.endpoints")
export OS_IP=$(juju status -m os opensearch/leader --format json | jq -r '.machines[] | .["dns-name"]')
export OS_PORT=9200
echo Endpoints: $OS_IP:$OS_PORT


export OS_USERNAME=$(cat ./opensearch/os-creds.yaml | yq -C ".username")
# export OS_USERNAME=$(cat ./opensearch/os-creds.yaml | yq -C ".opensearch.username")
echo Username: $OS_USERNAME

export OS_PASSWORD=$(cat ./opensearch/os-creds.yaml | yq -C ".password")
# export OS_PASSWORD=$(cat ./opensearch/os-creds.yaml | yq -C ".opensearch.password")
echo Password: $OS_PASSWORD

cat ./opensearch/os-creds.yaml | yq -C ".ca-chain" | tee ./opensearch/os-cert.yaml
echo Certificate saved under ./opensearch/os-cert.yaml
```

Connect using curl to check connectivity:
```bash
curl -k --cacert ./opensearch/os-cert.yaml -XGET https://$OS_USERNAME:$OS_PASSWORD@$OS_IP/
```

Create Opensearch secret and PodDefaults in the "admin" user namespace in Kubeflow. This requires that you log into the Kubeflow for the first time before running the script below.

```bash
sh ./opensearch/os-pod-default.sh
```

### Configure Object storage Bucket and Opensearch Index

Go to the Kubeflow and create a Kubeflow Notebook with all PodDefaults enabled.

In the Kubeflow notebook run:
- setup-bucket.ipynb
- setup-opensearch.ipynb

### Deploy ML models

TBD

### Deploy Chat UI

TBD

## Access the UIs

### Kubeflow, MLflow & COS
For the security purposes we do not expose the "Public IPs" publicly. 

First, add you public key to the Kuberentes leader node. I will use my launchpad ID, you can also add your public key directly to the ~/.ssh/authorized_keys on the remote host.

```bash
juju ssh -m mk8s microk8s/leader -- ssh-import-id barteus
```

Next step is expose them to your computer via sshuttle. Use new terminal window on your local computer. You will need root access to your computer, because sshuttle will add additional entries to your IP tables.

On the jumphost run:

```shell
MK8S_LEADER_IP=$(juju status -m mk8s microk8s/leader --format json | jq -r '.machines[] | .["dns-name"]')
echo $MK8S_LEADER_IP
```

On your local computer in new terminal run:

```bash
shuttle -r ubuntu@$MK8S_LEADER_IP 10.0.0.0/8 172.31.0.0/16
```

Get the IP of the COS entrypoint. In the catalog you can find links to other services.

```bash
juju run -m cos traefik/0 show-proxied-endpoints --format=yaml --model cos \
  | yq '."traefik/0".results."proxied-endpoints"' \
  | jq
```

**Grafana** admin user details can be extracted using Juju action:

```bash
echo Grafana access
juju run grafana/leader get-admin-password --model cos
```

Kubeflow access to the UI:

```bash
echo Kubeflow access
echo IP: $(kubectl -n kubeflow get svc istio-ingressgateway-workload -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo User: $(juju config dex-auth static-username)
echo Password $(juju config dex-auth static-password)
```

## Cleanup

Remove in the AWS cloud console:
- machines
- security groups

Remove configuration on the jumphost.

```bash
rm -Rf ~/.local/share/juju/
```

