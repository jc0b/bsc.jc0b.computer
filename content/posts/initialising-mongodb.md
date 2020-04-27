---
title: "Initialising Mongodb"
date: 2020-04-17
draft: false
toc: false
images:
tags:
  - docker 
  - mongo
  - setup
---

Initialising the databases was a little tricky to begin with - my understanding of Docker and mongo is not spectactular. However, I learned that mongo would run any ```.sh``` or ```.js``` files in ```/docker-entrypoint-initdb.d``` on container startup. This seems to be the recommended way of creating your databases and inserting any initial items you would like to have at the beginning.

I started with using ```.js``` files using some tutorials found in various documentation, but I couldn't seem to get it to work. Being more familiar with bash scripting, I decided to switch to using shell files.

I ended up with two shell files:
``` bash
#!/bin/bash
echo 'Creating opendc user and db'
mongo opendc \
        --host localhost \
        --port 27017 \
        -u $MONGO_INITDB_ROOT_USERNAME \
        -p $MONGO_INITDB_ROOT_PASSWORD \
        --authenticationDatabase admin \
        --eval "db.createUser({user: 'opendc', pwd: 'opendcpassword', roles:[{role:'dbOwner', db: 'opendc'}]});"
MONGO_CMD = "mongo opendc --host localhost --port 27017 -u $MONGO_INITDB_ROOT_USERNAME -p $MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase admin"
```
``` bash
#!/bin/bash

echo 'Creating opendc db schema...'

MONGO_CMD = "mongo opendc --host localhost --port 27017 -u $OPENDC_DB_USERNAME -p $OPENDC_DB_PASSWORD --authenticationDatabase opendc"

eval $MONGO_CMD --eval "db.createCollection('environments'); db.createCollection('rooms'); db.createCollection('datacenters');"
```

These are a bit messy, but the overall gist is that the first file creates a user on a database called `opendc`. In mongo, standard practice seems to be not to initialize things before they are needed. As a result, I can create the `opendc` database by only specifying a user with access to it. Mongo recognises that the database doesn't exist, and creates it at the moment it is needed.
The second script creates collections within the opendc database. I haven't yet decided what collections I want in it yet, so the three that are there are more or less placeholders, but the premise is the same for as many collections as I want to add. 

## Next steps:
Next, I'm looking to put all this functionality in one script. I'm still not sure in which order these are executed. I assume that it works in alphabetical order on the filename, but these initialisation files don't need to be seperate anyways.