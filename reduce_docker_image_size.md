# How to Reduce Docker Image Size: 6 Optimization Methods

If you want to reduce docker image size, you need to use the standard best practices in building a Docker Image.

This blog talks about different optimizations techniques that you can quickly implement to make the smallest and minimal docker image. We will also look at some of the best tools for Docker Image Optimization.

Docker as a container engine makes it easy to take a piece of code & run it inside a container. It enables engineers to collect all the code dependencies and files into a single location which can be run anywhere, quite quickly & easily.

The whole concept of “run anywhere” images starts from a simple configuration file called Dockerfile. First, we add all the build instructions, such as the code dependencies, commands, and base image details, in Dockerfile.

## Need for Docker Image Optimization
Even though the Docker build process is easy, many organizations make the mistake of building bloated Docker images without optimizing the container images.

In typical software development, each service will have multiple versions/releases, and each version requires more dependencies, commands, and configs. This introduces a challenge in Docker image build, as now – the same code requires more time & resources to be built before it can be shipped as a container.

I have seen cases where the initial application image started with 350MB, and over time it grew to more than 1.5 GB.

Also, by installing unwanted libraries, we increase the chance of a potential security risk by increasing the attack surface.

Therefore, DevOps engineers must optimize the docker images to ensure that the docker image is not getting bloated after application builds or future releases. Not just for production environments, at every stage in the CI/CD process, you should optimize your docker images.

Also, with container orchestration tools like Kubernetes, it is best to have small-sized images to reduce the image transfer and deploy time.

## How to Reduce Docker Image Size?

The following are the methods by which we can achieve docker image optimization.

1. Using distroless/minimal base images
2. Multistage builds
3. Minimizing the number of layers
4. Understanding caching
5. Using Dockerignore
6. Keeping application data elsewhere

# Method 1: Use Minimal Base Images
Your first focus should be on choosing the right base image with a minimal OS footprint.

One such example is alpine base images. Alpine images can be as small as **5.59MB**. It’s not just small; it’s very secure as well.

```
alpine       latest    c059bfaa849c     5.59MB
```
Nginx alpine base image is only 22MB.

By default, it comes with the sh shell that helps debug the container by attaching it.

You can further reduce the base image size using distroless images. It is a stripped-down version of the operating system. Distroless base images are available for java, nodejs, python, Rust, etc.

Distroless images are so minimal that they don’t even have a shell in them. So, you might ask, then how do we debug applications? They have the debug version of the same image that comes with the busybox for debugging.

Also, most of the distributions now have their minimal base images.

## Method 2: Use Docker Multistage Builds

The multistage build pattern is evolved from the concept of builder pattern where we use different Dockerfiles for building and packaging the application code. Even though this pattern helps reduce the image size, it puts little overhead when it comes to building pipelines.

In multistage build, we get similar advantages as the builder pattern. We use intermediate images (build stages) to compile code, install dependencies, and package files in this approach. The idea behind this is to eliminate unwanted layers in the image.

After that, only the necessary app files required to run the application are copied over to another image with only the required libraries, i.e., lighter to run the application.

Let’s see this in action, with the help of a practical example where we create a simple Nodejs application and optimize its Dockerfile.

First, let’s create the code. We will have the following folder structure.

Save the following as ```index.js```.

```
const dotenv=require('dotenv'); 
dotenv.config({ path: './env' });

dotenv.config();

const express=require("express");
const app=express();

app.get('/',(req,res)=>{
  res.send(`Learning to Optimize Docker Images with DevOpsCube!`);
});


app.listen(process.env.PORT,(err)=>{
    if(err){
        console.log(`Error: ${err.message}`);
    }else{
        console.log(`Listening on port ${process.env.PORT}`);
    }
  }
)
```

Save the following as ```package.json```.

{
  "name": "nodejs",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "dotenv": "^10.0.0",
    "express": "^4.17.2"
  }
}

