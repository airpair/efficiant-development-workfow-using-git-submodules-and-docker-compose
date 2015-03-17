## Problem

Since we hired our first remote developer at [Continuous Software](http://continuous.in.th), we realized how important it is to streamline our development workflow. Here are some typical scenarios we wanted to avoid while on-boarding new developers into our complex projects made of many applications:

* Missing stack components: node.js, PHP, PostgreSQL, ...
* Unclear overview of the projects components / applications
* Local configuration conflicts: listening ports, database settings, ..

Moreover, and from my personal experience, we - developers - loose way to often our first days in a company setting up our development environment and trying to understand how everything should work together instead of going right in trying to understand what the company applications are actually doing.

## Solution

Before going on with further details on the how, I would like to directly show you the development workflow we putted in place for our projects.

Each project has its own Team on Bitbucket (same as an Organization on Github). We then create one repository per application (e.g. `api`, `dashboard`, `cpanel`) within this Team. In addition to those, we create a repository called `development`. Along the submodules, we only have a `README.md` and a `docker-compose.yml` in the `development` repository. It looks like this:

```
kytwb@continuous:~/path/to/<project>/$ ls -la
total 40
drwxrwxr-x 11 kytwb amine 4096 Mar 14 16:30 .
drwxr-xr-x  4 kytwb amine 4096 Nov  1 20:17 ..
drwxr-xr-x 20 kytwb amine 4096 Mar 11 14:24 api
drwxr-xr-x 11 kytwb amine 4096 Mar  3 13:21 cpanel
drwxr-xr-x 10 kytwb amine 4096 Mar 12 11:37 dashboard
-rw-r--r--  1 kytwb amine 2302 Mar  2 15:28 docker-compose.yml
drwxrwxr-x  9 kytwb amine 4096 Mar 14 16:30 .git
-rw-r--r--  1 kytwb amine  648 Dec 22 17:20 .gitmodules
-rw-r--r--  1 kytwb amine 1706 Dec 17 16:41 README.md
```

To get to quickly setup, when a new developer join a project, we ask him to browse to the `development` repository on Bitbucket and to follow the instructions in the `README.md`. The instructions are as follow:

```
$ git -v
$ docker -v
$ docker-compose -v
$ git clone git@bitbucket.com:<project>/development.git <project> && cd <project>
$ git submodule init && git submodule update
$ git submodule foreach npm install
$ docker-compose up -d
```

At this stage, our developer has everything up and running in local.

![Efficiant development workfow using Git submodules and Docker Compose](http://i.imgur.com/i65giSK.jpg)

## How-To

Let's now develop on how-to setup the described workflow.

### Prerequistes

```
$ git -v
$ docker -v
$ docker-compose
```

Our development stack is completly based on Docker. Therefore, our developers first need to install Docker. They don't need to get very familiar with Docker at this point but by using it on their development stack, we indirectly introduce them to the world of containers, which will later be used as a bridge to explain them how we use Docker for Continuous Integration, Continuous Delivery, etc.  
We do not document how-to [install Docker](https://docs.docker.com/installation/) in our `README.md` as it should not be a problem - otherwise

We first started using docker-compose to orchestrate our containers in our development stack when it was still called [Fig](http://fig.sh). Docker then acquired Fig and rebranded it to Docker Compose. There was a proposal to merge Docker Compose within Docker binary, but it has been rejected for many reasons so Docker Compose is another binary listed as a  prerequiste.  
Same here, [installing Docker Compose](http://docs.docker.com/compose/install/) is not documented and should not be a problem.

### Setup the repositories

As we said earlier, you have to create a `development` repository and a repository per application. Here we will also create `api`, `dashboard` and `cpanel`. When these repositories are created, we focus on setting up the `development` repositories.

```
git clone git@bitbucket.com:<project>/development.git <project> && cd <project>
```

We are now going to add our applications repositories as submodules of the `development` repository. To do so, just type the following command lines:

```
$ git submodule add git@bitbucket.org:<project>/api.git
$ git submodule add git@bitbucket.org:<project>/dashboard.git
$ git submodule add git@bitbucket.org:<project>/cpanel.git
```

This will have for effect to create a `.gitmodules` file at the root of your `development` repository. That's how the developers are able to fetch all the applications at once when cloning the `development` repository and running:

```
$ git submodule init && git submodule update
```

For more information about submodules, refer to [the official Git documentation](http://git-scm.com/book/en/v2/Git-Tools-Submodules).

### Containerize all the things ™

We now have our `development` repository setup with access to all the different applications one `cd` away. We are now going to containerize all the applications and configure them thanks to the orchestration tool we previously mentionned: Docker Compose.

Let's start with the `api` application. Open `docker-compose.yml`, declare a container for the API and choose a base image for your container. In our case, our stack being based on Node.js we will just use the official Node.js image:

```yaml
api:
  image: dockerfile/nodejs
```

At this stage, running the command `docker-compose up -d` should create a container called `<project>_api_1` that does nothing (instant exit). You can run `docker-compose ps` to get informations about the containers orchestrated by your `docker-compose.yml`.

Let's configure the `api` container a little be more so it can be functionnal.  
To achieve so, we have to:

- mount the source code within the container
- declare which command to run to run the application
- expose appropriate port(s) to access the application

That translates into:

```yaml
api:
  image: dockerfile/nodejs
  volumes:
    - ./api/:/app/
  working_dir: /app/
  command: npm start
  ports:
    - "8000:8000"
```

By running `docker-compose up -d` now, you should have your `api` application up and running on `http://localhost:8000`. It might crash for many reasons; feel free to check the container logs with `docker-compsoe logs api`.  
At this stage, I suspect the `api` crashing because it can't connect to its database. So let's add a `database` container and make it available to our `api` container.

```yaml
api:
  image: dockerfile/nodejs
  volumes:
    - ./api/:/app/
  working_dir: /app/
  command: npm start
  ports:
    - "8000:8000"
  links:
    - database
database:
  image: postgresql
  ports:
    - "5432:5432"
```

By creating the `database` container and linking it to the `api` one, we made discovery of the `database` possible within the `api`. Try to display the environment of your API (e.g. `console.log(process.env)`) and you should be able to see variables such as `POSTGRES_1_PORT_5432_TCP_ADDR` and `POSTGRES_1_PORT_5432_TCP_PORT`. This is the variables you will use in the API config files related to your database.

The database, through the link, is now considered a dependecy of the api. It means Docker Compose will always start the database container before starting the api container.

We are now going to describe the other applications the same way we did for the API. This time, we will link the `api` to the `dashboard` and `cpanel` app so they can both resolve the API container address/port through the environment variables `API_1_PORT_8000_TCP_ADDR` and `API_1_PORT_8000_TCP_PORT`.

```yaml
api:
  image: dockerfile/nodejs
  volumes:
    - ./api/:/app/
  working_dir: /app/
  command: npm start
  ports:
    - "8000:8000"
  links:
    - database
database:
  image: postgresql
dashboard:
  image: dockerfile/nodejs
  volumes:
    - ./dashboard/:/app/
  working_dir: /app/
  command: npm start
  ports:
    - "8001:8001"
  links:
    - api
cpanel:
  image: dockerfile/nodejs
  volumes:
    - ./api/:/app/
  working_dir: /app/
  command: npm start
  ports:
    - "8002:8002"
  links:
    - api
```

Same as you did with your API configuration file for the database, you can know edit the dashboard and cpanel applications to use the environment variables to resolve the API instead of having it hardcoded.

Now you can run `docker-compose up -d` again, followed by `docker-compose ps`:

```
kytwb@continuous:~/path/to/<project>$ docker-compose up -d
Recreating <project>_database_1...
Recreating <project>_api_1...
Creating <project>_dashboard_1...
Creating <project>_cpanel_1...
kytwb@continuous:~/path/to/<project>$ docker-compose ps
docker-compose ps
     Name                   Command              State                Ports            
--------------------------------------------------------------------------------------
<project>_api_1         npm start                     Up         0.0.0.0:8000->8000/tcp
<project>_dashboard_1   npm start                     Up         0.0.0.0:8001->8001/tcp
<project>_cpanel_1      npm start                     Up         0.0.0.0:8002->8002/tcp
<project>_database_1    /usr/local/bin/run            Up         0.0.0.0:5432->5432/tcp  
```

Things should be up and running.  
Your api should be available on http://localhost:8000.  
Your dashboard should be available on http://localhost:8001.  
Your cpanel should be available on http://localhost:8002.

## Going Further

### Local Routing

After running all the containers using `docker-compose up -d`, we can access our applications on `http://localhost:<application_port>`. With the current setup, we could easily make local routing happen using [`jwilder/nginx-proxy`](https://github.com/jwilder/nginx-proxy) in a way that we could access local applications with URLs reflecting more what's in production. For instance, we could access the local version of `http://api.domain.com` directly by typing `http://api.domain.local`.

The [`jwilder/nginx-proxy`](https://github.com/jwilder/nginx-proxy) image makes things pretty straight forward. Just create describe a new container in your `docker-compose.yml` that we will call `nginx`. Configure the container as described in [`jwilder/nginx-proxy`](https://github.com/jwilder/nginx-proxy)'s README (mount your Docker daemon socket, expose port 80). Then, you only have to add the extra environment variables `VIRTUAL_HOST` and `VIRTUAL_PORT` to your existing containers, as follow:

```yaml
api:
  image: dockerfile/nodejs
  volumes:
    - ./api/:/app/
  working_dir: /app/
  command: npm start
  environment:
    - VIRTUAL_HOST=api.domain.local
    - VIRTUAL_PORT=8000
  ports:
    - "8000:8000"
  links:
    - database
database:
  image: postgresql
dashboard:
  image: dockerfile/nodejs
  volumes:
    - ./dashboard/:/app/
  working_dir: /app/
  command: npm start
  environment:
    - VIRTUAL_HOST=dashboard.domain.local
    - VIRTUAL_PORT=8001
  ports:
    - "8001:8001"
  links:
    - api
cpanel:
  image: dockerfile/nodejs
  volumes:
    - ./api/:/app/
  working_dir: /app/
  command: npm start
  environment:
    - VIRTUAL_HOST=cpanel.domain.local
    - VIRTUAL_PORT=8002
  ports:
    - "8002:8002"
  links:
    - api
nginx:
  image: jwilder/nginx-proxy
  volumes:
    - /var/run/docker.sock:/tmp/docker.sock
  ports:
    - "80:80"
```

The `nginx` container will check all the containers running on the Docker daemon (through the mounted `docker.sock` file) and create appropriate nginx config file for each container that have the `VIRTUAL_HOST` environment variable setup.

To finish setting up the local routing as we wish, we now have to add all the VIRTUAL_HOST we used to our `/etc/hosts`. To do so, I manually use the node.js `hostile` package, but I guess this could be automated the same way [`jwilder/nginx-proxy`](https://github.com/jwilder/nginx-proxy) dynamically works with nginx config files. To be digged.

You can know `docker-compose up -d` again and access your application on the same url as they are in production, replacing the `.com` TLD with the `.local` TLD.

## Suggestions?

This post being published on airpair, feel free to fork it and contribute by adding your own suggestions for the Going Further section. If you see any mistakes in what I previously wrote, same, feel free to correct.