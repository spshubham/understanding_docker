# Learning Docker with Node.js

Step-by-step notes from our Docker lessons, using one small Node.js web server.

## 1. What is Docker?

Docker is a tool for packaging applications and running them in isolated environments called **containers**.

A Node.js application needs its source code, a compatible Node.js runtime, and any application dependencies. Installing these separately on different computers can lead to missing packages or version differences: the familiar "it works on my machine" problem.

Docker lets us package the application and its runtime together into an **image**. Running containers from that image gives the application a consistent environment.

## 2. What are an image and a container?

| Term | Meaning | Our example |
|---|---|---|
| Docker | The tool that builds images and manages containers. | We use the `docker` command in the terminal. |
| Image | A packaged template containing the application, runtime, files, and startup configuration. | `understanding-docker:latest` contains Node.js and `server.js`. |
| Container | An instance created from an image. Starting it runs the application. | Our Node.js server running inside its own environment. |

An image does not run by itself. A container can be running or stopped.

One image can be used to create multiple containers. Each instance has its own processes and filesystem view.

```text
                     Node.js app image
                            |
                  +---------+---------+
                  |                   |
                  v                   v
             Container 1         Container 2
             App instance        App instance
```

## 3. Do we still need the codebase?

Yes. Docker needs our source code when building this application's image.

| What we want to do | What happens to the code |
|---|---|
| Build the image | Docker copies the application code into the image. |
| Run the built image later | The code is already inside the image, so a separate source folder is not required. |
| Run it on another computer | That computer needs a compatible Docker setup and access to the image. The source files do not need to be copied separately. |
| Update the application | Edit the source, rebuild the image, and create a new container from the updated image. |

```text
Source code + Dockerfile
           |
           v
      Build an image
           |
           v
 Image contains a copy of the code
           |
           v
 Create and run a container
```

The code is still required by the application: it uses the copy inside the image. Editing `server.js` on your computer does not automatically update an existing image or container in this setup.

## 4. Our project

```text
understanding_docker/
|-- Dockerfile
|-- README.md
`-- server.js
```

| File | Purpose |
|---|---|
| `server.js` | Our small Node.js HTTP server. |
| `Dockerfile` | Instructions for packaging the server into an image. |
| `README.md` | The concepts and commands covered in our lessons. |

## 5. Run the Node.js application locally

This is our complete `server.js`:

```javascript
const http = require("node:http");

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hello from Node.js!\n");
});

server.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});
```

The application uses Node's built-in HTTP module, so we do not need to install any npm packages.

From the project folder, run:

```powershell
node server.js
```

This uses **Node.js installed on your computer**. The terminal prints:

```text
Server running at http://localhost:3000
```

Open [http://localhost:3000](http://localhost:3000) in a browser. The response is:

```text
Hello from Node.js!
```

Press **Ctrl+C** in the terminal to stop this local server before running the container on the same computer port.

## 6. Check that Docker is ready

On this Windows setup, start **Docker Desktop** and wait for its engine to be ready before building or running containers.

These commands help check the setup:

| Command | What it checks |
|---|---|
| `node --version` | The Node.js version installed on your computer, used for the local example. |
| `docker --version` | Whether the Docker command-line tool is available and which version it is. |
| `docker info` | Whether the command-line tool can communicate with the Docker engine; also shows engine information. |

Having the Docker command installed and having its engine running are separate things. If a Docker command says it cannot connect to the engine, check that Docker Desktop has started successfully.

The container uses the Node.js runtime provided by its image. It does not use the Node.js installation on your computer.

## 7. What is a Dockerfile?

A Dockerfile is a text file containing instructions Docker follows to build an image.

Our file is named `Dockerfile`, without an extension:

```dockerfile
FROM node:24

WORKDIR /app

COPY server.js .

EXPOSE 3000

