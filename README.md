# Generic EKS Jupyterhub Cluster
Readme to add documentation for the generic EKS cluster for a Jupyterhub deployment.  

---
Purpose of this deployment
---
The purpose of this deployment is to create an AWS manage node cluster using the EKS service. The requirements for this to work before deployment are:
* AWS Account 
* AWS CLI setup 

---
What will be deployed:
---
* VPC
* Subnet A, B, C (in each availability zones)
* Internet Gateway
* Route associations 
* HTTP / HTTPS security group
* NFS security group
* Compute nodes (using Amazon's Managed Nodes) 
* Load balancer 
* EFS for shared data
* EFS hub-db-dir 
>Note: To provision both EFS shares the efs-generic.yaml CloudFormation template will want to deployed twice. This is by design, since the hub-db-dir may be a longer lived EFS deployment than shared data. 

---
Order of Deployment:
---
* VPC - CloudFormation 
* EFS - CloudFormation (x2)
* ec2 compute - eksctl 
* nginx ingress - kubectl
* helm initialization - helm 
* test jupyterhub - helm 

---
Deployment:
---
**VPC - CloudFormation**
1. Create new CloudFormation stack using vpc.yaml
2. Continue through installation, creating customization in the template parameters 
3. Confirm deployment went through succesfully

**EFS - CloudFormation** 
1. Create new CloudFormation stack using the efs-generic.yaml template 
2. Add customizations per requirements, use vpc and subnet id's from VPC output
3. Confirm deployment went through succesfully 

**ec2 compute - eksctl**
1. Verify the AWS CLI is installed and setup 
2. Run the following command, substiting the eks.yaml file (changes to eks.yaml may be required, this is a smaller installation)

        eksctl create cluster -f /path/to/files/eks.yaml
    >Note: You may want to update the eks.yaml file to have a different size instance, or different number of instances. Currently it is going to create a single instance of an r5.xlarge.

3.  Run the following command to verify installation has completed (after the progres is finished)

        eksctl get cluster

    The newly created cluster should now appear 

**nginx ingress - kubectl**
1. Run the three following commands to create the ingress controller on the cluster, and in AWS:   

        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/mandatory.yaml

        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/provider/aws/service-l4.yaml

        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/provider/aws/patch-configmap-l4.yaml

  These three commmands will create the ELB, and then setup the required pods and ingress in the infrastructure. 
  > NOTE: Since there is a load balancer in front of the service, the nginx-ingress ingress pod needs to be configured to terminate SSL. Use of kube-lego is deprecated. Configuring this using a helm chart (version 2) will be below. 
  At this point a CNAME record for your deployment should be created that points to the Amazon Elastic Load Balancer name. The format typically is host.domain.tld. This is important for later in the deployment. This assumes you have a domain to use for the deployment. 

**helm initialization - helm**
> Note: This is being pulled from the excellent and hard work from the team over at Jupyter. More documentation here: https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub/setup-helm.html
1. Run the following command to update the security of the cluster
        
        kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous
2. Create service account for Helm in cluster (assuming that helm is installed on local station)

        kubectl --namespace kube-system create serviceaccount tiller
3. Change permission for service account

        kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
4. Initialize the cluster, and the client:

        helm init --service-account tiller --wait
    >Note: If the cluster is already setup, and you just need to reconnect from a different machine you can initialize only the client using: 
        
        helm init --client-only
5. Setup Tiller to only be able to communicate inside of the cluster for better security practices  

        kubectl patch deployment tiller-deploy --namespace=kube-system --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
6. Verify that helm is communicating with the kubernetes service 

        helm version 
    You should receieve a response from the client and server. 

**test jupyterhub - helm**
> Note: There *should* be customizations to the config.yaml and values.yaml files. These will drive the deployment configuration. There are two main portions of the deployment. First will be the customizations to the helm chart. Secondly we will deploy with Helm. 
1. Generate a random string for use in securing the communication between proxy and pods. 

        openssl rand -hex 32
2. Replace the returned results in the config.yaml and values.yaml where "proxy-secret-goes-here" is listed. 

3. Update limits on each pod by changing these fields in config and value files:
        singleuser:
          cpu:
            guarantee: 2
            limit: 8
          memory:
            guarantee: 1G
            limit: 2G
    This will determine the resources each jupyter pod will have, and may impact the overall cluster's use - if there are not enough resources. 

4. Determine if you need a specific docker image for use in the environment. If no specific needs are found, then using the jupyter/datascience-notebook or the jupyter/tensorflow-notebook may be used. The advantage of having a custom image is having packages and environments setup and ready to go. A couple maintained by CalPoly:
   * calpolydatascience/datascience-base
   * calpolydatascience/rstudio-base
   * calpolydatascience/tensorflow-r
   This is updated in this portion:

        singleuser:
          image:
            name: jupyter/datascience-notebook
            tag: 45f07a14b422 # using latest version as of 11-12-2019
   The default will be jupyter/datascience-notebook:45f07a14b422. This must be changed in both the config and values file. 

5. Update storage in the config and values file. Inside of the storage configuration for the singleuser you will see extraVolumeMounts and extraVolumes. You will want to update the portion labeled "amazon-efs-shared-storage-server" with the AWS EFS server that was created for shared data. An example of a server is: fs-12345678.efs.location.amazonaws.com
    This should be updated on both the config and values configuration files. 

6. Update authentication configuration in the config and values file. 
    > Note: Pulling again from the Jupyter team. https://zero-to-jupyterhub.readthedocs.io/en/latest/administrator/authentication.html

            auth:
              admin:
                users:
                - GitHubAdminAccount
              github:
                callbackUrl: https://host.domain.tld/hub/oauth_callback
                clientId: XXXXXXX
                clientSecret: XXXXXXX
              type: github
    There should always be at least on admin account, with more often an admin faculty and staff account for whomever deployed service. 

7. Update host.domain.tld in both config and values files. Under the ingress controller, there will be listed "host.domain.tld". This should be replaced with the domain name that will be used. A couple examples are:
    * class-quarter.domain.tld
    * research-group.domain.tld
    * workshop.domain.tld

8. Customizations to values.yaml only:
    >Note: These will only be applied to the values.yaml and will not work until a helm upgrade is ran. 
    To update:
      * claimName: nfs-host-pvc
      * secretName: host.domain.tld
    
    What these updates do:
    claimName - migrates jupyterhub database directory to the efs share created using CloudFormation 
    secretName - used to point the hub at the imported SSL certificate

9. Updates to pvc.yaml 
    The pvc.yaml will lastly need to be updated with updated values. You will update it corresponding to this:
    |amazon-efs-shared-storage-server| AWS EFS server for shared storage|
    |---|---|
    |amazon-efs-hub-storage-server| AWS EFS server for the Jupyterhub database directory|

    What this will create is two PV's and two PVC's using shared storage, instead of locally attached storage 
    >Note: This is in response to an issue with Availability Zones in AWS and locally attached storage. For more information a few articles are here: https://github.com/jupyterhub/zero-to-jupyterhub-k8s/issues/870, https://discourse.jupyter.org/t/jupyterhub-hub-db-dir-pv-question/2157

10. Helm deploy
    Back at the CLI where the cluster can be accessed - run the following commands: 

        helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
        helm repo update
    This will add the repository to your available helm charts to install, and then run an update to verify to pull into the cache. 

11. Jupyterhub Install
    We can finally now run the first installation command with Helm:

        helm install -n name jupyterhub/jupyterhub --namespace namespace --values /path/to/files/config.yaml
    This will deploy a Jupyterhub installation, using the default configuration with our updates on top using the config.yaml file. Typically the name of the deployment is the same as the namespace. 

11. Kubernetes secret
    Now a Kubernetes secret will be created to make sure the configuration is SSL encrypyted. 

        kubectl -n namespace create secret tls host.domain.tld --key=/path/to/files/domain.key --cert=/path/to/files/domain.crt
    The host.domain.tld must match the secretName that is listed in the values.yaml configuration file for SSL to work. 

12. PVC deploy
    Now we will create the PV and PVC's using the file we modified earlier. 

      kubectl -n namespace apply -f /path/to/files/pvc.yaml 

13. PVC Shares
    Now that the PVC has been created we need to update the actual EFS share to have the correct top level folders. This is done easiest by connecting to the kubernetes node using ssh. 
    1. Connect via SSH to server. 
    2. Make sure either EFS utils or NFS utils are on the system. This shouldn't be needed since it will be an Amazon Linux server. 
          
            sudo yum install -y amazon-efs-utils
            or
            sudo yum install -y nfs-utils
    3. Create the mount directory:
    
            mkdir /tmp/hub /tmp/shared
            cd /tmp

    4. Mount shares:

            sudo mount -t efs amazon-efs-hub-storage-server:/ /tmp/hub
            sudo mount -t efs amazon-efs-hub-shared-storage:/ /tmp/shared 
          > Note that this we will substitute the command with the fields required. Instead of using the whole server name, you will only need the host entry that is available. An example is that you could have a server with the domain name of fs-12345678.efs.location.amazonaws.com, in this example you will only use the fs-12345678 as the substitute. Example:
          > hub efs storage is fs-12345678.efs.us-west-2.amazonaws.com, shared storage is fs-87654321.efs.us-west-2.amazonaws.com - the commands would look like this:
          > sudo mount -t efs fs-12345678:/ /tmp/hub
          > sudo mount -t efs fs-87654321:/ /tmp/shared 

    5. Create folders:
        Now that the folders are mounted, you will need to create subdirectories to be mapped in the containers. These are using the default server pathings in the config and values.yaml

            sudo mkdir /tmp/hub/host-hub /tmp/shared/shared 
            sudo chmod -R /tmp/hub && sudo chmod -R /tmp/shared 
        This should set the permissions on the folders as well to allow files and folders to be created inside of there. 

13. Jupyterhub Upgrade 
      Now that all of the other prerequisites are finished, we can tie the whole thing up with an upgrade to the application, using the values.yaml file. 

            helm upgrade --install name jupyterhub/jupyterhub --namespace namespace --values /path/to/files/values.yaml

14. Test application:
    You should now be able to go to the host.domain.tld that you setup in your DNS registrar. This will forward you to the ELB, then cluster and the solution. 

---
Extended documentation: Using manual SSL termination on Nginx Ingress controller:
---
1. Update Helm Chart
First the helm chart will need to be updated to have the updated ingress config:

        jupyterhub:
          ingress:
            enabled: true
            annotations:
              kubernetes.io/tls-acme: "true"
              kubernetes.io/ingress.class: nginx
            hosts:
              - YOUR-JUPYTERHUB-HOST-DOMAIN
            tls:
              - secretName: YOUR-JUPYTERHUB-HOST-DOMAIN
                hosts:
                  - YOUR-JUPYTERHUB-HOST-DOMAIN

> The "YOUR-JUPYTERHUB-HOST-DOMAIN" will need to be updated in this example on a per case by case basis. 
2. Create kubernetes secret 
The kubernetes secret is the ssl certificate being imported to the cluster. It's important to note, that these should be cleaned up on removal of a deployment - and that they need to be deployed in the namespace of the deployment, not in the kube-system or default namespace. 

        kubectl -n namespace create secret tls YOUR-JUPYTERHUB-HOST-DOMAIN --key=domain.key --cert=domain.crt
> Note that the secret name is being set to the fqdn which should match the secretName in the helm chart. This can be something besides the domain name, but this suffices since there is only one secret to handle in the deployment. 

3. Update deployment 
Once the chart has been updated, the deployment will need to be updated:

        helm upgrade --install name jupyterhub/jupyterhub --namespace name --version=version --values=values.yaml

> This should be a very quick update and no user pods will be restarted 

Once this process is completed it may take up to a minute for the updated ssl certificate to show in the hub. 

---
Extended documentation: Regarding kubectl nginx-ingress controller deployment
---
Note, that this is what created the load balancer in the infrastructure, and applied the instances to the load balancer. What needed to be ran was: 

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/mandatory.yaml

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/provider/aws/service-l4.yaml

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/provider/aws/patch-configmap-l4.yaml
These three commmands will create the ELB, and then setup the required pods and ingress in the infrastructure. 
> NOTE: Since there is a load balancer in front of the service, the nginx-ingress ingress pod needs to be configured to terminate SSL. Use of kube-lego is deprecated. 