Save the following port variable in a file named ```env```.

```
PORT=8080

```
A simple Dockerfile for this application would like this – Save it as Dockerfile1.

```
FROM node:16
COPY . .
RUN npm installEXPOSE 3000
CMD [ "node", "index.js" ]
```

Let’s see the storage space that it requires by building it.

```
docker build -t devopscube/node-app:1.0 --no-cache -f Dockerfile1 .

```
After the build is complete. Let’s check its size using –
```
docker image ls
devopscube/node-app   1.0       b15397d01cca   22 seconds ago   910MB
```
So the size is **910MBs**.

Now, let’s use this method to create a ```multistage build```.

We will use node:16 as the base image, i.e., the image for all the dependencies & modules installation, after that, we will move the contents into a minimal and lighter ‘alpine‘ based image. The ‘alpine‘ image has the bare minimum utilities & hence is very light.

Here is a pictorial representation of a Docker multistage build.

![image](https://user-images.githubusercontent.com/40743779/184480211-c8c8dedf-ac36-408a-a62b-44fa56042ca1.png)

Also, in a single Dockerfile, you can have multiple stages with different base images. For example, you can have different stages for build, test, static analysis, and package with different base images.

Let’s see what the new Dockerfile might look like. We are just copying over the necessary files from the base image to the main image.

Save the following as Dockerfile2

```
FROM node:16 as build

WORKDIR /app
COPY package.json index.js env ./
RUN npm install

FROM node:alpine as main

COPY --from=build /app /
EXPOSE 8080
CMD ["index.js"]
```
Let’s see the storage space that it requires by building it.
```
docker build -t devopscube/node-app:2.0 --no-cache -f Dockerfile2 .
```
After the build is complete. Let’s check its size using
```
docker image ls
```
This is what we get.
```
devopscube/node-app   2.0       fa6ae75da252   32 seconds ago   171MB
```
So the new reduced image size is **171MBs** as compared to the image with all dependencies.

That’s an optimization of over 80%!

However, if we would have used the same base image we used in the build stage, we wouldn’t see much difference.

You can further reduce the image size using distroless images. Here is the same Dockerfile with a multistage build step that uses the google nodeJS distroless image instead of alpine.

```
FROM node:16 as build

WORKDIR /app
COPY package.json index.js env ./
RUN npm install

FROM gcr.io/distroless/nodejs

COPY --from=build /app /
EXPOSE 3000
CMD ["index.js"]
```
If you build the above Dockerfile, your image will be 118MB,
```
devopscube/distroless-node   1.0       302990bc5e76     118MB
```
## Method 3: Minimize the Number of Layers

Create docker file like below [use pipe and && and create as only one layer]
```
FROM ubuntu:latest
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update -y && apt-get upgrade -y && apt-get install --no-install-recommends vim net-tools dnsutils -y
```
## Method 4: Understanding Caching

Often, the same image has to be rebuilt again & again with slight modifications in code.

Docker helps in such cases by storing the cache of each layer of a build, hoping that it might be useful in the future.

Due to this concept, it’s recommended to add the lines which are used for installing dependencies & packages earlier inside the Dockerfile – before the COPY commands.

```
FROM ubuntu:latest
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update -y
RUN apt-get upgrade -y
RUN apt-get install vim -y
RUN apt-get install net-tools -y
RUN apt-get install dnsutils -y
COPY . .
```

## Method 5: Use Dockerignore
As a rule, only the necessary files need to be copied over the docker image.

Docker can ignore the files present in the working directory if configured in the ```.dockerignore``` file.

This feature should be kept in mind while optimizing the docker image.

## Method 6: Keep Application Data Elsewhere

Storing application data in the image will unnecessarily increase the size of the images.

It’s highly recommended to use the volume feature of the container runtimes to keep the image separate from the data.

READ: https://devopscube.com/reduce-docker-image-size/
      https://blog.codacy.com/five-ways-to-slim-your-docker-images/

