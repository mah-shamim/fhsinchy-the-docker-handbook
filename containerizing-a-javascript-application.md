# Containerizing a JavaScript Application

Now that you've got some idea of creating images, it's time to work with something a bit more relevant. In this sub-section, you'll be working with the source code of the [fhsinchy/hello-dock](https://hub.docker.com/r/fhsinchy/hello-dock) image that you worked with on a previous section. In the process of containerizing this very simple application, you'll be introduced to volumes and multi-staged builds, two of the most important concepts in Docker.

To begin with, open up the directory where you've cloned the repository that came with this article. Code for the `hello-dock` application resides inside the sub-directory with the same name.

```text
.
├── index.html
├── package.json
├── public
│   └── favicon.ico
└── src
    ├── App.vue
    ├── assets
    │   └── docker-handbook-github.webp
    ├── components
    │   └── HelloDock.vue
    ├── index.css
    └── main.js
```

This is a very simple JavaScript project powered by the [vitejs/vite](https://github.com/vitejs/vite) project. Don't worry though, you don't need to know JavaScript or vite in order to go through this sub-section. Having a basic understanding of [Node.js](https://nodejs.org/) and [npm](https://www.npmjs.com/) will suffice.

Just like any other project you've done in the previous sub-section, you'll begin by making a plan of how you want this application to run. In my opinion, the plan should be as follows:

* Get a good base image for running JavaScript applications i.e. [node](https://hub.docker.com/_/node).
* Set the default working directory inside the image.
* Copy the `package.json` file into the image.
* Install necessary dependencies.
* Copy rest of the project files.
* Start the vite development server by executing `npm run dev` command.

This plan should always come from the developer of the application that you're containerizing. If you're the developer yourself, then you should already have a proper understanding of how this application needs to be run. Now if you put the above mentioned plan inside `Dockerfile.dev`, the file should look like as follows:

```text
FROM node:lts

EXPOSE 3000

WORKDIR /app

COPY ./package.json .
RUN npm install

COPY . .

CMD [ "npm", "run", "dev" ]
```

Explanation for this code is as follows:

* The `FROM` instruction here sets the official Node.js image as the base giving you all the goodness of Node.js necessary to run any JavaScript application. The `lts` tag indicates that you want to use the long term support version of the image. Available tags and necessary documentation for an image is usually found on [node](https://hub.docker.com/_/node) hub page.
* Then the `WORKDIR` instruction sets the default working directory to `/app` directory. By default the working directory of any image is the root. You don't want any unnecessary files sprayed all over your root directory, do you? Hence you change the default working directory to something more sensible like `/app` or whatever you like. This working directory will be application to any consecutive `COPY`, `ADD`, `RUN` and `CMD` instructions.
* The `COPY` instruction here copies the `package.json` file which contains information regarding all the necessary dependencies for this application. The `RUN` instruction executes `npm install` command which is the default command for installing dependencies using a `package.json` file in Node.js projects. The `.` at the end represents the working directory.
* The second `COPY` instruction copies rest of the content from the current directory \(`.`\) of the host filesystem to the working directory \(`.`\) inside the image.
* Finally, the `CMD` instruction here sets the default command for this image which is `npm run dev` written in `exec` form.
* The vite development server by default runs on port `3000` hence adding an `EXPOSE` command seemed like a good idea, so there you go.

Now, to build an image from this `Dockerfile.dev` you can execute the following command:

```text
docker image build --file Dockerfile.dev --tag hello-dock:dev .

# Step 1/7 : FROM node:lts
#  ---> b90fa0d7cbd1
# Step 2/7 : EXPOSE 3000
#  ---> Running in 722d639badc7
# Removing intermediate container 722d639badc7
#  ---> e2a8aa88790e
# Step 3/7 : WORKDIR /app
#  ---> Running in 998e254b4d22
# Removing intermediate container 998e254b4d22
#  ---> 6bd4c42892a4
# Step 4/7 : COPY ./package.json .
#  ---> 24fc5164a1dc
# Step 5/7 : RUN npm install
#  ---> Running in 23b4de3f930b
### LONG INSTALLATION STUFF GOES HERE ###
# Removing intermediate container 23b4de3f930b
#  ---> c17ecb19a210
# Step 6/7 : COPY . .
#  ---> afb6d9a1bc76
# Step 7/7 : CMD [ "npm", "run", "dev" ]
#  ---> Running in a7ff529c28fe
# Removing intermediate container a7ff529c28fe
#  ---> 1792250adb79
# Successfully built 1792250adb79
# Successfully tagged hello-dock:dev
```

A container can be run using this image by executing the following command:

```text
docker container run --detach --publish 3000:3000 --name hello-dock-dev hello-dock:dev

# 21b9b1499d195d85e81f0e8bce08f43a64b63d589c5f15cbbd0b9c0cb07ae268
```

Now visit `http://127.0.0.1:3000` do see the `hello-dock` application in action.

![](.gitbook/assets/hello-dock-dev.png)

Congratulations on running your first real-world application inside a container. The code you've just written is okay but there is one big issue with it and a few places where it can be improved. Let's begin with the issue first.

## Working with Volumes

If you've worked with any front-end JavaScript framework before, you should know that the development servers in these frameworks usually come with a hot reload feature. That is if you make a change in your code, the server will reload automatically reflecting any changes you've made immediately.

But if you make any change in your code right now, you'll see nothing happening to your application running in the browser. The reason behind this is the fact that you're making changes in the code that you have in your local file system but the application you're seeing in the browser resides inside the container file system.

![](.gitbook/assets/local-vs-container-file-system.svg)

To solve this issue, Docker has something called [volumes](https://docs.docker.com/storage/volumes/). You've already had a brief encounter of volumes in the [Working With Executable Images](container-manipulation-basics.md#working-with-executable-images) sub-section under [Container Manipulation Basics](container-manipulation-basics.md) section.

Using volumes, you can easily mount one of your local file system directory inside a container. Instead of making a copy of the local file system, the volume can reference the local file system directly from inside the container.

![](.gitbook/assets/bind-mounts.svg)

This way, any changes you make to your local source code will reflect immediately inside the container,  triggering the hot reload feature of vite development server. Changes made to the file system inside the container will reflect on your local file system as well and that's how the [fhsinchy/rmbyext](https://hub.docker.com/r/fhsinchy/rmbyext) image deletes files from local file system.

As you've learned in the previous section, volumes can be created using the `--volume` or `-v` option for the `container run` or `container start` commands. Just to remind you the generic syntax is as follows: 

```text
--volume <local file system directory absolute path>:<container file system directory absolute path>:<read write access>
```

Kill and remove your previously started `hello-dock-dev` container, open up a terminal window inside the `hello-dock` project directory and start a new container by executing the following command:

```text
docker container run --publish 3000:3000 --name hello-dock-dev --volume $(pwd):/app hello-dock:dev

# sh: 1: vite: not found
# npm ERR! code ELIFECYCLE
# npm ERR! syscall spawn
# npm ERR! file sh
# npm ERR! errno ENOENT
# npm ERR! hello-dock@0.0.0 dev: `vite`
# npm ERR! spawn ENOENT
# npm ERR!
# npm ERR! Failed at the hello-dock@0.0.0 dev script.
# npm ERR! This is probably not a problem with npm. There is likely additional logging output above.
# npm WARN Local package.json exists, but node_modules missing, did you mean to install?
```

Keep in mind, I've omitted the `--detach` option and that's to demonstrate a very important point. As you can see that the application is not running at all now.

That's because although the usage of a volume solves the issue of hot reloads, it introduces another one. If you have any previous experience with Node.js, you may know that the dependencies of a Node.js project lives inside the `node_modules` directory on the project root.

Now that you're mounting the project root on your local file system as a volume inside the container, the content inside the container gets replaced along with the `node_modules` directory containing all the dependencies.  Hence the `vite` package goes missing.

This problem here can be solved using an anonymous volume. An anonymous volume is identical to a regular volume except the fact that you don't need to specify the source of the volume here. The generic syntax for creating an anonymous volume is as follows:

```text
--volume <container file system directory absolute path>:<read write access>
```

So the final command for starting the `hello-dock` container with both volumes should be as follows:

```text
docker container run --detach --publish 3000:3000 --name hello-dock-dev --volume $(pwd):/app --volume /app/node_modules hello-dock:dev

# 53d1cfdb3ef148eb6370e338749836160f75f076d0fbec3c2a9b059a8992de8b
```

Here, Docker will take the entire `node_modules` directory from inside the container and tuck it away in some other directory managed by the Docker daemon on your host file system and will mount that directory as `node_modules` inside the container.

## Improving the Dockerfile.dev

The `Dockerfile.dev` you wrote in the previous section is okay but there are some improvements that can be made. So open up the `Dockerfile.dev` file once again and update it's content to look like as follows:

```text
FROM node:lts

EXPOSE 3000

USER node

RUN mkdir -p /home/node/app

WORKDIR /home/node/app

COPY ./package.json .
RUN npm install

CMD [ "npm", "run", "dev" ]
```

Let me explain the changes I've made one by one here:

* On line 5, I've added a new instruction `USER node`. By default Docker runs containers as the root user and according to [Docker and Node.js Best Practices](https://github.com/nodejs/docker-node/blob/master/docs/BestPractices.md) this can pose a security threat. So a better idea is to run as a non-root user whenever possible. The node image comes with a non-root user named `node` which you can set as the default user using the `USER` instruction.
* Given now you're using a non-root user, you do not have write access to the container root hence using `/app` is not possible. So instead of the root, you'll have to use a directory that is writable by the `node` user. On line 7, the `RUN mkdir -p /home/node/app` instruction creates a directory called `app` inside the home directory of the `node` user. The home directory for any non-root user in Linux is usually  `/home/<user name>` by default.
* On line 9, the `WORKDIR /home/node/app` instruction sets the `/home/node/app` directory as the working directory instead of the `/app` directory.
* Another change is the removal of the `COPY . .` instruction. This is done because mounting the project root as a volume inside the container working directory makes the `COPY` instruction redundant.

Apart from these four changes, rest of the file remains same. Now rebuild the image and try running a new container with the resultant image.

## Performing Multi-Staged Builds
