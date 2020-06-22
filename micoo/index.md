Micoo
--
![dashboard.png](./images/dashboard.png)

## About Micoo
[Micoo](https://github.com/Mikuu/Micoo) is a pixel based screenshots compare solution for visual regression test, some characters Micoo provides:

* a web application, for inspecting test results, making visual mismatch decision and maintain baseline build,
* an engine service, for comparing the latest screenshots against baseline screenshots, based on pixel difference,
* a methodology, about how to do visual regression test with service,
* quick local setup and server side deployment with Docker Compose,

Micoo does `NOT`:
* take screenshots from your SUT application,
* process screenshots before doing visual compare,
* provide Email notification for compare mismatch,
* provide user management to distinguish `teams`,

So, what Micoo targets at is the most stable and straightforward function for comparing screenshots. Micoo is not, and probably, would never be a powerful thing, like `JVM`, but hope to be an always useful helper, like `string.replace()`.

## Installation

According to different purposes, there are 3 ways to launch Micoo as described below.

> all the below steps require the project [repository](https://github.com/Mikuu/Micoo) be cloned to your local, and all the commands are executed from the project's root path.

### launch with Docker images from Docker Hub

```commandline
cd env
docker-compose up
```

this is the simplest approach to launch Micoo. Unless you would like to launch Micoo with your own source code changes, you can always use the images from Docker Hub.

### launch with local source code

- create exchange directory
```commandline
mkdir exchange
npm install
```

- containerize micoo-nginx service
```commandline
cd env/nginx/containerize
./build-image.sh
```

- launch environment
```commandline
cd env
docker-compose -f docker-compose.env.yaml up
```

- start micoo-dashboard service
```commandline
cd dashboard
npm install
MICOO_DB_USERNAME=micoo-user export MICOO_DB_PASSWORD=micoo-password npm start
```

- start micoo-engine service
```commandline
cd engine
npm install
MICOO_DB_USERNAME=micoo-user export MICOO_DB_PASSWORD=micoo-password npm start
```

- start micoo-postern service
```commandline
cd postern
npm install
MICOO_DB_USERNAME=micoo-user export MICOO_DB_PASSWORD=micoo-password npm start
```

with above commands, it should be successfully launch Micoo from the source code locally.

### launch with local docker images

once you have debugged your code change successfully, you can containerize them locally:

- containerize micoo-nginx service
```commandline
cd env/nginx/containerize
./build-image.sh
```

- containerize micoo-dashboard service
```commandline
cd dashboard
./build-image.sh
```

- containerize micoo-engine service
```commandline
cd engine
./build-image.sh
```

- containerize micoo-postern service
```commandline
cd postern
./build-image.sh
```

then, launch Micoo locally with your own docker images:

- launch all from local image
```commandline
cd env
docker-compose -f docker-compose.local.yaml up
```

the last thing, if you wish, you can publish your own docker images to your private or cloud registry, e.g. AWS ECR, to share with teams.

## Usage

## Clients

## CI integration

## Postern
