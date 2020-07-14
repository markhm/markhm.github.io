## Deploying a Vaadin app to Kubernetes
Because there are few things more exciting than deploying an AI app to your own cloud, we'll be deploying [Alejandre Duarte](https://vaadin.com/blog/author/alejandro-duarte)'s [Vaadin](https://www.vaadin.com) [Chatbot app](https://vaadin.com/blog/building-a-chatbot-in-java) to a local Kubernetes cluster today. It is actually very straightforward. 

If you did not read it yet, the blog on how to build the Chatbot app is found in [here](https://vaadin.com/blog/building-a-chatbot-in-java). 

In this article, we'll first checkout, build and run the application to confirm it works locally. Then we'll create a Docker image and deploy this image to our local Docker environment. Again, we will confirm that the application still works as expected. We'll then push the image to a private image registry and tell Kubernetes the proper credentials, which will allow Kubernetes to deploy and run our app. 

Let's get started. 

#### 1: Building the application 
We start by cloning the Chatbot app from GitHub: `git clone https://github.com/alejandro-du/vaadin-ai-chat/`. 

You want to ensure that everything works by first building and the app locally: 
`cd vaadin-ai-chat`
`mvn package`
`java -jar target/vaadin-chat-1.0-SNAPSHOT.jar`

The Maven `package` command will build the application, after which it is started as a Java jar application from the `target` directory. Check it out at http://localhost:8080 and ask Alice a few questions. Btw, you will get a nice grasp of what Alice is capable of by taking a look in `src/main/resources/bots/alice/aiml`.

#### 2: Creating an image
Now we know that this works, we'll create a container image so we can easily deploy the application to other environments. We need a Dockerfile to create a container image that contains the application that we can then deploy. This is where a feature called 'Docker configuration' that was recently added to [Vaadin Starter](https://start.vaadin.com/) comes in handy. We won't use the Starter App that is generated this time, but we will extract the `Dockerfile` from the zip file that is conveniently downloaded after configuring the application. 

So open https://start.vaadin.com/, select 'Docker configuration' in the lower left corner and click the blue *Download App* button at the top. Open the zip file and copy the `Dockerfile` to the `vaadin-ai-chat` project folder. 

Take a moment to look at the `Dockerfile` and see which steps it contains. It is interesting to note that the `Dockerfile` consists of two parts, or [multiple stages](https://docs.docker.com/develop/develop-images/multistage-build/). In the first part, the application is built using [Java 11](https://openjdk.java.net/projects/jdk/11/), [Maven](https://maven.apache.org/) and [NodeJS](https://nodejs.org/), and in the second part, the resulting jar file is actually run. You will see that the jar file is copied from the first stage into the second stage to `/usr/app/app.jar`. Note that this means that the original source code is not longer available in the image that is eventually run. 

A Docker image based on the `Dockerfile` is created as follows: `docker build --tag vaadin-ai-chat:1.0 .` Note that is tagged with its name `vaadin-ai-chat` and version number `1.0`.

#### 3: Deploying to Docker
Next, as an intermediate step, it is good to check that the image can be correctly deployed to your local Docker environment by telling Docker to run it:

`docker run --publish 8080:8080 --detach --name vaadin-chat vaadin-ai-chat:1.0`

This should give you a working application, again on port 8080, but this time served from a running Docker container. It is reassuring to know we're on the right track.

We can actually confirm that the application image does not contain the source code anymore by taking a quick peek in the running container. You can do so by looking up the process name: `docker ps` (you will learn the `<container name>`) and by creating a shell into the running container: `docker exec -it <container name> /bin/bash`. Look around at the container's directory structure and see that the Java application that is run is indeed found in `/usr/app/app.jar`.

After we've established the application and thus the image works correctly, we can remove the container again with: `docker rm --force vaadin-chat`.

#### 4: Pushing the image to a repository
Compared to Docker, Kubernetes works a bit differently, which means that we cannot just push our Docker image to the Kubernetes cluster. We need to push the image we created to a registry first. A registry is an online location from where images are stored and managed. We tag the image we created, to ensure we give the image a name and version number that uniquely identifies it.    

`docker tag vaadin-ai-chat:1.0 <docker user_name>/vaadin-ai-chat:1.0`

Replace `<docker user_name>` by your own Docker username. If you don't have one yet, you can easily [create one](https://www.docker.com/get-started).

With this tag, we can now push the image to a registry. We could push to a local registry or one that is run at your organisation, but in this case, we're using Docker Hub, Docker's default registry. Docker Hub allows you to have one private repository for free, which is nice to experiment with. We'll use a private repository, which means that we'll need to tell Kubernetes the Docker credentials. We'll look at that in a moment, we first push the image we created to the registry with:

`docker push <docker user_name>/vaadin-ai-chat:1.0`.

To check all is as expected, log in to [Docker Hub](https://hub.docker.com/) via a browser and see that the image is available. If necessary, modify the repository's visibility settings to `Private`.

Next, we need to create a *Secret*, which Kubernetes can use authenticate and download the image we've just pushed.  

Create such a *Secret* called `regcred` as follows:

`kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>`

(Replace the <attributes> with your own Docker credentials.)

Once you have created the *Secret*, it is good to inspect the result with: `kubectl get secret regcred --output=yaml`.

#### 5: Preparing Kubernetes
Kubernetes has a extensive command line utility called `kubectl` (by some pronounced as [Cube Control](https://opensource.com/article/18/12/kubectl-definitive-pronunciation-guide)), who's list of options can be slightly intimidating at the start. So before we're going to deploy to Kubernetes, we'll [install the Web UI Dashboard]
(https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/), which gives an interesting visual perspective on what is happening on our cluster.

Run the following from the command line:
- `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`
- `kubectl proxy`

Before we can login to our new Admin dashboard, we need to [create a login and bind this to admin dashboard](https://www.replex.io/blog/how-to-install-access-and-add-heapster-metrics-to-the-kubernetes-dashboard). We do so by running:

- `kubectl create serviceaccount dashboard-admin-sa`
- `kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa`

We confirm that this worked by retrieving the secret: `kubectl get secrets`. This gives the token we need to login to the UI Dashboard. Copy and be ready to paste when opening the [Web UI](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/).

#### 6: Deploying to Kubernetes
Before we can only deploy our application, we need to defined it in a Kubernetes configuration file, which follows the YAML format. Save the following in the root of the `vaadin-ai-chat` project with filename `vaadin-chat.yml`: 

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
There are several introductions to Kubernetes' configuration file format available online, e.g. [here](https://www.mirantis.com/blog/introduction-to-yaml-creating-a-kubernetes-deployment/), so we will not explain its full contents in this article. But we will point out the reference to the image (`docker.io/markhm/vaadin-ai-chat:1.0`) and 

Now, we can install the Chatbot application, by asking `kubectl` to apply the configuration, which means to deploy our application: `kubectl apply -f vaadin-chat.yml`.

If you open your browser to https://localhost:30001, you will see the Chatapp in action for the third time, this time deployed to your local Kubernetes cluster.

You can remove (undeploy) the application again with the following command: `kubectl delete -f vaadin-chat.yml`.

All in all, quite straightforward. 

Thanks for your attention...!

----
## Prerequisites
This article is based on the following tool stack:
- macOS Catalina (10.15.5)
- Java 11
- Maven 3.6.3
- kubernetes-cli stable 1.18.5 (bottled) via Homebrew
- Docker Desktop Community ed. version 2.3.0.3

### Sources used: 
- https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- https://kukulinski.com/10-most-common-reasons-kubernetes-deployments-fail-part-1/
- https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod
