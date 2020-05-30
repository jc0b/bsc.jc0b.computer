---
title: "Working With BSON"
date: 2020-05-30T19:26:46+02:00
draft: false
toc: false
images:
tags:
  - BSON
  - mongodb
  - database
  - JSON
---

So it turns out that mongoDB doesn't work with _just_ JSON [^1].

When I started working with mongoDB (and especially Pymongo), I was operating under the assumption that mongoDB would export objects in JSON. You can insert regular JSON objects into mongoDB without issue (even directly via the CLI on the mongoDB server), so this seemed like a logical assumption to make. I was wrong. One of the main tests of my export/import functionality is to re-import something that I have exported. For quite some time, I was confused as to why the "JSON" files I was exporting weren't directly serializable into JSON objects when I was importing them again. It turns out that mongoDB doesn't use pure JSON for exports.

BSON [^2] is a format devised by mongoDB, and is how mongoDB stores objects internally. The technical details aren't very interesting - mongoDB simply stores files in a binary format internally, and converts them back to something human-readable when you export them. However, it is important to note that the human-readable format is _not_ JSON, but is deceptively similiar: BSON uses single quotes (') where JSON uses double quotes(").

As a result, we need to use an additional library from pymongo to convert the BSON to a (JSON) string, and then re-import that string as a JSON object. That process looks a little bit like this:
```python
import json
from bson.json_util import loads, dumps

bson = prefabs_collection.find({'name': prefab_name})
json_string = dumps(bson) #convert BSON representation to JSON
chosen_prefab = json.loads(json_string[1:-1]) #load as a JSON object
```

In the last line, we leave out the first and last characters of the JSON string. This is because mongoDB's `find()` will return all objects with that name, and so returns an array of JSON objects. This implementation is not ideal: we would be better off using `find_one()` to get only a single object (not contained in array). However, this would likely require some additional search criterion to avoid users that store prefabs with the same names retrieving eachothers prefabs, as `find_one()` will return the first match in the collection.

To help me visualize how this process of converting from BSON to JSON works, I sketched it out into the diagram below.
![BSON to JSON flow](/images/bson-2-json.jpeg)

## What's next?
Next up, I need to strip mongo's `_id` field from exported JSON objects before writing to a file. Mongo won't import an object if it's declared ID already exists in the database, which is messing up my test cases and would also make cloning prefabs a little trickier.
I also need to determine how only sub-components of prefabs can be copied, which may be quite complicated.

[^1]: JavaScript Object Notation 
[^2]: Binary JSON(? This is an interesting acronym)