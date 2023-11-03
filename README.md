# gitlab-ci-monitoring-helm-chart

## Prerequisites

- Make sure the Kubernetes CLI tool kubectl is installed on your local machine - [kubctl1.24](https://docs.aws.amazon.com/zh_tw/eks/latest/userguide/install-kubectl.html "kubctl1.24")

- Make sure the Helm client is installed on your local machine - [Helm3](https://helm.sh/ "helm3 documentation")

- Make sure you have the access to deploy helm chart to the Kubernetes Cluster - [aws sso config](https://www.notion.so/bluext/How-to-configure-AWS-SSO-to-access-EKS-b1aa4469be624bd78014dff1195ddf66?pvs=4 "aws sso configuration")

- Make sure the Gitlab API token has already been deployed to the Kubernetes secrets
  ```
  kubectl create secret generic SECRET_NAME \
  --from-literal=token="xxxxx" \
  -n YOUR_NAMESPACE
  ```
- [Optional] Make sure the .dockerconfigjson has already been deployed to the Kubernetes secrets. (if the docker registry is private) - *for local development*
  ```
  {
    "auths": {
      "docker.io": {
        "username":"YOUR_DOCKER_USERNAME",
        "password":"YOUR_DOCKER_PASSWORD",
        "email":"YOUR_REGISTRATION_EMAIL"
        "auth":"YOUR_PASSWORD_IN_BASE64_FORMAT"
      }
    }
  }
  ```

- Make sure the Grafana adminUser and adminPassword has already been deployed to the Kubernetes secrets within your target namespace
  ```
  kubectl create secret generic prometheus-grafana \
  --from-literal=admin-user="xxxxx" \
  --from-literal=admin-password="xxxx" \
  -n YOUR_NAMESPACE
  ```
## Installations

1. Create / Choose your own namespace in EKS cluster if needed.

2. Use below CMD to deploy the package you've created toward the namespace. If the release has exist, it will use upgrade, or it will install a new releas instead.
   ```
   helm --debug upgrade -n NAMESPACE -i RELEASE_NAME ./ -f ./deploy-env/YOUR_CUSTOM_DEPLOYMENT_YAML
   ```

## Redeployment Instruction to another namespace
### PersistenVolumeClaim Rebound (From Source namespace to Target namespace)
1. Checkout Reclaim Policy, Status and Claim attributes within  PersistentVolume (PV) of Grafana and Prometheus
    ```console
    kubectl get pv PERSISTENT_VOLUME_NAME
    ```
2. Make sure the PV **Reclaim Policy** has been changed from Delete to **Retain**. If not, below cmd is to keep pv from being delete after removing PersistenVolumeClaim (PVC)
    ```console
    kubectl patch pv PERSISTENT_VOLUME_NAME -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
    ```
3. Remove the old PVC in source namespace to release PV
    ```console
    kubectl delete pvc PERSISTENT_VOLUME_CLAIM_NAME -n SOURCE_NAMESPACE
    ```
4. Delete entire block of spec.claimRef in PV (Notice: Only claimRef!!! And it would be recreated after you bound new PVC in target namespace)
    ```console
    kubectl edit pv PERSISTENT_VOLUME_NAME (press i and delete whole claimRef, then type :wq to store the edit)
    ```
5. Back to the helm chart you want to deploy, make sure you have specify the pv you plan to bound in **deploy-env/custom-values.yaml**.
*Notice: Grafana and Prometheus use different pv, 
    ```console
    grafana:
      persistence:
        type: pvc
        enabled: true
        accessModes:
          - ReadWriteOnce
        ...
        volumeName: GRAFANA_PERSISTENT_VOLUME_NAME
    
    prometheus:
      server:
        volumeName: PROMETHEUS_PERSISTENT_VOLUME_NAME
    ```
6. Make sure you've added below configuration in PVC annotation section within **deploy-env/custom-values.yaml** for Grafana and Prometheus respectively. 
*This setting could keep resource being deleted after upgrading or delete release*
    ```console
    annotations: 
        "helm.sh/resource-policy": keep
    ```

7. Install your new helm chart to target namespace
    ```console
    helm --debug upgrade -n NAMESPACE -i RELEASE_NAME ./ -f ./deploy-env/YOUR_CUSTOM_DEPLOYMENT_YAML
    ```
8. After successfully installing the Helm chart and binding the new PVC to the PV, remove the 'volumeName' you had specified and now specify the new PVC in the 'existingClaim' section of the **deploy-env/custom-values.yaml** file.

9. Now you can confidently upgrade your helm chart
    ```console
    helm --debug upgrade -n NAMESPACE -i RELEASE_NAME ./ -f ./deploy-env/YOUR_CUSTOM_DEPLOYMENT_YAML
    ```
## Uninstall Chart

```console
helm uninstall [RELEASE_NAME]
```

This removes all the Kubernetes components associated with the chart and deletes the release.

_See [helm uninstall](https://helm.sh/docs/helm/helm_uninstall/) for command documentation._