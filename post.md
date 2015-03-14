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


We are now going to add our applications repositories as submodules of the `development` repository. To do so, just type the following command lines:

```
$ git submodule add api
$ git submodule add dashboard
$ git submodule add cpanel
```

### Containerize all the things ™

## Going Further

### Local Routing

After running all the containers using `docker-compose up -d`, we can access our applications on `http://localhost:<application_port>`. With the current setup, we could easily make local routing happen using [`jwilder/nginx-proxy`](https://github.com/jwilder/nginx-proxy) in a way that we could access local applications with URLs reflecting more what's in production. For instance, we could access the local version of `http://api.domain.com` directly by typing `http://api.domain.local`. Here's how.

## Suggestions?

This post being published on airpair, feel free to fork it and contribute by adding your own suggestions for the Going Further section. If you see any mistakes in what I previously wrote, same, feel free to correct.




