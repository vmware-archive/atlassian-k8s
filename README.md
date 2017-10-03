# Atlassian Bamboo and Bitbucket images for GKE clusters

Atlassian is an enterprise software company that develops products are widely used by enterprise customers
Atlassian products cover a  wide range of products, but here we're gonna work with:
* Bitbucket: a web based source code and project management solution
* Bamboo: a continuous integration server which integrates with Bitbucket.


## Requisites

You'll need an Atlassian account. If you need an account, go [here](https://id.atlassian.com/signup). The account is required to obtain an evaluation license. If you don't own a license, you can request an evaluation license from the set up page of each product directly.

An GKE cluster able to run both software. We recommend a cluster with at least one n1-standard-2 machine. You can launch your cluster with this command: machine. You can launch your cluster with this command: 
`gcloud alpha container clusters create $NAME --num-nodes=1 --enable-autoscaling --min-nodes=1 --max-nodes=3 --no-enable-cloud-logging -m n1-standard-2` 


See Atlassian requirements about that: 

* https://confluence.atlassian.com/bamboo/bamboo-best-practice-system-requirements-388401170.html#BambooBestPractice-SystemRequirements-CPUandmemory

* https://confluence.atlassian.com/bitbucketserver/scaling-bitbucket-server-776640073.html

# Configure kubecfg with GKE 

To be able to deploy your workloads on GKE you will need to grant access to your cluster. To do so, we're going to create a Service Account, with the role type Container Engine Developer . A service account is very similar to an IAM group on AWS. You can create it in: https://console.cloud.google.com/iam-admin/serviceaccounts/

Depending of your situation, you may need to enable the Container Engine API on your GCP project, too. 

Do not forget to click on `Furnish a new key as JSON file` and store it securely. The contents of the file will going to be used as a secret. To do so, you need to encode as base64 and paste in the file.

Obtain the secret template:
```
wget https://raw.githubusercontent.com/jbianquetti-nami/atlassian-k8s/master/bamboo/secret.yaml
```
Then edit the file to adapt it to your environment:
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
Use a command like this to base64 encode your data:
```
echo -n somevalue | base64 
```

The secret will be use to create a config file to be used by kubecfg. By this way you're authorized to run kubectl/kubecfg commands 

Create the secret in your cluster:
```
kubectl create -f secret.yaml
```


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




## Useful tips for debugging 
If you face some issues trying to reproduce this steps, here's a few tips to look:
* Bamboo requires a good amount of resources to run (see #Requisites section), so if you find that your Bamboo install process is stalled, try to use a n1-standard-4 machine
* Check that your Atlassian licenses are still valid. They are valid for 30 days.
* Check that your GKE service account is proper configured, and counts with at least the Container Engine Developer role access level.
* Remember that values in a Kubernetes secret are Base64 enconded!
* If you see this message `invalid configuration: default cluster has no server defined` means that your service account data is not correct. To figure out which is the problem, execute:
```
# Access to the Bamboo pod 
$ kubectl exec -it atlassian-bamboo-xxxxxx-xxxx  -c bamboo  bash
# Now, in the pod execute:
export CLOUDSDK_CONFIG=/var/atlassian/bamboo/gcloud
gcloud auth activate-service-account --key-file /var/lib/kubernetes/gcloud-svc-account/gcloud-svc-account
gcloud container clusters get-credentials `cat /var/lib/kubernetes/gcloud-svc-account/gcloud-cluster` --zone `cat /var/lib/kubernetes/gcloud-svc-account/gcloud-cluster-zone `
```
