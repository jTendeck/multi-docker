# Docker

## Common Commands

```bash
docker run <image-name> <command-override>
```

- Creates a docker container based on a given `image-name`
  - You can override the startup command by typing in your own `command-override`. For example, running: `docker run busybox echo lol jk` will print "lol jk".
- `docker run` is actually composed of two different commands: `docker create` and `docker start`. `docker create` initializes the container while `docker start` runs the container startup command.

```bash
docker ps
```

- List all containers currently running on machine
  - Can include `--all` flag to see all containers that have been run

```bash
docker create <image-name>
```

- Initialize container based on image name. Outputs a container ID.

```bash
docker start <container-id>
```

- Start a container. Can also restart exited containers.
  - To view output of container in console you can attach the `-a` flag

```bash
docker system prune
```

- Deletes all containers and downloaded images. Note that this also deletes cached images, meaning they will have to be redownloaded.

```bash
docker logs <container-id>
```

- View all data emitted from a container. Does not actually start a container.

```bash
docker stop <container-id>
```

- Stops container more gracefully. Sends SIGTERM signal to process running in container allowing it to finish up, finish writing files, etc. Usage is recommended over `docker kill`
  - If container does not stop in 10 seconds, SIGKILL signal is sent. More below.

```bash
docker kill <container-id>
```

- Sends SIGKILL signal to process running in container which stops the process _right now._

```bash
docker exec -it <container-id> <command>
```

- Execute commands inside a given container
  - `-it` allows you to provide input to the container.
  - Note the -it flag is two seperate commands: -i and -t. -i = attach input of our terminal to docker container STDIN. -t = format output from container more prettily

```bash
docker exec -it <container-id> sh
```

- Open up a terminal inside of container and run commands. Negates having to rerun the above command.
  - To exit container try either `ctrl + c` or `ctrl + d` or type `exit` then hit enter
  - Can also attach -it and sh to docker run, but this will override the startup command. Typically you want to start up a container and then attach a shell to it

```bash
docker build <directory-containing-Dockerfile>
```
- Build image based on Dockerfile

```bash
docker commit -c <change> <container-id>
# Eg: docker commit -c 'CMD ["redis-server"]' 3h12gjkd1
```
- Create a new image based on an existing container with some kind of change. This command is generally not used; rather changes are made to Dockerfile.

```bash
docker run -p <local-machine-port>:<container-port> <image-id>
# Eg: docker run -p 8080:8080 acrophobicowl/simpleweb 
```
- Run container, mapping local machine port to container port

```bash
docker attach <container-id>
```
- Attaches terminal input/output to running container stdin/stdout/stderr