CMD ["node", "server.js"]
```

| Instruction | Meaning |
|---|---|
| `FROM node:24` | Start from the official Node.js image with Node.js 24 already installed. |
| `WORKDIR /app` | Create `/app` if needed and make it the working folder inside the image. |
| `COPY server.js .` | Copy the project's `server.js` into that working folder, producing `/app/server.js`. |
| `EXPOSE 3000` | Record that the application uses container port 3000. Publishing it to the computer is a separate step. |
| `CMD ["node", "server.js"]` | Save the default startup command: run `node server.js` when the container starts. |

`CMD` does not start our server during the image build. We do not need `RUN npm install` because this application has no external npm dependencies.

See the [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) and the [official Node.js image](https://hub.docker.com/_/node).

## 8. Why is CMD written as separate strings?

This is **exec form**:

```dockerfile
CMD ["node", "server.js"]
```

The first item identifies the program, and the following items supply its arguments:

| Item | Meaning |
|---|---|
| `"node"` | The program to execute. |
| `"server.js"` | The argument telling Node which file to run. |

This is not the correct way to express that command:

```dockerfile
CMD ["node server.js"]
```

The array now contains one command string. Docker does not split it at the space into a program and an argument.

You can also write the complete command using **shell form**:

```dockerfile
CMD node server.js
```

In shell form, a shell interprets the command. Our Dockerfile uses exec form, which keeps the program and arguments explicit and avoids an extra shell for interpreting them.

See [shell and exec form](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form).

## 9. Build our first image

Open a terminal in the project folder and run:

```powershell
docker build -t understanding-docker .
```

This means: **build an image using this folder and name it `understanding-docker`.**

| Part | Meaning |
|---|---|
| `docker` | Use the Docker command-line tool. |
| `build` | Build an image by following a Dockerfile. |
| `-t` | Assign an image name and optional tag. |
| `understanding-docker` | The image name we chose. |
| `.` | Use the current folder as the build context: the files available to the build. |

Docker looks for `Dockerfile` in that folder by default. In our workspace, the current folder is `H:\It_Prep\understanding_docker`.

The name is our choice. This alternative would name the image `my-node-app`:

```powershell
docker build -t my-node-app .
```

Our lessons use `understanding-docker` consistently. Because we did not specify a tag after a colon, Docker uses the default tag `latest`:

```text
understanding-docker:latest
```

`latest` is a tag label; it does not automatically update the image when we edit code.

On the first build, Docker downloads the required base-image layers. It then creates `/app`, copies `server.js`, and saves the configuration. Later builds can reuse downloaded layers and cached steps when applicable.

The result is an image stored locally in Docker. Building it does not start our application.

See the [build command](https://docs.docker.com/reference/cli/docker/buildx/build/) and [build cache](https://docs.docker.com/build/cache/).

## 10. What does the dot mean in different places?

| Example | Meaning of `.` |
|---|---|
| `docker build -t understanding-docker .` | The current folder on your computer, used as the build context. |
| `COPY server.js .` | The destination working folder inside the image: `/app`, set by `WORKDIR`. |

We also discussed this broader copy instruction:

```dockerfile
COPY . .
```

The first dot selects the contents of the build context. The second dot is the destination working folder inside the image. Our actual Dockerfile copies only `server.js`, which is the application file we need.

See [COPY](https://docs.docker.com/reference/dockerfile/#copy).

## 11. See the image we built

```powershell
docker image ls understanding-docker
```

This lists local images with the repository name `understanding-docker`.

| Column | Meaning |
|---|---|
| `REPOSITORY` | The image name, here `understanding-docker`. |
| `TAG` | Its tag, here `latest`. |
| `IMAGE ID` | The image identifier. |
| `CREATED` | When the image was created. |
| `SIZE` | The reported image size. |

An image listing does not tell us whether a container is running. For that, we use `docker ps`.

See [docker image ls](https://docs.docker.com/reference/cli/docker/image/ls/).

## 12. Create and run a container

```powershell
docker run -p 3000:3000 understanding-docker
```

| Part | Meaning |
|---|---|
| `docker run` | Create a new container from an image and start it. |
| `-p` | Publish a container port through a port on your computer. |
| `3000:3000` | Map computer port 3000 to container port 3000. |
| `understanding-docker` | The image used to create the container. With no explicit tag, this refers to `latest`. |

The port order is:

```text
-p COMPUTER_PORT:CONTAINER_PORT
```

Docker runs the image's configured command, `node server.js`. Visit [http://localhost:3000](http://localhost:3000) to reach the application:

```text
Browser
   |
   v
