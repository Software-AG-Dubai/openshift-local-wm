# Running WebMethods in OpenShift Local (CRC)
## Setup CodeReady Containers (CRC) 
1. Login/Register to Redhat Console [here](https://console.redhat.com/) with **personal account** (not IBM)
2. Download Installation Binary and Pull secret from [here](https://console.redhat.com/openshift/create/local)
3. Follow installation steps  
5. (Optional) Increase memory and disk 
    ``` shell
    crc config set memory 20480
    crc config set disk-size 50
    crc config set cpus 6
    # crc config set nameserver 8.8.8.8
    ```
4. Run `crc setup` and point to pull secret location
5. Start `crc start`

## Create Project and Configure Docker
1. Run the following 
    ``` shell
    eval $(crc oc-env)
    oc new-project webmethods 
    ```
2. Login to OpenShift Local Embedded Image Registry
    ``` shell
    docker login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps-crc.testing
    podman login  --tls-verify=false -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps-crc.testing
    ```
3. Add `default-route-openshift-image-registry.apps-crc.testing` as an insecure registry to your docker daemon. PORT is `443` if required
    1. For Docker Desktop on Mac [here](https://stackoverflow.com/a/74856653) and add registry certificate to trusted cert store as mentioned [here](https://docs.docker.com/engine/network/ca-certs/#macos) 
    2. For Podman Desktop 
        1. `podman machine ssh -- 'echo "192.168.127.254 default-route-openshift-image-registry.apps-crc.testing" | sudo tee -a  /etc/hosts'` [ref](https://github.com/crc-org/crc/issues/3897#issuecomment-1882629521)
        2. `podman machine ssh -- 'echo -e "\n[[registry]]\nlocation = \"default-route-openshift-image-registry.apps-crc.testing\"\ninsecure = true" | sudo tee -a /etc/containers/registries.conf'`
        3. `podman machine stop && podman machine start`

### Pull and Push wM images
* If not previously logged in to IBM Registry [cp.icr.io]() 
    * Fetch Creds using IBM ID [https://myibm.ibm.com/products-services/containerlibrary](https://myibm.ibm.com/products-services/containerlibrary)
    * Login to WM CR 
        ```
        docker login cp.icr.io -u cp -p <PASTE_TOKEN_GENERATED>
        ```

The flag `--platform linux/amd64` might be needed if you are running in ARM environment.
``` shell
## API GW
#docker pull cp.icr.io/cp/webmethods/api/api-gateway:11.1 --platform linux/amd64

docker pull cp.icr.io/cp/webmethods/api/api-gateway:11.1 
docker tag cp.icr.io/cp/webmethods/api/api-gateway:11.1 default-route-openshift-image-registry.apps-crc.testing/webmethods/api-gateway:11.1 
docker push default-route-openshift-image-registry.apps-crc.testing/webmethods/api-gateway:11.1 

## MSR
#docker pull cp.icr.io/cp/webmethods/integration/webmethods-microservicesruntime:11.1.0.0 --platform linux/amd64
docker pull cp.icr.io/cp/webmethods/integration/webmethods-microservicesruntime:11.1.0.0
docker tag cp.icr.io/cp/webmethods/integration/webmethods-microservicesruntime:11.1.0.0 default-route-openshift-image-registry.apps-crc.testing/webmethods/webmethods-microservicesruntime:11.1.0.0
docker push default-route-openshift-image-registry.apps-crc.testing/webmethods/webmethods-microservicesruntime:11.1.0.0

## MSR for helm
docker pull sagcr.azurecr.io/webmethods-microservicesruntime:10.15
docker tag sagcr.azurecr.io/webmethods-microservicesruntime:10.15 default-route-openshift-image-registry.apps-crc.testing/webmethods/webmethods-microservicesruntime:10.15
docker push default-route-openshift-image-registry.apps-crc.testing/webmethods/webmethods-microservicesruntime:10.15

# Elasticsearch
#docker pull docker.elastic.co/elasticsearch/elasticsearch:8.2.3 --platform linux/amd64
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.2.3 
docker tag docker.elastic.co/elasticsearch/elasticsearch:8.2.3 default-route-openshift-image-registry.apps-crc.testing/webmethods/elasticsearch:8.2.3
docker push default-route-openshift-image-registry.apps-crc.testing/webmethods/elasticsearch:8.2.3
```
## Deploy using ArgoCD
1. Install Marketplace capability if not installed
    * Check `oc get clusterversion version -o jsonpath='{.spec.capabilities}{"\n"}{.status.capabilities}{"\n"}'`
    * Install `oc patch clusterversion/version --type merge -p '{"spec":{"capabilities":{"additionalEnabledCapabilities":["marketplace"]}}}'`
    * You might need to restart the pods for changes to reflect `oc delete pods --all -n openshift-marketplace`
2. Install GitOps Operator from Operator Hub (Default Installation)
3. Give ArgoCD permissions on `webmethods` namespace(project) `kubectl apply argo-config/argocd-config.yaml`
4. Get ArgoCD admin password `kubectl get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d`
5. Apply all configurations in the `argo-config` folder `kubectl apply -f argo-config/`

## Deploy K8s YAML
1. Run the following commands
    ```shell
    kubectl apply -f app/wm-poc-k8s.yaml 
    kubectl apply -f app/wm-poc-k8s-ingress.yaml 
    ```
2. Add the following to hosts file. For Unix Like `/etc/hosts`
    ```
    127.0.0.1        wm-msr.apps-crc.testing wm-gw.apps-crc.testing
    ```
3. Open URLs 
    1. [http://wm-msr.apps-crc.testing:5555/](http://wm-msr.apps-crc.testing:5555/)
    2. [http://wm-gw.apps-crc.testing:9072/](http://wm-gw.apps-crc.testing:9072/)

---
## More Resources 
* Debugging CRC https://github.com/crc-org/engineering-docs/blob/main/content/Debugging.md
* Deleting and starting CRC 
    ``` shell
    crc delete --force
    crc cleanup

    crc config view
    crc setup

    crc start
    ```
* Adding insecure registry `vi /etc/docker/daemon.json`
    ``` json
    {
        "insecure-registries" : [ "default-route-openshift-image-registry.apps-crc.testing:443" ]
    }
    ```
* Connecting to SSH 
    ``` shell
    ssh -i ~/.crc/machines/crc/id_ed25519 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -p 2222 core@127.0.0.1
    ```
<!-- -------------------
eval $(crc oc-env)
oc new-project webmethods 

oc whoami
# podman login --tls-verify=false -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps-crc.testing
docker login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps-crc.testing

vi /etc/docker/daemon.json
{
    "insecure-registries" : [ "default-route-openshift-image-registry.apps-crc.testing:443" ]
}

# Add to trusted keychain certificates


----------------------------

hosts file 
sudo vi /etc/hosts
127.0.0.1 	local-wm.io 

================================================
oc registry login --insecure=true


podman machine ssh -- 'echo "192.168.127.254 default-route-openshift-image-registry.apps-crc.testing" | sudo tee -a  /etc/hosts'

# Add as insecure registry in podman (Edit Password and Login)
podman machine ssh --username root
vi /etc/containers/registries.conf

[[registry]]
location = "default-route-openshift-image-registry.apps-crc.testing"
insecure = true


`podman machine ssh -- "cd /etc/pki/ca-trust/source/anchors && openssl s_client -showcerts -connect default-route-openshift-image-registry.apps-crc.testing:443 </dev/null 2>/dev/null | openssl x509 -outform PEM > openshift.pem && update-ca-trust"`
 -->
