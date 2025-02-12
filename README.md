# Vaultwarden-Mariadb Kubernetes Deployment

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) is an alternative implementation of the Bitwarden server API, written in Rust and fully compatible with [upstream Bitwarden clients](https://bitwarden.com/download/). 

This Kustomization manifest can be used to deploy a stable Vaultwarden/MariaDB configuration to a Kubernetes cluster.

vaultwarden version: 1.33.1 - mariadb version: 11.4.4

## Prerequisites

Before deploying Vaultwarden, ensure that your Kubernetes environment meets the following requirements:

- **Kubernetes Cluster:** A functional Kubernetes cluster where you can deploy your applications.
- **Kubectl:** The `kubectl` command-line tool must be installed on your deployment system and configured to interact with your Kubernetes cluster.
- **cert-manager:** Ensure that `cert-manager` is installed on your Kubernetes cluster and a certificate issuer is properly configured.
- **NGINX Ingress Controller:** The NGINX Ingress controller must be installed on the cluster to manage HTTP(S) traffic routing.
- **Storage Class:** A valid storage class must be installed on the cluster to handle persistent storage needs.

## Configuration steps

### Install and Configure Cert-Manager 

Configure a ssl certificate, you can accomplish this task using [cert-manager](https://cert-manager.io/docs/installation/).
After install cert manager create a ClusterIssuer and point it as a issuerRef on the manifest ssl-cert.yaml

```yaml
## Edit ssl-cert.yaml 
## Using letsencrypt-issuer issuer by default, please replace with a issuer name available in your cluster.

  issuerRef:
    name: letsencrypt-issuer

## Also modify <URL_DOMAIN> and provide the dns name used to access the vaultwarden app.
## Do not use url format, just the complete domain name.
## example vaultwarden.mydomain.com

    commonName: <URL_DOMAIN>
    dnsNames:
        - <URL_DOMAIN>

```

***Optional:*** A sample configuration YAML manifest for a self-signed cluster issuer is provided. You can use this to generate and deploy a self-signed certificate to your cluster.

```yaml
kubectl apply -f opt/self-signed-issuer.yaml 

```

Then use 'selfsigned-cluster-issuer' as the cluster issuer on ssl-cert.yaml.


### Deploy and Configure Nginx Ingress controller

```sh
## use helm chart to deploy nginx ingress 
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 
# list all versions availables
helm search repo ingress-nginx --versions

# Deploy a version compatible with your kubernetes server version, listed here https://github.com/kubernetes/ingress-nginx
helm upgrade --install ingress-nginx  --version 4.12.0  --namespace ingress-nginx --create-namespace

## or just install the latest version
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace

```

Edit the manifest **ingress.yaml** replace the **<URL_DOMAIN>** by your vaultwarden domain, same dns name used in ssl-cert.yaml manifest config.  


### Storage class

Modify the storage class on manifest config file **storage.yaml** ian use a storage class installed on your Kubernetes cluster.

```yaml
# Just set the sc on the following lines on storage.yaml file on both claims mysql-pv-claim and vaultwarden-pv-claim

  storageClassName: your_storage_class

```

By default we have set the sotrage to Dynamic Provisioning of Kubernetes HostPath Volumes [Information here](https://github.com/rimusz/hostpath-provisioner)

```sh
# install dynamic hostpath provisioner Helm chart
helm repo add rimusz https://charts.rimusz.net
helm repo update
helm upgrade --install hostpath-provisioner --namespace kube-system rimusz/hostpath-provisioner

```

## Kustomization variables

```yaml
## kustomization.yaml file secrets configuration
## Database User Name used by vaultwarden, default: admin     
- username=<my_user>

## Database Password used by vaultwarden, also the Mariadb root user password. default: mypassword
- password=<my_password>

## Database connection string used by vaultwarden application, username and password use the same defined above default: mysql://admin:mypassword@vaultwarden-mysql:3306/vaultwarden
- DATABASEURL=mysql://<my_user>:<my_password>@vaultwarden-mysql:3306/vaultwarden

## Token to access the vaultwarden administration section, you can generate with 'openssl rand -base64 48' 
- admin-token=<token> 

```


## Deploying the Manifest

After editing the following configuration files (**ssl-cert.yaml**, **ingress.yaml**, **storage.yaml**, and **kustomization.yaml**) with your specific cluster details, it's time to deploy the application. 

You can deploy and review your application using the following commands:

```sh
# Deploy the application
kubectl apply -k deployment/

# Verify the deployed objects
kubectl get all -n vaultwarden

# Check the status of all running pods
kubectl get po -n vaultwarden

# Review the logs of a running pod (use the pod name from the previous command)
kubectl logs <pod-name> -n vaultwarden

# Remove the entire deployment (if needed)
kubectl delete -k deployment/
```

### Notes:

- Replace `<pod-name>` in the `kubectl logs` command with the actual name of the pod, which you can obtain from the `kubectl get po` command.
- Ensure that the deployment manifests are properly configured before applying them to your cluster.


## After deployment 

If your deployment is successful and free of errors, you should be able to access your Vaultwarden instance by navigating to the DNS URL you configured within your ingress and certificate manifests. For example: `https://vaultwarden.domain.com`.

Upon reviewing the logs of the running pod, you may encounter the following message:

> [NOTICE] You are using a plain text `ADMIN_TOKEN`, which is insecure.  
> Please generate a secure Argon2 PHC string by using `vaultwarden hash` or `argon2`.

This warning occurs because Vaultwarden defaults to a static token for administrative access, which is not secure. To address this, follow the official documentation [here](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page#secure-the-admin_token) to generate a secure token. Once you've done that, update the token in the administration section, accessible at `https://vaultwarden.domain.com/admin`.

Additionally, be sure to configure your domain URL and SMTP settings to enable Vaultwarden to send notifications.

---

### Congratulations! 🎉  
You've successfully completed the setup of your Vaultwarden server.

I hope this guide and the provided code have been helpful to you. Feel free to reach out with any comments, suggestions, or questions you may have. If you'd like to support my work, please consider [making a donation](https://www.paypal.com/donate/?hosted_button_id=H34CRN93Z259J).