:::important
When you run a container, multiple processes may start, such as `npm run test`. Note however that `docker attach` only attaches the terminal to the primary process running in a given container (you can see processes by running `docker exec -it <container-id> sh` then running `ps`. The process with an id of `1` is the primary process. For this reason, you will not be able to control other processes, such as Jest.
:::

## Creating an Image

To create a custom image you must outline a few steps in a `Dockerfile`. These steps are:

```Dockerfile
# 1. Use an existing docker image as a base
FROM alpine

# 2. Download and install dependencies. 'apk' = alpine package manager
RUN apk add --update redis

# 3. Tell the image what to do when it starts as a container
CMD ["redis-server"]
```

To create the image, you then run:
```bash
docker build <directory-containing-Dockerfile-and-project-files>
# Eg: docker build . if running in current directory
```

The above command wll spit out a image ID. You can then create a container with this iage using `docker run <image-id>`

When you build an image, temporary images or snapshots are created after each step (or at least after the FROM, RUN, and CMD steps). The next step then takes this image snapshot and builds upon in. For example, first alpine is downloaded in the FROM step, then the image is saved. The RUN step then takes this image, installs redis and the image is saved, along with all the downloaded redis files. The CMD step then takes that image with its redis files, and inserts "redis-server" as the primary command in the image from the previous step. This file image is then saved and can be used by the user.

When you build an image, the temporary images are cached so that if you rebuild an image again in the future it will build faster. Eg: having `RUN apk add --update redis` then `RUN apk add --update gcc` will hit the cache if the commands are run again in that order, but if `RUN apk add --update gcc` then `RUN apk add --update redis` is run, the cache will not be hit, and the image will have to rebuild - taking a longer amount of time.

### Tagging an Image

Building an image outputs a hard to remember ID. You can tag your image with a more memorable ID - or tag - by using:

```bash
docker build -t <your-docker-ID>/<repo-or-project-name>:<version> <Dockerfile-location>
# Eg: docker build -t acrophobicowl/my-project:latest .
```

You can now create a container using: `docker run acrophobicowl/my-project` - don't have to sepecify version; `latest` will be used by default

## Copying Build Files

Remember that a container is segmented portion of the harddrive (among other things) and so it does have access to the rest of your harddrive. If you have a `package.json` file in your project and try running `npm isntall` inside the container, it will fail since the container does not have access to that file.

To remedy this, you must include the `COPY` instruction in your Dockerfile in order to copy an necessary files from your project to the container.

```Dockerfile
FROM ...

COPY <machine-source> <container-destination>
# Eg: COPY ./ ./
# Note that <machine-source> is relative to the build context.
# Want to copy before we run commands below. For example, we might want to copy a package.json file before running npm install

RUN ...
```

After we run `docker build` we can then access this container.

__NOTE:__ If you make a change to the project, your change will not be reflected in the running. You must instead stop the container and rebuild it to see your changes. 

## Container Port Mapping

Our containers can reach out into the world and install dependencies (at least by default), but in order to access our application, we must map a port on our computer to one of the ports in the container. Note that this is done at runtime, not within the Dockerfile. To map ports we run the following:

```bash
docker run -p <local-machine-port>:<container-port> <image-id>
# Eg: docker run -p 8080:8080 acrophobicowl/simpleweb 
```

## Specify Working Directoy

When using the COPY command this will copy the files to the home directory of the container. This is bad practice as it might conflict/overwrite files and directories that live there. Instead we need to specify a working directory for where our project can live. This can be done by including `WORKDIR` inside of Dockerfile:

```Dockerfile
FROM ...

# Should come before COPY instruction
WORKDIR /usr/app

COPY ...
```

## Minimizing Cache Busting, Rebuilds

Some commands can take a long time to run, such as `npm install`. If you make a change to an unrelated file - say `index.js` - and you rebuild the container, it will have to reinstall all the NPM modules. To save time, consider breaking up the COPY command into multiple pieces. Consider:

```Dockerfile
WORKDIR ...

# Copy just package.json
COPY ./package.json ./
# Then install dependencies
RUN npm install
# Then copy rest of files.
COPY ./ ./

CMD ...
```

## Multiple Images in One Dockerfile

Running multiple images in a single container - say `node` and `redis` - is a valid approach, though it has some issues. If we need our application to scale - that is, create more containers to handle more traffic - the `node` and `redis` instance are copied. This is fine for the `node` instance, but redis houses persistent data that we might want share across all containers. If redis is copied, then there will be different copies of data across the containers. We want to create seperate node and redis containers because of this.

## Docker Compose

Docker Compose is meant to the reduce repetitive code/commands needed to build, run, and network containers. Here's an example that builds a node and redis container:

```yaml
version: '3'

services:
  redis-server:
    image: redis
  node-app:
    build: . # Look in current dir for Dockerfile, and build using that.
    ports:
      - "4001:8081" # Map port 4001 on my machine to container port 8081
```

__NOTE:__ Containers created in docker-compose are networked to eachother; you do not have to sepcify any sort of networking between the services that get created.

In order to use redis in our application, we can connect to it simply by referring the the container name. Eg:

```js
const redis = require('redis')

const client = redis.createClient({
    host: 'redis-server', // Name of redis container
    port: 6379 // redis default port num; does not need to be specified
});
```

To run docker compose file use:

```bash
docker-compose up

# OR:
docker-compose up --build
# To freshly build container; should be run after making changes to code.
```
- Run docker-compose in background: `docker-compose up -d`

Stop containers: `docker-compose down`

:::tip
You can see containers running with the docker-compose command by running `docker-compose ps`. Note that this must be run within the directory container the docker-compose.yml file.
:::

### Restarting Containers

Your containers may crash for one reason or another - say an error in the code. You can get docker-compose to restart a container if this happens. There are a number of restart policies available:
- `"no"` : Never attempt a restart if container crashes (quotes are needed; `no` is a yaml keyword == `False`)
- `always` : Attempt restart if container crashes for any reson
- `on-failure` : Attempt restart only if container stops with an error code (ie: an exit code other than 0)
- `unless-stopped` : Alywas restart unless we - the developers - forcibly stop it

These are added to the docker-compose.yml file:

```yml
services:
  node-app:
    image: .
    restart: always
```

## Production Grade Workflow

We want two Dockerfiles - one for development, another for production. For the dev Dockerfile, we name in `Dockerfile.dev`. In the dev container, we just run the application in a devlopment setting - Eg: `npm run start`, but in production we actually want to build and then serve the application - Eg: `npm run build`

To build using the dev Dockerfile, you must specify the filename in the docker build command: `docker build -f Dockerfile.dev .`

## Docker Volumes

Rather than copying data from our machine to the container, we can reference the files on our machine instead. This allows us to do things like hot reloading, rather than having to rebuild/copy he files over each time we make a change to our code. To attach a volume to our container, run:
```bash
docker run -v <file/dir-on-local-machine>:<folder-in-container-we-map-to>
# I don't exactly understand how this command works yet, but here is an example.
# For some reason I could not get this to work. Though the docker-compose file works.
docker run -p 3000:3000 -v /app/node_modules -v $(pwd):/app <image-id>
```
- With the above example we map the present working directory (pwd) to the /app folder inside the container. With with `-v /app/node_modules` we are creating a placeholder reference and we are not mapping anything to... rather it is meant to just leave that folder alone inside the container

With this, do we need `COPY . .` command in the Dockerfile? Technically no, but it is a good idea to leave it in just in case something is changed with the volumes; this way no matter what happens all our files are still copied over to the container.

## Creating a docker-compose file with Volumes

```yml
version: '3'

services:
  web: 
    environment: # This section may be needed on Windows to enable hot reloading
      - CHOKIDAR_USEPOLLING=true 
    build:
      context: . # Look at the current directory...
      dockerfile: Dockerfile.dev # ... and build using Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - /app/node_modules # Leave this file set in stone inside the container
      - .:/app # Map the working director to the /app dir inside the container
```

### Running tests in a container

If you have a node container, you can run tests like so (after building an image):

```bash
docker run -it <image-id> npm run test
```

Don't forget `-it` else you won't be able to input commands.

### Attaching terminal to run tests

The problem with the method above is we have to rebuild if we want to see new tests to run in the console. If we want to "live reload" tests we can do the following: first run the container either using `docker-compose up` or `docker run`. Get the container ID then run:

```bash
docker exec -it <container-id> npm run test
```

With this, we can add/remove tests and we will see a live update after each save. The problem is this is a bit cumbersome to do. Rather what we can do is create a container dedicated to tests.

### Tests using docker-compose

We can update our `docker-compose.yml` with a test container which will show test output. Under `services` we can add:

```yml
  test:
    environment:
      - CHOKIDAR_USEPOLLING=true # Windows garbage
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - /app/node_modules
      - .:/app
    command: ["npm", "run", "test"] # Override defulut start command to start tests
```

:::warning
This implementation does not allow you to input commands into the terminal. Some test environments - like Jest - require you to input commands to run or quit running tests. You will not be able to do this.

Also, this does implementation does not appear to work currently on Windows (Home).
:::

You think you could run `docker attach <container-id>` to input commands put unfortuntealy you cannot do that.


## Creating a Production Environmnent

In our project we have a development server running. This is not efficient for a production environment. To be more efficient we need to build our project and then serve the files (eg: index.html or app.js) using a server. For this we will use nginx.

In order to do this, we need a multi-step process: one where we build our project using `node` and another where we serve the content using `nginx`.

To implement this we create a Dockerfile. Note the other Dockerfile we created is called `Dockerfile.dev`; this one is just called `Dockerfile`:
```Dockerfile
FROM node:alpine
WORKDIR '/app'
COPY package.json .
RUN npm install
COPY . .
RUN npm run build # builds project in app/build
 
FROM nginx
COPY --from=0 /app/build /usr/share/nginx/html # Copy contents of build to the dir that nginx will serve
```
:::note
Each `FROM` statement acts as the beginning of a new "block". So after the `node:alpine` block runs, we can no longer run commands for the `node:alpine` image, and now start running `nginx` commands.
:::

:::note
The `--from=0` command in the `nginx` block means we want to copy files from the block with an index of 0, ie: the node:alpine phase.
:::

:::note
Starting the nginx container automatically starts serving up files; no need to input any sort of start up/`RUN` commands.
:::

Once we have this Dockerfile configured, we just have to build it using `docker build .` (don't have to specify the Dockerfile name)

Once this is built, we can copy the image ID. Note that we must assign ports. By default nginx uses port `80`:

```bash
docker run -p 8080:80 <image-id>
```

Visit `localhost:8080` and you will be able to see your built application!

## Travis CI

Travis is used for continuous integration. With this we can do things like build a Docker image, run tests, etc. before we build and deploy to production.

We need to enable Travis for a repository (as of now Travis had been enabled on all repos on my GtiHub account; if creating a new project, Travis may have to be enabled for the new repo).

Once this is done, you must add a `.travis.yml` file to your project root. Adding this file will trigger Travis when you push code to your repo. Here is an example:

```yml
sudo: required # Give super user permissions
language: generic

services: 
  - docker # tell Travis to install Docker

before_install: # Run before tests are run; setup
  - docker build -t acrophobicowl/docker-react -f Dockerfile.dev . # Build image; we need to add a tag
  # so we can refer to this image in future commands

script: # commands needed to run test
  - docker run -e CI=true acrophobicowl/docker-react npm run test -- --coverage 
  # -- --coverage ensures the test exits when complete
```

To deploy to Elastic Beanstalk we have to do some additional configuration. First create a sample app running Docker in the EBS console. Next we create a new IAM username.

When creating this IAM user, we give it `programmatic access` only. Under permissions, we select `Attach existing policies directly`. The policy titled `AdministratorAccess-AWSElasticBeanstalk` was attached.

Copy or download the access keys + secret. Open Travis console and go to the pipeline for your repo. On the right side select `More Settings` and scroll down ntil you see Environment Variables. Add a `AWS_ACCESS_KEY` and `AWS_SECRET_KEY` with their respective values from AWS. Be sure to escape any special characters in teh env values with `\`.

Update the .travis.yml with the following:

```yml
sudo: required # Give super user permissions
language: generic

services: 
  - docker # tell Travis to install Docker

before_install: # Run before tests are run; setup
  - docker build -t acrophobicowl/docker-react -f Dockerfile.dev . # Build image; we need to add a tag
  # so we can refer to this image in future commands

script: # commands needed to run test
  - docker run -e CI=true acrophobicowl/docker-react npm run test -- --coverage 
  # -- --coverage ensures the test exits when complete

deploy:
  provider: elasticbeanstalk # Travis has built in config for some services, such as EBS
  region: us-west-2
  app: "docker-react" # EBS app name when creating EBS app
  env: "Dockerreact-env"  # EBS environment
  bucket_name: "elasticbeanstalk-us-west-2-944047486125" # S3 bucket used by EBS
  bucket_path: "docker-react" # Same as "app"
  on:
    branch: master # Only trigger when pushing code to master
  access_key_id: $AWS_ACCESS_KEY # Environment variables; stored in the Travis console
  secret_access_key: $AWS_SECRET_KEY
```

When pushing to github, code will then be deployed to AWS. Note that the EBS app will fail. The reason being we did not map our ports so EBS could access our containers.

To do this, we need to modify our `Dockerfile` - not `Dockerfile.dev`. We add the following:
```Dockerfile
FROM nginx
EXPOSE 80  # !!!
COPY --from=0 /app/build /usr/share/nginx/html
```

`EXPOSE` exposes port 80. EBS sees this and automatically routes our app to that port. 

In the demo I had to recreate follow steps 1, 2, and 4 from [this link](https://www.udemy.com/course/docker-and-kubernetes-the-complete-guide/learn/lecture/11437154#questions/12731600) to get the application running in the cloud.

## Feature Branches and Pull Requests

After you push code changes to a feaure branch in GitHub, you can open a new pull request to merge the changes with master. When you do this, Travis CI will run and you can see the outcome of the tests in pull request window.

After the tests run you can confirm the pull request and merge the code into master. When this happens, Travis CI will run again, executing the tests etc, and then deploy code - in our case to EBS.

## Calculate Fib Docker App

Folder: `08_118_Worker_Process_Setup`

This app is intended to showcase app creation using multiple docker containers. Basically it uses different dependencies like Postgres, Express, and Redis to calculate and store values at a given fib index.

When creating this app, different folders were created for different services. Eg: `worker`, `server`, etc. Inside each of these folders is a `keys.js` file which returns different values stored in process.env based on a key. Eg:

```js
module.exports = {
    redisHost: process.env.REDIS_HOST,
    redisPort: process.env.REDIS_PORT
};
```

Inside each folder is an `index.js` file which houses the main logic for that service. There is also a `package.json` folder, defining the depencies for each service. Each service has a `nodemon` dependency... not sure why yet, but it has something to do with Docker. There are also these scripts:

```json
"scripts": {
  "start": "node index.js",
  "dev": "nodemon"
}
```

## Dockerizing Application

We need to place Dockerfiles in each of the services (client (React), server, and worker). We also want to prevent ourselves from having to rebuild the image each time we make minor changes.

Dockerfile.dev for `client` (React application):

```Dockerfile
FROM node:alpine
WORKDIR '/app'
COPY ./package.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "start"]
```

Dockerfile.dev for `server` and `worker`:

```Dockerfile
FROM node:14.14.0-alpine # Needed to change because newer node version breaks code
WORKDIR "/app"
COPY ./package.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"] #!!!
```
Note that we use `npm run dev`. In the scripts section of `package.json`, we have this run `nodemon`. Nodemon allows the server the refresh after a change has been made to the code.

### Adding docker-compose.yml

We now have Dockerfile.dev files for each of our services. It's time to add docker-compose.yml file to run each of these containers in unison. We also have to pass in env variables, which are made accessible through the `keys.js` file and the `process.env` variables. Here is the `docker-compose.yml` file used in this project (so far):

```yml
version: '3'

services:
  postgres:
    image: 'postgres:latest'
    environment:
      - POSTGRES_PASSWORD=postgres_password # Need to add this according to recent updates to PG image
  redis:
    image: 'redis:latest'
  api:
    build:
      dockerfile: Dockerfile.dev # Want to use Dockerfile.dev...
      context: ./server # ... inside this folder
    volumes:
      - /app/node_modules # Do not try to overwrite this folder inside container
      - ./server:/app # Any changes in local ./server folder will be reflected inside /app in the container (except /app/node_modules, as mention above)
    environment:
      - REDIS_HOST=redis # Reference to redis 'service'
      - REDIS_PORT=6379 # Default redis port, according to redis docs
      - PGUSER=postgres
      - PGHOST=postgres # Reference to postgres 'service'
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password # Default pw according to pg image docs
      - PGPORT=5432
  client:
    stdin_open: true
    build:
      dockerfile: Dockerfile.dev
      context: ./client
    volumes:
      - /app/node_modules # Do not overwrite node_modules
      - ./client:/app # Any changes in local ./client folder will be reflected inside /app in the container (except /app/node_modules, as mentioned above)
  worker:
    build:
      dockerfile: Dockerfile.dev
      context: ./worker
    volumes:
      - /app/node_modules # Do not overwrite node_modules
      - ./worker:/app # Any changes in local ./worker folder will be reflected inside /app in the container (except /app/node_modules, as mentioned above)
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
```

:::note
With env variables, you can specify them at runtime using `variableName=value` or by taking value stored in your comute by just using `variableName`
:::

### Nginx Server
We also need to add in an nginx server. In our previous project we used this to serve up React website files (index.html, main.js, etc.) in a production environemt, but now we want to use them in development as well. We use Express to act as an API, giving us access to things like `/values/all` and `/values/current`. Nginx wil determine what service to use based on a given request - either the React services or Express service.

In our React application (inside `Fib.js`) we make requests to endpoints like: `/api/values/current` while in our server index.js file we make requests to endpoints like `/values/current`. Nginix will determine if the path starts with `/api/` or just `/` and route to the appropriate service (/api/ goes to our API, / goes to React)

Theoretically we could not use nginx and assign a port to either the React or Express service and send requests like: `values/current:3000` where our API is running on port 3000 but we don't want to do this because ports can be finnicky and change for various reasons. Using nginx prevents us from having to specify ports in our requests. Using paths like `/api/` is more explicit and less likely to change.

### Adding nginx

We must have a default.conf file to configure nginx. This will determine various behaviours of nginx, like routing / request to React (ie: client:3000) and /api/ requests to Express (ie: api:5000) (these services are known as 'upstream' servers)

:::note
We refer to these as `client:3000` and `api:5000` because `client` and `api` are the names we give them under `services` in the `docker-compose.yml` file. Each of these services run on a express server with the respect port number (client on express port 3000, api on express port 5000)
:::

### default.conf

Inside our project root we create another folder called `nginx`. Inside this folder we create a file called `default.conf` with these contents:

```conf
upstream client {
    server client:3000;
}

upstream api {
    server api:5000;
}

server {
    listen 80;

    location / {
        proxy_pass http://client;
    }

    location /api {
        rewrite /api/(.*) /$1 break;
        proxy_pass http://api;
    }
}
```

Basically this file is used to configure the service and determines where different request should be route to. So if a user sends a request to a path beginning with `/`, it is directed to the `client` running on port 3000, etc. 

:::note
Note the `rewrite` rule outlined in `location /api`. Inside our API service, we make requests to endpoints like `/values/current`, and not `/api/values/current`. The rewrite rule gets rid of the `/api` part of the path.
:::

### Nginx Dockerfile

Inside our `nginx` directory, we add `Dockerfile.dev`:

```Dockerfile
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

We the update our `docker-compose` file with the following:

```yml
services:
  # ...
  nginx:
    depends_on: # Resolves a possible edge case for error: failed (111: Connection refused)
      - api
      - client
    restart: always # We always want the service to be up
    build:
      dockerfile: Dockerfile.dev
      context: ./nginx
    ports:
      - '3050:80' # Nginx by default runs on port 80
```

## Running the service

The first time you run it is likely you will see errors. For this reason, it is usually a good idea to build once, cancel everything, then restart the container. So that might look like:
```bash
docker-compose up --build
# hit ctrl + c to stop
docker-compose up
```

## Resolving Websocket Connections

If we check the console we will notice a WebSocket connection error.

To remedy this, we need to modify the `default.conf` file located in the `nginx` directory.

In the error message we notice the path has `/sockjs-node` so we need to target that.

```conf
    location / { ... }

    location /sockjs-node {
        proxy_pass http://client;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }

    location /api { ... }
```

We need to rebuild our container again using `docker-compose up --build`. This should clear up the error.

## CI for Multiple Images

Inside each of our services we create a `Dockerfile`. This is very similar to our `Dockerfile.dev`, except rather than running in dev (ie: run a command like: `CMD ["npm", "run", "dev"]`) we change it to one more suitable for a production environment. As an example, in our `worker` and `server` directories we just change the `dev` CMD to `CMD ["npm", "run", "start"]`. For nginx, the two files are the same.

:::note
Even though the dev and prod Dockerfiles are very similar, it is still recommended to create different Dockerfiles for each environment.
:::

### Altering nginx Listen Port

In our example, we are going to be using two nginx servers: one for routing and another for serving up the React pages... just to add a bit more complexity and align more closely with the real world.

Inside our `client` directory, we add an `nginx` directory. We place a default.conf file with the following contents:

```conf
server {
    listen 3000;

    location / {
        root /usr/share/nginx/html;
        index index.htm;
    }
}
```

We change the Dockerfile to:
```Dockerfile
FROM node:alpine
WORKDIR '/app'
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
 
FROM nginx
EXPOSE 3000 # Want to serve React files on port 3000
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=0 /app/build /usr/share/nginx/html
```

### Travis setup

A new repo for the project was created, and at the project root we add a `.travis.yml` file. With this file, before pulling on Elastic Beanstalk, we want to push our files to Docker Hub. Here is the resulting file:

```yml

```

:::important
Be sure you are logged into docker before pushing to docker hub. This can be achieved by running `docker login`
:::

:::important
Since we are logging in to Docker in the travis.yml file, we need to add our login credentials as environment variables for the travis build project.
:::