
# NETCoreDockerIBMCloud

The purpose of this project (or it was supposed to be a guide?) is to containerize an application of your own and to publish it into IBM Cloud using your created image being handled on Kubernetes.
We are treating IBM Cloud specifics here, but the overall concept can be used for any cloud-friendly environment you choose to play with.

This project is pretty much the standard .NET Web API template with the addiction of a Dockerfile to be used when deploying your app to IBM Cloud.

## Warm up

Before we start, you must have your IBM Cloud account already set up.
If you don't have it, go to https://cloud.ibm.com/ and create yours!

**If you are an IBM employee, you can use your federated account to do so. During this step-by-step, this is how I am going to authenticate myself, but you can proceed with a regular account with no problems.**

Besides that, you will need:
1. .NET Core SDK - https://dotnet.microsoft.com/download/dotnet-core
	>This specific project uses .NET Core 3.1 - Check yours with `dotnet --version`
2. Visual Studio Code - [https://code.visualstudio.com/Download](https://code.visualstudio.com/Download)
3. Docker Desktop - [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
4. Docker Extension for Visual Studio Code - `CTRL+SHIFT+P` > `Install Extensions`
5. IBM Cloud CLI - Covered later on this guide.

## Create your project

On Visual Studio Code terminal, move to the directory where you want to create your project.

As you may have noticed already, I created a Web API project with a self-explanatory (aka ridiculous-long) name, so create yours by typing: `dotnet new webapi -n NetCoreDockerIBMCloud` 

## Execute it for the first time!

Are you already comfortable with this template? If you are not, explore a little bit the generated code before you proceed.

Move to the working directory of your project and execute: `dotnet run`
Now you should see a bunch of information stating where your application is running. By default, you can access it on: [https://localhost:5001/weatherforecast](https://localhost:5001/weatherforecast)
> **Important:** Double check the ports your application is running on.

## Let's have some fun - Containerizing time

On this step, we need a tricky file that will teach Docker how he is supposed to create our image: Dockerfile

Since you already have Docker Desktop and Docker Extension for Visual Studio Code installed, press: `CTRL+SHIFT+P` and type "Add Docker Files to Workspace".  You will have to choose from a few options now:
1. Application Platform: .NET: ASP.NET Core
2. OS: Personally, I chose Linux.
3. Optional files: Nah, not right now.
4. Ports: Feel free to use the default ports: 80 and 443

You will notice we now have our Dockerfile created. Okay... what does that mean? By definition: A `Dockerfile` is a text document that contains all the commands a user could call on the command line to assemble an `image`.

Said that, let's assemble ours!

On your terminal, execute: `docker build -t mynetcoreapi .`
> Notice the . at the end of the command. This little guy tells the Dockerfile is on our working directory.

Check your recently created image using: `docker images`. You should see `mynetcoreapi` somewhere on the list.

## Run your container

Awesome! The image is created. Now what?
On your terminal, execute: `docker run -p 80:80 --name myapp mynetcoreapi`. This will tell Docker to start running our image.

Open your browser and check what we have on: [http://localhost/weatherforecast](http://localhost/weatherforecast). This is our app running on a container! That. Easy. Execute `docker ps` and will you see your container on the list.

## Preparing the cloud environment

Finally we will start handling stuff on the cloud! 

To begin with, we need to create a cluster so we can start deploying our API to the cloud. 

I am not going to cover everything in details here since it's something that can change pretty fast, but in general terms, from your IBM Cloud interface you should be able to create a new free cluster to work with. All you should have to provide is a cluster name, the type of plan (free!) and the cluster type: for this sample, we are working with Kubernetes.
> **Since it's a free cluster, it will expire in 30 days after its creation.**

From now on, the interface should give you the most important detail you have to focus on: install the `IBM Cloud CLI`.
> Notice: the next steps may change, so I suggest double checking what the IBM Cloud interface tells you to do.

Installing the `IBM Cloud CLI`:
Run this command with your `PowerShell` to download and install a few CLI tools and plugins.
`Set-ExecutionPolicy Unrestricted; iex(New-Object Net.WebClient).DownloadString('http://ibm.biz/idt-win-installer')` 
> Take a rest, this will take a while.

Log into your IBM Cloud: `ibmcloud login -a cloud.ibm.com -r us-south -g Default -sso`
> If you are using your federated account like me, type "Y" when prompted and use the code on your browser to authenticate on your terminal. If you are not, just remove the `-sso` from the command and proceed as usual.

Download the kubeconfig files for your cluster: `ibmcloud ks cluster config --cluster bpskn55d0g6kf9q70kl0`
> The hash at the end of the command will be unique for your cluster. Use the one provided on the IBM Cloud interface.

Set the KUBECONFIG environment variable. Copy the output from the previous command and paste it in your terminal. 

Verify that you can connect to your cluster with: `kubectl version --short`

The output of the previous command should look like: `Client Version: v1.15.5 Server Version: v1.16.8+IKS`
> You may have to enable Kubernetes on your Docker Desktop to use `kubectl`.

## Creating the registry on your cluster

Pretty much the same as we did on the previous step. Let's follow the interface guidance.

Install the Container Registry plug-in: `ibmcloud plugin install container-registry -r 'IBM Cloud'`

Choose a name and create your namespace: `ibmcloud cr namespace-add phillipnamespace`

## Pushing your image

It's time to send the image we previously created to the cloud!
On your terminal, execute `ibmcloud cr login`. You will see the endpoints you are connecting to. This is where we are going to push our image on the next steps.

Let's tag our image to prepare it to be sent to our private repository on IBM Cloud: `docker tag mynetcoreapi us.icr.io/phillipnamespace/mynetcoreapi:v1 `
> Notice that a portion of the tag is the address of the private repository: `us.icr.io` and our namespace as well.

Once again, you can see your image by executing: `docker images`

Finally, let's push our image: `docker push us.icr.io/phillipnamespace/mynetcoreapi:v1`

Check if your image was successfully sent using: `ibmcloud cr image-list`
> This command performs the same action as `docker images`, but now we are looking at your images on IBM Cloud.

## Creating a deployment

Now we are going to create a deployment using our recently pushed image.

Execute: `kubectl run mynetcoreapi-dep --image=us.icr.io/phillipnamespace/mynetcoreapi:v1`
> If everything went well with your deployment, you can now see a running pod with `kubectl get pods`

## Exposing the pod through a service

Awesome, we are finally there. We have our pod/s running on the cloud, we just need to expose them.
> Due to a limitation of the free cluster, we may only have a single pod running.

Okay, create a service using: `kubectl expose deployment/mynetcoreapi-dep --type=NodePort --port=80 --name=mynetcoreapi-service --target-port=80`

Did that, we have everything set up! We just need to check how we can actually access our API now. For that, we will need the `public IP` of our cluster and the `port` our service is located.

Let's get the public ip of the cluster: `ibmcloud cs workers mycluster`
> Take note of the `Public IP`.

Execute: `kubectl describe service mynetcoreapi-service`
> Now, take note of the `NodePort`.

## Try your API on the cloud!

Finally, we just need to combine the information we got on the previous two commands and we should be able to call our API on the IBM Cloud. You will have something like this: [http://184.173.5.151:31554/weatherforecast](http://184.173.5.151:31554/weatherforecast)
> Most likely, at the moment you may test this API, the cluster will be already down due to its 30 day expiration period, but trust me, it worked once, and `it was not on my machine!` :)

# Any last words?

With this step-by-step, we were able to create a new project using the default `.NET Core Web API` template, build a `Docker image` with it and send it to `IBM Cloud` to be managed using `Kubernetes`. For the next steps, I suggest you to explore the remaining items available `for free` at `IBM Cloud` and even play around with your already created `Kubernetes` instance.