# Atlassian Bamboo and Bitbucket images for GKE clusters

Atlassian is an enterprise software company that develops products are widely used by enterprise customers
Atlassian products cover a  wide range of products, but here we're gonna work with:
* Bitbucket: a web based source code and project management solution
* Bamboo: a continuous integration server which integrates with Bitbucket.


## Requisites

You'll need an Atlassian account. If you need an account, go [here](https://id.atlassian.com/signup)

An GKE cluster able to run both software. See Atlassian requirements about that. 

## Install the Atlassian software 

We're going to expose the services as service type `LoadBalancer` 

To run Bitbucket, execute:
```
# Create the server
$ kubectl create -f https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bitbucket/deployment.yaml
# Expose the web app on port 80 and git on port 7990
$ kubectl create -f  https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bitbucket/svc.yml
```

To run Bamboo, execute:
```
# Create the server 
$ kubectl create -f https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bamboo/deployment.yaml
# Create the secret with the GKE ServiceAccount to configure kubecfg
$ kubectl create -f https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bamboo/secret.yaml
# Expose the web app on port 80
$ kubectl create -f https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bamboo/svc.yml
```

You'll need a pair of minutes while the pods and load balancer gets up and running. 

## Configure Bitbucket
```
# Use this command to find the Bitbucket public URL 
$ echo "http://$(kubectl get svc atlassian-bitbucket  -o jsonpath={.status.loadBalancer.ingress[].ip}).xip.io"
```

Point your browser to the addresses, this will correspond to Bitbucket service.


* Click 'Next' and select 'I have a Bitbucket license key'. Introduce your Bitbucket license key. 
* Click 'Next' and introduce the username, Full name, email address and password to create an admin user.
* Click 'Go to Bitbuket'. Once done, you'll be redirected to the Login page. 
* Introduce your admin login.
* Click on 'Projects' and create a project called 'Kubecfg demo' , use 'a kubecfg demo' as 'Description' and click on 'Create project'
* Click on 'Import repository'  and click on the 'Git' icon and introduce 'https://github.com/jbianquetti-nami/kubecfg-nonroot-nginx' as URL. Click and 'Import repository'
* Clone the repository to your elsewhere in your disk. 


## Configure Bamboo

```
# Use this command to find the Bamboo public URL 
$ echo "http://$(kubectl get svc atlassian-bamboo -o jsonpath={.status.loadBalancer.ingress[].ip}).xip.io"
```


* Introduce your Bamboo license key and click on 'Custom install'. 
* Introduce the same user you've created for Bitbucket. 
* Wait a bit while Bamboo is being configured. Keep sure that your cluster meets the requirements of Bamboo to be installed. Otherwhise your install process can be stalled 


Bamboo image includes a kubectl proxy listen on port 8001 as a sidecar container. This means that you can interact with your cluster using kubecfg commands.

# Configure kubecfg with GKE 

To be able to deploy your workloads on GKE you will need to grant access to your cluster. To do so, we're going to create a Service Account. A service account is very similar to an IAM group on AWS. You can create it in: https://console.cloud.google.com/iam-admin/serviceaccounts/

Do not forget to click on `Furnish a new key as JSON file` and store it securely. The contents of the file will going to be used as a secret. To do so, you need to encode as base64 and paste in the file. Use [this file](https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bamboo/secret.yaml) as template.

```
apiVersion: v1
kind: Secret
metadata:
  name: gcloud-svc-account
data:
  gcloud-cluster: --- Your GKE cluster name, BASE64 ENCODED ---
  gcloud-cluster-zone: --- Your GKE cluster zone , BASE64 ENCODED --- 
  gcloud-svc-account: --- CONTENTS OF THE JSON SVC ACCOUNT, BASE64 ENCODED --
```

The secret will be use to create a config file to be used by kubecfg. By this way you're authorized to run kubectl/kubecfg commands 

