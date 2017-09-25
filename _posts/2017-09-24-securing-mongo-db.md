---
layout: post
title: "Securing containerized MongoDB"
comments: true
description: "How to quickly deploy a mongo container and avoid paying ransom"
keywords: "docker mongodb ransomfree"
---

**Getting the Docker container**

```bash

$ docker run --name mongoDB -d -p 27217:27017 -v /mongo-data:/data/db mongo --auth

```
**Note** mongoDB is the name of my contianer yours may differ. Also I am exposing a 27217 port not the standard 27017.

The above line will pull the latest official mongo docker image, and deploy it. The -p option exposes the port 27217 publicly and the -v option helps you persist data even if your docker container gets killed.

The last --auth flag is very important for our use case, it forces the mongodb container to check for authentication for all requests which is what we want.

Once you have done this connect to your docker continer using:

```bash
$ docker exec -it mongoDB bash
```
**Note** mongoDB is the name of my contianer yours may differ.

Connect to the mongo service by:
```bash
$ mongo
```

Create a new admin user:
```bash
> db.createUser(
{
user: "super",
pwd: "superpassword",
roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
}
)
```

Create another user for controlling access to your db:
```bash
> db.createUser(
  {
    user: "client",
    pwd: "clientpassword",
    roles: [{ role: "readWrite", db: "appdb" } ]
  }
)
```

Now you can connect to your db by using

```python
MONGO_URL = 'mongodb://client:clientpasssword@<public_ip>:27217/appdb?authSource=admin'
```

```bash
mongo appdb -u client -p clientpassword --host <public_ip> --port 27217
```