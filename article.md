### Prerequisites
macOS
Java 11
Maven 
Docker Desktop Community ed. version 2.3.0.3



## Deploying a Vaadin app to Kubernetes

Because there are few things more exciting than deploying an AI app to the cloud, we'll be deploying Alejandre Duarte's Chatbot app to a local Kubernetes cluster today.

It is actually very straightforward. 

For those that did not yet see it yet, the article on how to build the Chatbot app is found [here](https://vaadin.com/blog/building-a-chatbot-in-java). 

#### Building the application 
We start by checking out the Chatbot app: `git clone https://github.com/alejandro-du/vaadin-ai-chat/`. 

You want to ensure that everything works by quickly running the app locally: 
`cd vaadin-ai-chat`
`mvn package`
`java -jar target/vaadin-chat-1.0-SNAPSHOT.jar`

These commands will build and run the application locally. Check it out at http://localhost:8080.

#### Creating an image
Now we know that this works, we'll create a container image so we can easily deploy the application to other environments. Next, we need a Dockerfile to create a container image that we can deploy. I recently noticed that the [Vaadin Starter](https://start.vaadin.com/) now has a convenient option called 'Docker configuration'. We won't use the starter app that is generated this time, but we will extract the `Dockerfile` from the zip file that is conveniently downloaded after configuring the application. 

So copy the `Dockerfile` to the root of the `vaadin-ai-chat` project and open the file to see what is happening. It is interesting to note that the Docker image consists of two different parts, or [multiple stages](https://docs.docker.com/develop/develop-images/multistage-build/). In the first part, the application is built and in the second part, the resulting jar file is run. Note that this means that the original source code is not longer available in the image that is eventually run. 

A Docker image based on the `Dockerfile` is created as follows: `docker build --tag vaadin-ai-chat:1.0 .` Note that is given the name `vaadin-ai-chat` and version number `1.0`.

#### Deploying to Docker
As a next, intermediate step, it is good to check that the image can be correctly deployed to a local Docker environment.  

`docker run --publish 8080:8080 --detach --name vaadin-chat vaadin-ai-chat:1.0`

This should give you a working application, again on port 8080, but this time served from a running Docker image. This is reassuring we're on the right track.

We can actually confirm that the application image does not contain the source code anymore by taking a quick peek in the running container. We do so by looking up the process name: `docker ps` (you will learn the `<container name>`) and by creating a shell into the running container: `docker exec -it <container name> /bin/bash`. 

After we've established the application and thus the image works correctly, we can remove the container again with: `docker rm --force vaadin-chat`.

#### Pushing the image to a repository
Compared to Docker, Kubernetes is a bit more advanced, which means that we cannot just push our Docker image to the Kubernetes cluster. We need to push the image we created to a registry first. A registry is a place from where images are stored and managed.

We tag the image to ensure we give the image a name and version number that uniquely identifies it.    

`docker tag vaadin-ai-chat:1.0 <docker user_name>/vaadin-ai-chat:1.0`

(Please replace `<docker user_name>` by your own Docker username. You can easily [create one](https://www.docker.com/get-started), if needed.)

With this name, we can push the image to a registry. In this case, we're using Docker Hub, Docker's default registry that allows you to have one private repository for free. We'll use a private repository, which means that we'll need to tell Kubernetes the Docker credentials. We'll look at that in a moment.  

`docker push <docker user_name>/vaadin-ai-chat:1.0`

Next, log in to [Docker Hub](https://hub.docker.com/) via a browser and see that the image is available. If necessary, modify the repository's visibility settings to Private.  

Next, we need to create a Secret which Kubernetes can use authenticate and download the image we've just pushed.  

Create a Secret by providing credentials on the command line

We'll create our secret called `regcred` as follows:

`kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>`

(Replace the attributes with your own Docker credentials.)

Once you have created you secret, it is good to inspect the result: 

`kubectl get secret regcred --output=yaml`

#### Preparing Kubernetes
Before we're going to deploy to Kubernetes, it is a good idea to [install the Web UI Dashboard
(https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/), which gives an interesting perspective on what is happening on our local cluster.

Run:
- `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`
- `kubectl proxy`  
- `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

Before we can login to our new Admin dashboard, we need to [create a login and bind this to admin dashboard](https://www.replex.io/blog/how-to-install-access-and-add-heapster-metrics-to-the-kubernetes-dashboard):

- `kubectl create serviceaccount dashboard-admin-sa`
- `kubectl create clusterrolebinding dashboard-admin-sa 
                                                                                       --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa`
We confirm that this worked by retrieving the secret: `kubectl get secrets`. This gives the token we need to login to the UI Dashboard.  

#### Deploying to Kubernetes
Before we can only deploy our application, we need to defined it in a Kubernetes configuration file, which follows the YAML format. 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vaadin-chat
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      vaadin-chat: web
  template:
    metadata:
      labels:
        vaadin-chat: web
    spec:
      containers:
        - name: vaadin-chat
          image: docker.io/markhm/vaadin-ai-chat:1.0
      imagePullSecrets:
      - name: regcred
---
apiVersion: v1
kind: Service
metadata:
  name: vaadin-chat
  namespace: default
spec:
  type: NodePort
  selector:
    vaadin-chat: web
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30001

``` 

Now, we can install the Chatbot application, by asking KubeControl to apply the configuration.  

`kubectl apply -f vaadin-chat.yml`

If you open your browser to https://localhost:30001, you will see the Chatapp in action for the third time, this time deployed to your local Kubernetes cluster.

You can remove (undeploy) the application again with the following command:

`kubectl delete -f vaadin-chat.yml`

All in all, quite straightforward. 

Thanks for your attention...!

----
sources: 
- 
- https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- https://kukulinski.com/10-most-common-reasons-kubernetes-deployments-fail-part-1/
- https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod
