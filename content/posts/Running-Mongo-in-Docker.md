---
title: "Running Mongo in Docker"
date: 2020-04-07
draft: false
toc: false
images:
tags:
  - docker
  - mongo
  - getting started
---

Geting mongo to run in Docker was pretty straight-forward... for the most part. I started with creating a new directory under the main project to keep my mongo-related stuff in, and added a Dockerfile for the mongo image. The Dockerfile is fairly simple:
``` docker
FROM mongo:4.2.5
MAINTAINER Jacob Burley <j.burley@vu.nl>

# Import init script
ADD mongo-init-opendc-db.sh /docker-entrypoint-initdb.d
ADD mongo-opendc-schema.sh /docker-entrypoint-initdb.d
```

I've frozen the version number in order to make sure that the version stays the exact same throughout development, in order to avoid updates breaking functionality and causing a lot of debugging headaches.

I've also added two ```ADD``` lines, to add scripts to the docker-entrypoint-initdb directory on the docker host at buildtime. Every script in that folder will be run during container startup, so I plan to use them to initialize a new database, ```opendc```, for storing prefab information in, as well as creating the default user and assigning access rights. These scripts currently don't work properly, and I'm wondering whether it is because they are not executed in the intended order. I need to figure out what order they are executed in, but worst case I will just condense all initialization tasks into one single script. (You can read more about these scripts [here](https://bsc.jc0b.computer/posts/initialising-mongodb/))

## docker-compose.yml
OpenDC uses docker-compose in order to actually build and run containers, so I need to add mongo to the ```docker-compose.yml``` file.
I added both mongo, as well as mongo express, as services within docker-compose. Mongo express is a web frontend for mongodb, which allows me to view databases and their contents. I added the below to ```docker-compose.yml```:
``` yaml
mongo:
    build:
        context: ./mongodb
    restart: on-failure
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: rootpassword
      MONGO_INITDB_DATABASE: admin
      OPENDC_DB: opendc
      OPENDC_DB_USERNAME: opendc
      OPENDC_DB_PASSWORD: opendcpassword
    ports:
      - 27017:27017

mongo-express:
    image: mongo-express
    restart: on-failure
    ports:
      - 8082:8081
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: rootpassword
```
I've added some environment variables for mongo for use in the scripts added in the Dockerfile, and set the context for the mongodb build to be the folder with my Dockerfile in it. I'm only using mongo express for testing purposes, so I'm not really too worried about fleshing it out with its own Dockerfile and folder. Both services will restart on failure, and both have their various configuration settings pre-set. 
I think it's probably going to be worth removing the environment variables from mongo express, because the opendc database isn't viewable by the root user of mongo, so mongo express isn't very helpful in debugging that database at the moment.