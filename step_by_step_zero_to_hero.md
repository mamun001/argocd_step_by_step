
#
#     STEP BY STEP GUIDE FOR ARGO CD SETUP & FUNCTIONING PIPELINE
          (Continuous Deployment for Kubernetes)

##    Use this step-up-step guide if you want to learn how use ARGO CD to set up Continuous Deployment for Kubernetes Infrastructure and Applications
#


##    By: Mamun Rashid
###   Connect with me on Linkedin: https://www.linkedin.com/in/mamunrashid/
#

#

## Pre-requisites
#

#


### 1. MacOS or Ubuntu Linux
        I will add Windows equivalent later.
### 2. Access to a GCP project
     Take a note of the project name:
       e.g. foobar-project

### 3. kubernetes Engine API is enabled on that GCP project
        e.g. https://console.cloud.google.com/apis/api/container.googleapis.com/overview

### 4. Your GCP account has permission to admin k8s clusters

### 5. gcloud (GCP cli installed) on your local Mac or Ubuntu Machine

### 6. kubectl installed on your local Mac or Ubuntu Machine

#

#


## COST
#
### As of September 2022, a standard Kubernetes Cluster with 2 Small Instances will cost $0.06/Hour (6 Cents/Hour)
#

#


##  Steps:
#

### 7. Create vpc, subnet:
         You will ony need to create a vpc and subnet if you don't already have a default VPC and at least 1 subnet in it

### 8. Create a Kubernetes Cluster
         e.g. gcloud container clusters create foobar-cluster --region us-central1  --project foobar-project
         Make a note of the name of the cluster and the region it is in. You will need that shortly.
         e.g. 
          foobar-cluster
          us-central1

### 9. kubectl config current-context
         You will see that you are NOT connected to your newest cluster you just created

### 10. Make the kuberneetes cluster your current context:
         gcloud container clusters get-credentials name_of_kubernetes_cluster  --region region_where_kubernetes_cluster_is_in --project foobar-project

### 11. Confirm that now you ARE connected to your kubernetes cluster:
    kubectl config current-context


### 12. Create argocd namespace
          kubectl create namespace argocd
          Confirm by:
          kubectl get ns


### 13. Install argocd on your cluster:
          kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
          Confirm by:
          kubectl get svc -n argocd

### 14. On a mac, brew updates take too long. Turn off auto-update for brew.
          export HOMEBREW_NO_AUTO_UPDATE=1
          Confirm by:
          env | grep HOMEBREW

### 15. (On Mac) Install argocd CLI:
           brew install argocd 
           Confirm by:
           argocd version

### 16. On Ubuntu Linux:
           wget https://github.com/argoproj/argo-cd/releases/download/v2.2.5/argocd-linux-amd64 -O argocd
           chmod 755 argocd
           sudo mv argocd /usr/local/bin/

### 17. Change the argocd-server service type to LoadBalancer: (This will allow us to get to UI from Mac using port forwarding)
          kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
          Confirm by:
          kubectl describe svc argocd-server -n argocd 

### 18. Set up port forwarding to argocd Server UI:
          kubectl port-forward svc/argocd-server -n argocd 8080:443 &
          Confirm by:
          Use chrome and go to localhost:8080

### 19. Get the initian admin password for argocd UI:
          kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
          Note down the string you get back. That is your password.


### 20. Get the IP of the argocd_server service:
          kubectl get svc -n argocd | grep argocd-server | grep -v metrics
          Note down the external IP address that you see  (NOT the one that starts with 10.)

### 21. Login to argocd server via argocd CLI:
          argocd login IP_address_that_you_noted_down_in_the_previous_step
            Username will be admin
            Password will be the one you noted down earlier
          Confrim by:
            Getting the following message: 'admin:login' logged in successfully


### 22. (Optional): Change the admin password to something that easier to remember:
           argocd account update-password
           Note down the new password if you do choose to change the admin password.


### 23. Find a repo that has Kubernetes Yaml files that you can deploy:
          You have a few choices here:
            You can use my public one (has a simple nginx deployment)
              https://github.com/mamun001/argocd_playground
          Or you can use the one that argocd official documenation has:
              https://github.com/argoproj/argocd-example-apps.git
          Or you can write your own in your own git repo.

     #### In this example we will use my repo above (https://github.com/mamun001/argocd_playground)


### 24. Add an app into argocd configuration using my simple repo. 
          argocd app create foobar-nginx --repo https://github.com/mamun001/argocd_playground.git --path nginx_deployment --dest-server https://kubernetes.default.svc --dest-namespace default 
          Confirm when you see:
            application 'foobar-nginx' created
          Explanations:
            app create: create an applictaion that argocd will monitor and pull in a regular interval. Whenever it sees a new commit in the main branch, it will automatically pull down the changes and deploy (provided sync policy us set to automate). This is the WHOLE IDAE of argocd.
            foobar-nginx: This is just name of the app we chose. It can be almost anything (no underscore allowed, though)
            --repo: Git repo where the application code for all Kubernetes objects are kept.
            --path: Within that repo, you can choose a particular folder to point to (to pull from). In this case, my yaml files are kept in nginx_deployment folder.
            --dest-server: which cluster this will deploy to
            --dest-namespace: which namesspace this will deploy to.

### 25. Confirm that your argocd app has been created and see its settings:
          argocd app get foobar-nginx

    You should get:
      Name:               foobar-nginx
      Project:            default
      Server:             https://kubernetes.default.svc
      Namespace:          default
      URL:                https://127.0.0.1:50418/applications/foobar-nginx
      Repo:               https://github.com/mamun001/argocd_playground.git
      Target:             
      Path:               nginx_deployment
      SyncWindow:         Sync Allowed
      Sync Policy:        <none>
      Sync Status:        OutOfSync from  (8aacf95)
      Health Status:      Missing
      
      GROUP  KIND        NAMESPACE  NAME   STATUS     HEALTH   HOOK  MESSAGE
      apps   Deployment  default    nginx  OutOfSync  Missing       


### 26. You see that app is out of sync. Sync the app.
          argocd app sync foobar-nginx
          Confirm by:
            Seeing:  Message:            successfully synced (all tasks run) 

### 27. Confirm that deployment is indeed created in your cluster in the default namespace.
          kubectl get deploy

      You should see:
      NAME    READY   UP-TO-DATE   AVAILABLE   AGE
      nginx   1/1     1            1           55s


### 28. Delete the argocd app:
          argocd app delete foobar-nginx
        Confirm by:
          argocd app list
          You should no longer see the app that you deleted

### 29. Now create the same app again. This time we will tell argocd to automatically sync from repo whenever there is a commit. This is main idead of argocd.
          argocd app create foobar-nginx --repo https://github.com/mamun001/argocd_playground.git --path nginx_deployment --dest-server https://kubernetes.default.svc --dest-namespace default --sync-policy automated
         Confirm by:
           argocd app list
      
         You should see:
           NAME          CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                               PATH              TARGET
           foobar-nginx  https://kubernetes.default.svc  default    default  Synced  Healthy  Auto        <none>      https://github.com/mamun001/argocd_playground.git  nginx_deployment  


### 30. Clean up: Delete the app:
          argocd app delete foobar-nginx

### 31. Destroy the Kubernetes cluster so that you stop incurring cost of running a small cluster.
          gcloud container clusters delete kubernetes_cluster_name --zone zone_where_the_cluster_is --project project_name_where_the_cluster_is


