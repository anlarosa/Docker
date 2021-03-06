- [index](https://github.com/KiraDiShira/Docker/blob/master/README.md#docker)

# Containerizing an App

## The Big Picture

We've got our code. So running it means running it in a container. But how do we do that?

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa1.PNG" />

Our app code. To get it into an image, we create a **Dockerfile** (this is a list of instructions on how to build an image with our code inside). We use the docker `image build` command to actually build the image. And out pops an image!

## Containerizing an App

It's normal to put your Dockerfile in the root folder of your app.

Now, all a Dockerfile is, is a list of instructions on how to build an image with an app inside. This is going to document the app. So, describe it to the rest of the team, and to new starts, and also to the ops guys.

The example I'm using here is a Linux app, right? So we need some form of a Linux base image. I'm a fan of Alpine these days, so I'm going to start it from that.

It's convention that **instructions** in a Dockerfile are capitalized. In fact, all this file really is is a list of key value pairs. Almost. You always start a Dockerfile with a `FROM` instruction. This is going to be the base image that we add our app on top of.

```
FROM alpine
```

Next up, I'm going to list myself as the maintainer to show you how to use labels. 

```
LABEL mainteiner="alarosa@hotmail.com"
```

The app that we're containerizing happens to be a node app, so we need to install `node` and `npm`, the node package manager. 

```
RUN apk add --update nodejs nodejs-npm
```

The from instruction up here pulls the alpine image and uses that as the base layer for the image that we're building. The label that I'm using here, all that does is adds a bit of metadata to the image config. But then `RUN` here creates a new layer. We're adding some software to the image, so we get a new layer.

Next, we want to copy in our source code, so everything from the same directory as our Dockerfile here, and we'll copy that into `/src` image. 

```
COPY . /src
```

We're adding more content, so we get another layer. 

Well, we're set the working directory to `/src`.

```
WORKDIR /src
```

We won't need a layer for that, it's just metadata.

Next up, we want to install our dependencies. 

```
RUN npm install
```

It's the RUN instruction again, and like before up here, it is running commands to install and build stuff. It's adding content. So we get another layer. 

This particular app listens on port 8080, so we'll expose that.

```
EXPOSE 8080
```

Just metadata again. 

And then last but not least we need to run an app. 

```
ENTRYPOINT ["node","./app.js"]
```

And this is relative to WORKDIR up here, okay? Anyway, these here don't really create layers. Just metadata in the config. 

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa2.PNG" />

We'll go, docker image build. 

```
docker image build -t psweb .
```

So we're done. That means that if we do `docker image ls`, there it is.

Let's run a container from it. 

```
docker container run -d --name web1 -p 8080:8080 psweb
```

## Digging Deeper

Your build context can absolutely be a remote Git repo.

```
docker image build -t psweb https://github...
```

So we got the base image at the bottom. Next up, we have the maintainer label. To do this, Docker spins up a new container, and it churns out a layer. Once the layer's created, it gets rid of that intermediate container. Poof, gone.

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa3.PNG" />

Spin up that intermediate container, produce the layers, and shoot the containers. That's the process. 

Only the layers that contain actual data are kept. So, even though it looks like these entry points and expose instructions create layers, they really don't. At least, not ones that we keep. So if we do:

```
docker  history psweb
```

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa4.PNG" />
 
We can see that only four of these layers actually contain any data. The others are just metadata. So if we inspect the image, like this

```
docker image inspect psweb
```

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa5.PNG" />

## Multi-stage Builds

I am hoping by now that it is abundantly clear that size matters when it comes to images. And smaller is definitely better. So, with smaller images, we're talking about things like faster builds, faster deployments, less money wasted on storage, and less attack surface for the bad guys to aim at. 

So, running our apps with a minimal OS and minimal supporting packages, that is the gold standard. 

Small, small images is what we're aiming for.

The problem is that we start out with a pretty big image. You know, one that's got a shedload of packages and build tools that, who knows? Maybe we'll need them. But maybe we won't. Then we copy in a bunch more tools that we know we need. We throw in our app code, and then we do the build. And the image is huge! And this is where it falls apart, right? Then we ship it to production with all the build stuff left in! I mean, it's crazy! 

**Multi-stage builds** to the rescue. This here, what we're talking about now, is official support to technology from the core Docker project. We're talking a single Dockerfile and zero scripting wizardry.

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa6.PNG" />

The main thing to note is that we have got multiple `FROM` instructions, three in this example. Each one marks a distinct **build stage**. So, internally, right, they're numbered from zero at the top. But even better, we can give them friendly names. So, stage zero here is storefront, and stage one here is appserver. And then the last stage, production. 

<img src="https://github.com/KiraDiShira/Docker/blob/master/ContainerizingAnApp/Images/caa7.PNG" />

Stage zero of the build here takes a giant image, great for building, okay? Not so great for production. Well, it builds some app code and it spits out even bigger image. Stage one is the same. Pulls a big image, adds some stuff in, throws out an even bigger one. At this point in the build, we've got four images, and inside of them, we have got a boatload of build time machinery, but just a small amount of production code. 

```
docker image build -t multistage .
```
