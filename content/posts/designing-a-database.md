---
title: "Designing a Database"
date: 2020-04-27T14:00:00+02:00
draft: false
toc: false
images:
tags:
  - databases
  - mongo
  - design
---

For the purposes of representing prefabs in OpenDC, we need to be able to store these prefabs. We define prefabs as a collection of components that, at a minimum, must be able to execute workloads. This means that at the very least, a prefab is a single server. We think of servers as collections of components, and we can apply the same argument as we move upwards: racks can be a collection of servers, and datacenters can be a collection of racks. At the highest level, an environment is just a collection of datacenters.

OpenDC currently uses an SQL database to represent hardware, with foreign keys linking tables. If we were to export an object (i.e. a server), we would have to traverse the database recursively, visiting all the tables referenced in each foreign key used. This adds complexity to prefabs, so we started looking at other solutions. 
MongoDB is a document-oriented database which can represent parent-child relationships between objects. The benefit of this architecture is that we can insert objects directly as children to specified parent objects. Additionally, we can also query a parent object and receive its children objects without having to traverse multiple tables. This provides a much more logical approach to designing and using the database. 

## Maintaining database integrity
A major difference between MongoDB and SQL is that MongoDB doesn't use a schema. MongoDB will, by default, quite happily accept data in any form you give it. While this gives us a lot of flexibility when inserting documents into our database, it also means that it is much more important to specify our desired database design. In SQL, if you try to insert invalid data, it won't let you. In a default MongoDB configuration, it will let you do it, making it a little trickier to make assumptions about the state of our database. We want to take advantage of this flexibility; who knows how much more expandable future datacenter hardware will be? We can't make assumptions about everything anyone could possibly model in a datacenter, so we should only try and enforce certain characteristics of objects. We can do this using [JSON schema validation](https://docs.mongodb.com/manual/core/schema-validation/#schema-validation-json). 

While we want to enable users to be flexible in their design choices (potentially even allowing completely hypothetical scenarios), we also want to be able to make guarantees about certain object characteristics. Every CPU should have a core count, for example, with every rack having a specific size. Values that are important for the simulation of the datacenter should be enforced; no systems should be without memory, or power. We can then present these fields as mandatory in the frontend, so that the frontend can communicate mandatory characteristics to the user. In this way, we can have validation in the frontend as well as the backend of OpenDC, providing verbosity to users as well as maintaining the integrity of our database.
 