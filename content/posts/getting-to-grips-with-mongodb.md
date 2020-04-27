---
title: "Getting to grips with this project"
date: 2020-04-27T22:00:00+02:00
draft: false
toc: false
images:
tags:
  - mongo
  - docker
  - setup
  - learning
---
Getting used to how Mongo works has been relatively challenging. I am not that used to working with databases anyways, and Mongo is already quite different to SQL by definition. I'm going to detail some of the things I've struggled with over the last few weeks in this post, documenting some of my failures as well as some successes.

## /docker-entrypoint-initdb.d
As it turns out, initialising Mongo is not as easy as just throwing some scripts in this folder and hoping for the best. Mongo, like many other databases, uses ACID [^1] properties when making modifications to the databases, in order to maintain database integrity. Naturally, I had somewhat expected that Mongo would apply these properties to each of the queries used in the init scripts. What I _hadn't_ expected, however, was that the entire script had to execute _without error_ before Mongo would commit those queries to the database. This led to much confusion surrounding why my scripts weren't taking. So far, I've jumped into the deep end with regards to how I've started learning to use Mongo, but perhaps starting at the beginning may have helped me realise the issue much sooner.

Additionally, it didn't help that I'd written the init scripts incorrectly. Instead of setting variables, I was calling them, and then sometimes calling variables that didn't exist. Overall, not ideal.

Looking forward, I'm considering seperating the init scripts so that one failure doesn't lead to the entire init script failing. However, I think that this would be bad practice - either the database should be completely set up, or it shouldn't be set up at all. I have since put all the initialisation steps into one file, and added a schema validation to a little test schema in that initialisation file. It works well, so now I need to think about how I want to store prefabs a little more. I've included the script below:

``` bash
#!/bin/bash

echo 'Creating opendc user and db'

mongo opendc --host localhost \
        --port 27017 \
        -u $MONGO_INITDB_ROOT_USERNAME \
        -p $MONGO_INITDB_ROOT_PASSWORD \
        --authenticationDatabase admin \
        --eval "db.createUser({user: '$OPENDC_DB_USERNAME', pwd: '$OPENDC_DB_PASSWORD', roles:[{role:'dbOwner', db: '$OPENDC_DB'}]});"
MONGO_ROOT_CMD="mongo $OPENDC_DB --host localhost --port 27017 -u $MONGO_INITDB_ROOT_USERNAME -p $MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase admin"

#echo 'Creating opendc db schema...'
MONGO_CMD="mongo $OPENDC_DB -u $OPENDC_DB_USERNAME -p $OPENDC_DB_PASSWORD --authenticationDatabase $OPENDC_DB"
$MONGO_CMD --eval 'db.createCollection("environments", {
	validator: {
		$jsonSchema: {
			bsonType: "object",
			required: ["name"],
			properties: {
				name: {
					bsonType: "string",
					description: "The name of the environment i.e. Production, or Compute Cluster"
				},
				datacenters: {
					bsonType: "object",
					required: ["name, location, length, width, height"],
					properties: {
						name: {
							bsonType: "string",
							description: "The name of the datacenter i.e. eu-west-1, or Science Building"
						},
						location: {
							bsonType: "string",
							description: "The location of the datacenter i.e. Frankfurt, or De Boelelaan 1105"
						},
						length: {
							bsonType: "double",
							description: "The physical length of the datacenter, in centimetres"
						},
						width: {
							bsonType: "double",
							description: "The physical width of the datacenter, in centimetres"
						},
						height: {
							bsonType: "double",
							description: "The physical height of the datacenter, in centimetres"
						}
					}
				}
			}
		}
	}
});'
```

## Database design
This is quite difficult. It's quite hard to consider how to represent the data within a prefab. It's relatively simple to decide what constitutes a prefab at minimum - the bare minimum to execute workloads. But, those bare minimums can be children of parent objects such as racks. I'm undecided whether racks are mandatory (I don't think they constitute part of a prefab; if you have a rack with space for another prefab, that second prefab doesn't need a rack, for example), but I feel like it is definitely bad data center practice to run an environment without a rack.

My current train of thought is that you (at minimum) should have an environment, which contains at least one data center, containing at least one rack, containing at least one machine, and so forth. Maybe we can enforce checks so that prefabs that take up rack space require a rack to be added to the datacenter, but don't actually contain a rack as part of the stored prefab.

It's also quite difficult to actually represent these ideas in JSON. I think I will endeavour to find another way to represent them during the design phase next week. I think I have an idea of how to actually design this (writing this post has also helped a little), so I should try and get that on paper.

##Docker
On a somewhat more positive note, working with Docker has been relatively problem-free. Docker on macOS has it's own quirks sometimes, but overall it has been super easy to iterate through designs and debug the issues I've been having. Definitely a much shallower learning curve than MongoDB, but I'm also a lot more familiar with virtualization and containerization. I only wish I could rebuild faster, but I am beginning to suspect that I may be able to only rebuild my containers each time instead of the whole project. 


[^1]: Atomicity, Consistency, Isolation and Durability