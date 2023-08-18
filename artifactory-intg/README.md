# Install Admission Controller and  Integrate Hosted Artifactory with Lacework

Stept to deploy lacework Admission Controller and leverage the proxy scanner (deployed togather with admission controller) to integrate hosted artifactory with lacework for container vulnerability scanning. 

This guilde will use precreated kubernetes secrets for certificate, lacework API tokens and registry access credentials

## 1. Create secrets for credentials
### create secret for lacework token

- [Create Proxy Scanner Integration in Lacework](https://docs.lacework.net/onboarding/integrate-proxy-scanner#create-a-proxy-scanner-integration-in-lacework)
- Create k8s secret - [example yaml](./lw-proxyscanner-token.yaml)

### create secret for certificate to be used by Admission Controller
- create secret from cert (follow this sample, use default Opaque type not tls type) - [example yaml](./cert-secret.yaml)
```
$ kubectl apply -f cert-secret.yaml
```

### create secret for registries which proxy scanner will pull images from
- create a yaml file with all registries which proxy scanner will retrieve images for scanning exmaple [here](./registries.yaml) 

   > **_NOTE:_**  For your use case, make sure artifactory access information and credentials are added in this format and saved in vault. 

- Base64 encode the yaml content
```
$ cat registries.yaml | base64  
```
- use encoded content to create secret [example](./registries-secret.yaml)
```
$ kubectl apply -f registries-secret.yaml
```

## 2. Deploy admission controller and proxy scanner

Make sure to update [helm values](./helm-values.yaml) to refer to the correct secrets and lacework account.
```
helm upgrade --install --create-namespace --namespace lacework \
--values helm-values.yaml \
lacework-admission-controller lacework/admission-controller
```


## 3. Expose proxy scanner to be internet accessible
I used kubernetes [Ingress](./proxy-scanner-ingress.yaml) to expose proxy scanner. You can use any way of you choice (eg. Load Balancer etc)
```
$ kubectl apply -f proxy-scanner-ingress.yaml
```
verify by checking if pods are running 
```
$ kubectl get pod -n lacework
NAME                                             READY   STATUS    RESTARTS   AGE
lacework-admission-controller-6dcdbdc6fb-mh7lq   1/1     Running   0          14m
lacework-proxy-scanner-f9cbb96b4-mwdqj           1/1     Running   0          14m
```

## 4. Configure Artifactory webhook
Follow this [documentation](https://docs.lacework.net/onboarding/integrate-proxy-scanner-with-jfrog-registry#configure-the-jfrog-registry-webhook-for-optional-notifications) to create webhook in Artifactory

> **_NOTE:_**  In my case, the URL invoked by webhook is {my ingress controller domain}/v1/notification?registry_name=kwan-jfrog. Make sure formatting correct URL.