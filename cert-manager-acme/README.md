

## Description
* This repo provides kubernetes YAML configuration files and steps to both install Cert Manager as well as configure a sample web application and obtain/install a certificate from the ACME v2 compliant server such as Let's Encrypt.

## Disclaimer
* The configuration and steps are verified as of May 2018, your mileage may vary.

* `kubectl version`

  ```
  Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-03-27T00:14:31Z", GoVersion:"go1.9.4", Compiler:"gc", Platform:"darwin/amd64"}
  Server Version: version.Info{Major:"1", Minor:"8+", GitVersion:"v1.8.8-gke.0", GitCommit:"6e5b33a290a99c067003632e0fd6be0ead48b233", GitTreeState:"clean", BuildDate:"2018-02-16T18:26:58Z", GoVersion:"go1.8.3b4", Compiler:"gc", Platform:"linux/amd64"}
  ```  


* Cert Manager version: `v0.3.0-alpha.2`


## Resources

* [Cert Manager](https://github.com/jetstack/cert-manager)
* [gke-letsencrypt](https://github.com/ahmetb/gke-letsencrypt)
  * Configuration files are originally coming from this repo.
  * The difference is that the step to reserve a global static IP address is not necessary.


## Create GKE Cluster

* Use either GCP console or gcloud to create a kubernetes cluster

## Create ClusterRoleBinding

* Execute `kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=your-email@address-here`

  * name `cluster-admin-binding` can be any of your choice.   

* This is needed due to an [issue](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control) in GKE

  >Because of the way Kubernetes Engine checks permissions when you create a Role or ClusterRole, you must first create a RoleBinding that grants you all of the permissions included in the role you want to create.

  >An example workaround is to create a RoleBinding that gives your Google identity a cluster-admin role before attempting to create additional Role or ClusterRole permissions.

  >This is a known issue in RBAC in Kubernetes and Kubernetes Engine versions 1.6 and later.


## Install Cert Manager

As of May 2018, Cert Manager's support for ACME v2 is still in [alpha release](https://github.com/jetstack/cert-manager/releases/tag/v0.3.0-alpha.2), you will have to install Cert Manager in a slightly different way.

* Execute `git clone https://github.com/jetstack/cert-manager.git`
* Go to `contrib/manifests/cert-manager/rbac` directory
* Execute `kubectl apply -f .` to install Cert Manager v0.3 version



## Prepare YAML Configuration Files

* Edit `issuer.yml` file to provide an email address.
* Edit `ingress-tlw.yml` file to provide your domain name such as `www.example.com`.
* Edit `certificate.yml` file to provide your domain name such as `www.example.com`


## Create Resources
* Execute `kubectl apply -f sample-app.yml` to create `Deployment` and `Service` resources for the sample `hello-app` application.
* Execute `kubectl apply -f ingress.yml` to create `Ingress Controller` resource.
* Execute `kubectl apply -f issuer.yml` to create `Cluster Issuer` resource.

  * Status can be checked by `kubectl describe -f issuer.yml`, you can see something similar like the following


```
Name:         acme-client-cluster-issuer
Namespace:
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"certmanager.k8s.io/v1alpha1","kind":"ClusterIssuer","metadata":{"annotations":{},"name":"acme-client-cluster-issuer","namespace":""},"sp...
API Version:  certmanager.k8s.io/v1alpha1
Kind:         ClusterIssuer
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-05-05T03:04:34Z
  Generation:          0
  Initializers:        <nil>
  Resource Version:    3659
  Self Link:           /apis/certmanager.k8s.io/v1alpha1/acme-client-cluster-issuer
  UID:                 09be6a40-5011-11e8-8967-42010a800104
Spec:
  Acme:
    Email:  001@def.com
    Http 01:
    Private Key Secret Ref:
      Key:
      Name:  acme-client-private-key
    Server:  https://{acme-v2-server-url}}/directory
Status:
  Acme:
    Uri:  https://{acme-v2-server-url}}/acct/7
  Conditions:
    Last Transition Time:  2018-05-05T03:04:34Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
  ```


* Update DNS Record Mapping For IP Address
  * For the domain name you are requesting certificate for, point DNS to the static IP address.
  * You can find out the IP address by `gcloud compute addresses list --global`


* Execute `kubectl apply -f certificate.yaml` to create certificate.

  * Status can be checked by `kubectl describe -f certificate.yml`, you can see something similar like the following


```
Name:         acme-client-certificate
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"certmanager.k8s.io/v1alpha1","kind":"Certificate","metadata":{"annotations":{},"name":"acme-client-certificate","namespace":"default"},"...
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Certificate
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-05-05T14:11:52Z
  Generation:          0
  Initializers:        <nil>
  Resource Version:    5560
  Self Link:           /apis/certmanager.k8s.io/v1alpha1/namespaces/default/certificates/acme-client-certificate
  UID:                 426859bc-506e-11e8-b45b-42010a8000d0
Spec:
  Acme:
    Config:
      Domains:
        www.{your-domain-name}.com
      Http 01:
        Ingress:  acme-client-ingress
  Common Name:    www.{your-domain-name}.com
  Dns Names:
    www.{your-domain-name}.com
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       acme-client-cluster-issuer
  Secret Name:  acme-client-secret
Status:
  Acme:
    Order:
      Challenges:
        Authz URL:  https://{acme-v2-server-url}/authz/33
        Domain:     www.{your-domain-name}.com
        Http 01:
          Ingress:  acme-client-ingress
        Key:        738bdb96118d4881aa998163a39b7d2e.zgMQJsUbQxDQEaM-r2EKHXMhoVY2gcZa3HrblJyftlo
        Token:      738bdb96118d4881aa998163a39b7d2e
        Type:       http-01
        URL:        https://{acme-v2-server-url}/authz/33/33
        Wildcard:   false
      URL:          https://{acme-v2-server-url}/order/33
  Conditions:
    Last Transition Time:  2018-05-05T14:36:56Z
    Message:               Certificate issued successfully
    Reason:                CertIssued
    Status:                True
    Type:                  Ready
Events:
  Type    Reason          Age   From          Message
  ----    ------          ----  ----          -------
  Normal  CreateOrder     25m   cert-manager  Created new ACME order, attempting validation...
  Normal  DomainVerified  1m    cert-manager  Domain "www.{your-domain-name}.com" verified with "http-01" validation
  Normal  IssueCert       59s   cert-manager  Issuing certificate...
  Normal  CertObtained    58s   cert-manager  Obtained certificate from ACME server
  Normal  CertIssued      58s   cert-manager  Certificate issued successfully
```  

  * Wait until you see the `Certificate issued successfully` message

* Execute `kubectl apply -f ingress-tls.yml` to add the certificate to the load balancer.
  * The additional change compare to `ingress.yml` is the `tls` section to the `Ingress Controller`
  * This may take a while to take effect.

* Visit your domain using HTTPS to verify the certificate.


## Clean Up

* Execute `kubectl delete -f .` to delete all resources
* Additionally, delete the GKE cluster
