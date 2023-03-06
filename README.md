# ocp-gitops-argocd-with-external-secrets-operator

This repository collects information about implementing a secret management strategy based on GitOps and the  .

External Secrets Operator is a Kubernetes operator that integrates external secret management systems like AWS Secrets Manager, HashiCorp Vault, Google Secrets Manager, Azure Key Vault, IBM Cloud Secrets Manager, and many more. The operator reads information from external APIs and automatically injects the values into a Kubernetes Secret.

## Prerequisites

- Openshift +4.12
- Oc CLI +4.12 - [Official Doc](https://docs.openshift.com/container-platform/4.12/cli_reference/openshift_cli/getting-started-cli.html)
- AWS Secrets Manager Account

## Setting Up

This section includes a set of procedures to set up the External Secrets Operator and create a first protected secret in AWS Secrets Manager and Openshift.

### AWS Secret

It is required to create a secret in AWS in order to store the private information that had to be used in Openshift. For this reason, it is necessary to execute the following command:

```$bash
aws secretsmanager create-secret \
     --name openshift-mysecret01 \
     --secret-string securedinformation \
     --region us-east-2
```

Once the command is executed, it is possible to access this information through the AWS console.

### External Secrets Operator

First of all, it is required to install the External Secrets Operator following the next procedure:

- Install the operator and its configuration
  
```$bash
oc apply -f openshift/00-externalsecretsoperator-subs.yaml
oc apply -f openshift/00-externalsecretsoperator-conf.yaml
```

- Check the respective pods

```$bash
oc get pods -n openshift-operators
NAME 
...                                                             READY   STATUS    RESTARTS   AGE
cluster-external-secrets-58f5c4958d-4mnqj                       1/1     Running   0          2m27s
cluster-external-secrets-cert-controller-59c79789d6-hc5bl       1/1     Running   0          2m27s
cluster-external-secrets-webhook-754976f868-2zk56               1/1     Running   0          2m27s
external-secrets-operator-controller-manager-85945bfc57-7xg22   1/1     Running   1          2m43s
```

Once the Operator is installed, it is time to create the respective resources in order to be able to synchronize secrets from an external provider. Please follow the next steps to configure the AWS Secrets Manager:

- Create a specific secret with the AWS Secrets Manager Credentials

```$bash
oc new-project external-secrets
echo -n 'KEYID' > ./access-key
echo -n 'SECRETKEY' > ./secret-access-key
oc create secret generic awssm-secret --from-file=./access-key --from-file=./secret-access-key -n external-secrets
```

### External Secrets Objects

Installing the operator and creating the different secrets in AWS Secrets Manager is the first step of the process. Once the previous requirement is met, it is time to start working with AWS secrets.

First of all, it is required to create a *Secret Storage*. The idea behind the SecretStore resource is to separate concerns of authentication/access and the actual Secret and configuration needed for workloads. The ExternalSecret specifies what to fetch, the SecretStore specifies how to access. This resource is namespaced.

- Create a *Secret Storage* (*edit region if it is required*)

```$bash
oc apply -f openshift/01-secretstorage.yaml -n external-secrets
```

Once the *Secret Storage* is created, it is required to create the respective *External Secret* object that declares what data to fetch. It has a reference to a SecretStore which knows how to access that data. The controller uses that ExternalSecret as a blueprint to create secrets.

With this in mind, it is required to create an *External Secret* per AWS secret that it is required to obtain following the next procedure:

- Create the respective *External Secret* in order to get the secret from AWS (*Edit any information required*)

```$bash
oc apply -f openshift/02-externalsecret.yaml -n external-secrets
```

At this moment, it is possible to access the respective information in Openshift. The following procedure includes a set of steps to ensure everything is working properly:

- Check the *Secret Storage*

```$bash
oc describe SecretStore secretstore-aws 
...
Events:
  Type    Reason  Age                From          Message
  ----    ------  ----               ----          -------
  Normal  Valid   21s (x8 over 25m)  secret-store  store validated
```

- Check the *External Secret*

```$bash
oc describe ExternalSecret aws-openshift-mysecret01
...
Events:
  Type    Reason   Age                   From              Message
  ----    ------   ----                  ----              -------
  Normal  Updated  32s (x10 over 8m33s)  external-secrets  Updated Secret
```

- Check the final secret in Openshift

```$bash
oc get secret aws-openshift-mysecret01 -o yaml
...
data:
  privatedata: c2VjdXJlZGluZm9ybWF0aW9u

oc extract secret/aws-openshift-mysecret01 --to=-
# privatedata
securedinformation
```

## Operations

There are multiple operations and situations that it could be possible to assume during the External Secrets solutions lifecycle. The following sections include some use cases or hypothetical situations with the respective procedures to face them.

### Identify Error Decrypting External Secret Objects

After rotate certificates, it is possible to find errors if the *SealedSecret* objects are not modified. Execute the following procedure to detect these errors: 

- Create an *External Secret* with errors, for example referencing a secret that does not exist

```$bash
oc apply -f examples/externalsecret-error.yaml
```

- Review the *External Secret* object

```$bash
oc get ExternalSecret aws-openshift-error
NAME                  STORE             REFRESH INTERVAL   STATUS              READY
aws-openshift-error   secretstore-aws   1m                 SecretSyncedError   False
```

- Review the *External Secret* manager logs

```$bash
POD=$(oc get pods --no-headers -o custom-columns=":metadata.name" -l app.kubernetes.io/name=external-secrets -n openshift-operators)
oc logs $POD -n openshift-operators
...
{"level":"info","ts":1678135254.75357,"logger":"provider.aws","msg":"using aws session","region":"us-east-2","credentials":{}}
{"level":"info","ts":1678135254.7536502,"logger":"provider.aws.secretsmanager","msg":"fetching secret value","key":"openshift-mysecret-error","version":"AWSCURRENT"}
{"level":"error","ts":1678135254.760182,"logger":"controllers.ExternalSecret","msg":"could not get secret data from provider","ExternalSecret":"openshift-operators/aws-openshift-error","error":"Secret does not exist","stacktrace":"github.com/external-secrets/external-secrets/pkg/controllers/externalsecret.(*Reconciler).Reconcile\n\t/home/runner/work/external-secrets/external-secrets/pkg/controllers/externalsecret/externalsecret_controller.go:190\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Reconcile\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.14.1/pkg/internal/controller/controller.go:122\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.14.1/pkg/internal/controller/controller.go:323\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.14.1/pkg/internal/controller/controller.go:274\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Start.func2.2\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.14.1/pkg/internal/controller/controller.go:235"}
```

### Modifying Created External Secrets

Secrets lifecycle does not only include creation and deletion, modification operations should be performed from time to time in order to rotate or change values. External Secrets and the *refreshInterval* parameter included in the *External Secret* object allow user to not be concerned about changes in Openshift when the source of trust change. External Secrets manager identifies the changes and synchronizes the data automatically.

In order to test this functionality, it is possible execute to execute the following procedure:

- Modify the secret in AWS

```$bash
aws secretsmanager update-secret \
     --secret-id openshift-mysecret01 \
     --secret-string securedinformationrotated \
     --region us-east-2
```

- Check the *External Secret*

```$bash
oc describe ExternalSecret aws-openshift-mysecret01
...
Status:
  Conditions:
    Last Transition Time:   2023-03-06T20:13:44Z
    Message:                Secret was synced
    Reason:                 SecretSynced
    Status:                 True
    Type:                   Ready
  Refresh Time:             2023-03-06T21:28:48Z                       <-------------------------- The time when the secret refresh was performed
  Synced Resource Version:  1-5a8b4f603c97cc2d4647d916b6858467
Events:
  Type    Reason   Age                   From              Message
  ----    ------   ----                  ----              -------
  Normal  Updated  32s (x10 over 8m33s)  external-secrets  Updated Secret
```

- Check the final secret in Openshift

```$bash
oc extract secret/aws-openshift-mysecret01 --to=-
# privatedata
securedinformationrotated
```

## Author

Asier Cidon @RedHat
