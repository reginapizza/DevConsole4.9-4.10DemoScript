**SCRIPT:**

***Part 1: KAM Bootstrapping***

1. Have cluster running, log in through terminal
2. Go to OperatorHub and install GitOps operator
3. Also in OperatorHub install the pipelines operator
4. Before running kam bootstrap, show that https://github.com/reginapizza/gitops is 404 not found
5. Show & run kam bootstrap command: 
``` 
kam bootstrap /
--service-repo-url https://github.com/reginapizza/taxi.git / 
--gitops-repo-url https://github.com/reginapizza/gitops.git /
--image-repo quay.io/reginapizza/taxi /
--dockercfgjson ~/Downloads/rescott-robot-auth.json / 
--git-host-access-token $GIT-TOKEN /  
--output  ~/output / 
--push-to-git=true /  
--overwrite
```
Kam is a CLI tool, it stands for Kubernetes Application Manager and we’re going to use it to interact with our gitops repos. I will be having everything in this demo through github but you can also use gitlab. I also have a public instance of these repos but it will also work for enterprise github and gitlab. 
In this command, first we have an example of a service repo, Taxi, which contains the source code, deployment manifests and CI/CD pipelines for the taxi application.
Our `gitops-repo-url` is the URL where our gitops manifests will be created. If you already have this repo in existence then this command will fail.
The `dockercfgjson` is something I set up already in Quay.io. I have a repo in Quay called taxi and I gave a robot account write access to it, so when I run my kam bootstrap command the CI pipeline will build an image of taxi and push it to this repo in Quay.  I downloaded my dockerconfig for it and I just have it in my Downloads folder so it’s getting that information from there. 
Our `--git-host-access-token` is the token that I have in my account that should have write access to my git account
Output is just the location and name of the file that will be created from the output of this kam bootstrap command, it’ll contain all my git manifests and it should be the same as what will be created in my gitops-repo-url repo in github that we just talked about above.
I’m also specifying that `--push-to-git` is true so that it’ll push that gitops-repo-url to git.
And lastly the `--overwrite` argument is to specify that if anything is already located in the ~/output folder location, that it’s okay to overwrite it
6. Go to Developer view in OpenShift and click on Environments to show that there are none right now.
7. Show that https://github.com/reginapizza/gitops now exists and has files
8. Run:
```
cd ~/output
oc apply -k config/argocd/
```
	This will install argocd and create `argo-app`, `cicd-app`, `dev-app-taxi`, `dev-env`, and `stage-env`
9. Switch to developer view and go to `openshift-gitops` namespace, click on Secrets tab, and then find openshift-gitops-cluster. Click on it and then copy the password at the bottom of the page. Go to launcher and go to “Cluster Argo CD”, and log in with username admin and password from clipboard 
OR
Click “Login with OpenShift” and it should get your OpenShift credentials and bring you to the page where you will have to click “Allow selected permissions” and then you should see the Argo CD UI.
10. Add the webhook for the gitops repo
11. Re-sync applications

App should now show in the Environments List page!

***Part 2: Adding more environments***

Now that we have one environment, let’s add a few more before we explore them in the UI here so we can get a more complete look of things.

First we’re going to be adding a new environment, aptly named new-env:
1. Run:
```
kam environment add \
  --env-name new-env \
  --pipelines-folder ~/output
```

2. Now we’re going to be adding a new service in new-env:
Run:	

```
kam service add \
  --env-name new-env \
  --app-name app-bus \
  --service-name bus \
  --git-repo-url http://github.com/reginapizza/bus.git \
  --pipelines-folder ~/output
```
3. Open vscode and go to environments → new-env → apps/app-bus → services/bus → base → config

You will see that all these folders have been generated for our new environment and service. This kustomization.yaml file is generated to reference the new service.

Now we’re going to be adding a few new files to the config folder here. The first one we’re going to add is deployment yaml to specify how the service should be deployed. 

4. Create a  new folder called 100-deployment.yaml containing:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  # creationTimestamp: null
  name: bus
  namespace: new-env
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: bus
      app.kubernetes.io/part-of: app-bus
  strategy: {}
  template:
    metadata:
      # creationTimestamp: null
      labels:
        app.kubernetes.io/name: bus
        app.kubernetes.io/part-of: app-bus
    spec:
      containers:
      - image: nginxinc/nginx-unprivileged:latest
        imagePullPolicy: Always
        name: bus
        ports:
        - containerPort: 8080
        resources: {}
      serviceAccountName: default
status: {}
```
5. Now we're going to add and make the yaml file for the bus service. 
```
apiVersion: v1
kind: Service
metadata:
  # creationTimestamp: null
  labels:
    app.kubernetes.io/name: bus
    app.kubernetes.io/part-of: app-bus
  name: bus
  namespace: dev
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: bus
    app.kubernetes.io/part-of: app-bus
status:
  loadBalancer: {}
```

6. Finally, we will make a kustomization.yaml file to make references to both resources. 
```
resources:
- 100-deployment.yaml
- 200-service.yaml
```

7. Now, `git add .` and `git commit -m "Add new service"` and then `git push origin main` to push all your change to git. When Argo CD sees that changes have been detected in the git repo, it will automatically apply those changes to the cluster and deploy your new service. 


***Part 3: GitOps ListPage + Details Page***
1. Switch to new cluster with already deployed apps and environments at https://github.com/reginapizza/gitops
2. Show the GitOps List Page
  a. Application Name - clicking on the name will take you to the details page for that application
  b. Git Repository - clicking on this link will take you to that repo
  c. Environment Status- here you'll see the number of environments under each application for a certain Sync status: either Synced, OutOfSync, or Unknown. 
  d. Last Deployment - is the time of the most recent deployment for any application, regardless of status. 
  
3. Click on an app name and go to the details page for that app

[ to be continued ]
  