Computer port 3000
   |
   v
Container port 3000
   |
   v
Node.js app -> Hello from Node.js!
```

`EXPOSE 3000` in the Dockerfile records the intended port. The `-p 3000:3000` option creates the mapping we use from the browser.

This command keeps the terminal attached to the container and displays its output. Press **Ctrl+C** to stop this foreground demo.

Each `docker run` creates a new container. The command does not select an old stopped container to restart.

See [docker run](https://docs.docker.com/reference/cli/docker/container/run/).

## 13. List running containers

Open a second terminal while the app is running, then execute:

```powershell
docker ps
```

This shows currently running containers.

| Column | Meaning |
|---|---|
| `CONTAINER ID` | An identifier for the container. |
| `IMAGE` | The image used to create it. |
| `COMMAND` | The container's startup command, possibly shortened in the display. |
| `CREATED` | How long ago the container was created. |
| `STATUS` | Its current state; `Up` means running. |
| `PORTS` | Container ports and any published port mappings. |
| `NAMES` | The container's name. Docker generates one when we do not supply a name. |

The image name and container name are separate things. Our image is `understanding-docker`; the container may have a generated name.

See [docker ps](https://docs.docker.com/reference/cli/docker/container/ls/).

## 14. List stopped containers too

```powershell
docker ps -a
```

**`-a` means all.** It includes stopped containers as well as running ones.

| Command | What it lists |
|---|---|
| `docker ps` | Running containers. |
| `docker ps -a` | All existing containers, including stopped ones. |

After stopping our demo with Ctrl+C, its container no longer appears in `docker ps`, but it remains visible in `docker ps -a`, usually with an `Exited` status.

**Stopping a container does not delete it.** A stopped container can be started again. Neither stopping it nor listing it removes the image used to create it.

See [listing all containers](https://docs.docker.com/reference/cli/docker/container/ls/#all).

## 15. Start an existing stopped container

First, find our stopped container:

```powershell
docker ps -a
```

Look for `understanding-docker` in the `IMAGE` column and `Exited` in `STATUS`. Copy its `CONTAINER ID` or `NAMES` value.

Then run:

```powershell
docker start CONTAINER_NAME_OR_ID
```

Replace `CONTAINER_NAME_OR_ID` with that actual value. This command starts the same container again, using its saved startup command and port mapping.

| Command | What it does |
|---|---|
| `docker run -p 3000:3000 understanding-docker` | Creates and starts a new container from the image. |
| `docker start CONTAINER_NAME_OR_ID` | Starts an existing stopped container. |

`docker start` returns control to the terminal while the app runs. Check it with `docker ps`, then open [http://localhost:3000](http://localhost:3000) for our container with the saved `3000:3000` mapping.

To start a stopped container and stay attached to its output instead, use:

```powershell
docker start -a CONTAINER_NAME_OR_ID
```

Here, `-a` means **attach**. You can watch the demo's output and use Ctrl+C to stop it. In `docker ps -a`, the same letter means **all**: option meanings depend on the command.

Attaching displays new output; it does not replay earlier log messages. If the container is already running, this command does not restart the application to produce another startup message.

See [docker start](https://docs.docker.com/reference/cli/docker/container/start/).

## 16. Why can an attached terminal show no logs?

An attached terminal can be quiet while the application is running correctly. In our example, the startup message had already been printed before the terminal attached.

Our server prints this line only when it starts:

```text
Server running at http://localhost:3000
```

It does not log each browser request. `res.end("Hello from Node.js!\n")` sends an HTTP response to the browser; it does not print that text in the terminal.

Open a second terminal to check the running container with `docker ps`. To read its saved logs, run:

```powershell
docker logs CONTAINER_NAME_OR_ID
```

Replace the placeholder with the container's actual name or ID. You may see several startup messages if the same container has been started multiple times.

To read the saved logs and keep watching for new output, run:

```powershell
docker logs -f CONTAINER_NAME_OR_ID
```

Here, **`-f` means follow**. If the application writes nothing new, the display waits. Ctrl+C exits this log viewer while the container keeps running.

See [attaching to container output](https://docs.docker.com/reference/cli/docker/container/attach/) and [docker logs](https://docs.docker.com/reference/cli/docker/container/logs/).

## 17. Stop a running container

In a free terminal, run:

```powershell
docker stop CONTAINER_NAME_OR_ID
```

Replace the placeholder with the container's name or ID from `docker ps`. For example, for our container whose ID begins with `e9e2`:

```powershell
docker stop e9e2
```

Docker asks the application to stop and waits for it to exit. If it does not exit within the allowed time, Docker stops it forcefully, so the command can take a few seconds.

Confirm the result with:

```powershell
docker ps -a
```

The container should show `Exited`. It still exists, and its image remains available. Use `docker start CONTAINER_NAME_OR_ID` when you want to start it again.

Remember that Ctrl+C while watching `docker logs -f` only exits the log viewer. Use `docker stop` to stop the container itself.

See [docker stop](https://docs.docker.com/reference/cli/docker/container/stop/).

## 18. Run a container in detached mode

So far, `docker run` has kept the terminal attached to the container. To run the container in the background, add `-d`:

```powershell
docker run -d -p 3000:3000 understanding-docker
```

| Part | Meaning |
|---|---|
| `docker run` | Create and start a new container. |
| `-d` | Run it in **detached mode**, which means in the background. |
| `-p 3000:3000` | Map computer port 3000 to container port 3000. |
| `understanding-docker` | Create the container from this image. |

Docker prints the new container's long ID and immediately returns control to the terminal. The Node.js application continues running in the background.

Check it with:

```powershell
docker ps
```

Then open [http://localhost:3000](http://localhost:3000). Because detached mode does not show application output directly, use `docker logs CONTAINER_NAME_OR_ID` when you want to read it.

Only one container can normally publish computer port 3000 at a time. If Docker reports that the port is already allocated, find the container using it with `docker ps` and stop that container before running this command.

Stop the detached container with:

```powershell
docker stop CONTAINER_NAME_OR_ID
```

Detached mode changes where the container runs relative to your terminal; closing the terminal does not stop it.

See [detached mode in docker run](https://docs.docker.com/reference/cli/docker/container/run/#detach-from-the-container--d---detach).

## 19. Give a container a name

If we do not provide a name, Docker generates one. Use `--name` to choose a clear name yourself:

```powershell
docker run -d --name node-app -p 3000:3000 understanding-docker
```

| Part | Meaning |
|---|---|
| `-d` | Run the container in the background. |
| `--name node-app` | Give this new container the name `node-app`. |
| `-p 3000:3000` | Map computer port 3000 to container port 3000. |
| `understanding-docker` | Use this image to create the container. |

You can now use `node-app` instead of its container ID:

```powershell
docker logs node-app
docker stop node-app
docker start node-app
```

The container name and image name serve different purposes:

| Name | What it identifies |
|---|---|
| `understanding-docker` | The reusable image. |
| `node-app` | One container created from that image. |

Container names must be unique in Docker, including among stopped containers. If Docker says `/node-app` is already in use, that container already exists. You can start it with `docker start node-app`; we will learn how to remove containers in the next lesson.

Also make sure no other running container is already publishing computer port 3000.

See [assigning a container name](https://docs.docker.com/reference/cli/docker/container/run/#name---name).

## 20. Remove a container

Stopping and removing are different operations:

| Operation | Result |
|---|---|
| `docker stop node-app` | Stops the application, but keeps the container so it can be started again. |
| `docker rm node-app` | Deletes the stopped container. It cannot be started again. |

Docker normally requires the container to be stopped before removal:

```powershell
docker stop node-app
docker rm node-app
```

Confirm that it has been removed:

```powershell
docker ps -a
```

`node-app` should no longer appear. The `understanding-docker` image is not deleted, so you can create a fresh container from it:

```powershell
docker run -d --name node-app -p 3000:3000 understanding-docker
```

Removing a container releases its name, which is why `node-app` can now be used again. It also removes that container's writable filesystem changes, configuration, and logs. Data that must survive container removal will later be stored in a volume.

Docker also supports `docker rm -f node-app`, which forcefully stops and removes a running container. Prefer `docker stop` followed by `docker rm` so the application gets a chance to shut down normally.

See [docker rm](https://docs.docker.com/reference/cli/docker/container/rm/).

## 21. Remove an image

Use this command to remove our locally built image:

```powershell
docker image rm understanding-docker
```

The shorter alias is:

```powershell
docker rmi understanding-docker
```

Both commands mean the same thing. Docker interprets the missing tag as `latest`, so this targets `understanding-docker:latest`.

Containers depend on the image from which they were created. If Docker reports that the image is in use, first stop and remove those containers:

```powershell
docker stop node-app
docker rm node-app
docker image rm understanding-docker
```

Check the result with:

```powershell
docker image ls
```

Removing the image from Docker does not delete `Dockerfile`, `server.js`, or any other project file. You can recreate the image at any time:

```powershell
docker build -t understanding-docker .
```

The base image `node:24` may remain because it is a separate image and its downloaded layers can be reused in later builds.

| Command | Removes |
|---|---|
| `docker rm node-app` | A container created from an image. |
| `docker image rm understanding-docker` | The locally stored application image. |

This command only affects the image stored in your local Docker engine. It does not delete source code from GitHub or remove an image from an online registry.

See [docker image rm](https://docs.docker.com/reference/cli/docker/image/rm/).

## 22. Command quick reference

Run the project commands from the folder containing `server.js` and `Dockerfile`.

| Command or action | Purpose |
|---|---|
| `node --version` | Check the local Node.js version. |
| `docker --version` | Check the Docker CLI version. |
| `docker info` | Check engine connectivity and information. |
| `node server.js` | Run the app using Node.js on your computer. |
| `docker build -t understanding-docker .` | Build and name the application image. |
| `docker image ls understanding-docker` | Find the built image locally. |
| `docker image rm understanding-docker` | Remove the locally stored application image. |
| `docker run -p 3000:3000 understanding-docker` | Create and start a container with a port mapping. |
| `docker run -d -p 3000:3000 understanding-docker` | Create and start a container in the background. |
| `docker run -d --name node-app -p 3000:3000 understanding-docker` | Create a background container named `node-app`. |
| `docker ps` | List running containers. |
| `docker ps -a` | List running and stopped containers. |
| `docker start CONTAINER_NAME_OR_ID` | Start an existing stopped container using its saved settings. |
| `docker start -a CONTAINER_NAME_OR_ID` | Start a stopped container and attach to its output. |
| `docker stop CONTAINER_NAME_OR_ID` | Stop a running container, keeping it available to start again. |
| `docker rm CONTAINER_NAME_OR_ID` | Remove a stopped container. |
| `docker logs CONTAINER_NAME_OR_ID` | Read a container's saved logs. |
| `docker logs -f CONTAINER_NAME_OR_ID` | Read saved logs and follow new output; Ctrl+C exits the viewer. |
| Ctrl+C in the foreground app terminal | Stop the local server or the foreground container demo. |

The Dockerfile instructions (`FROM`, `WORKDIR`, `COPY`, `EXPOSE`, and `CMD`) belong in the Dockerfile. They are not commands to enter directly into PowerShell.

## 23. Keeping this learning project on GitHub

Project changes and learning notes are committed and pushed to the `main` branch of [spshubham/understanding_docker](https://github.com/spshubham/understanding_docker).

GitHub stores the source files and Dockerfile. The image built with `docker build` is stored locally in Docker; pushing a Git commit does not automatically upload that image.
