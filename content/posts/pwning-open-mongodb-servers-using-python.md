 ---
title: "Pwning Open MongoDB Servers Using Python"
date: 2019-09-13T10:31:07+05:30
draft: false
description: "Search for MongoDB servers using masscan and use python to enumerate open MongoDB servers. Honestly, I'm a bit late to the party, but I was able to score a **Critical** severity bug on **hackerone** using this."
author: "zer0ttl"
tags: ["python", "MongoDB", "web hacking", "bug bounty"]
---

## tldr;

Search for MongoDB servers using masscan and use python to enumerate open MongoDB servers. Honestly, I'm a bit late to the party, but I was able to score a **Critical** severity bug on **Hackerone** using this.

## Word of Caution :

The focus of this article is to educate the reader on the horrors of open MongoDB servers and how to avoid them. You may use the code shown in this post at your own risk. I am/will not be responsible for your actions.

## MongoDB :

<a href="https://www.MongoDB.com/" target="_blank" rel="noopener noreferrer">MongoDB</a> free and open-source NoSQL is a general purpose, document-based, distributed database. MongoDB uses the **document data model** contrary to the tabular data model used by relational databases.

Every record is stored as a JSON-like document in MongoDB. This means every document can have a different structure. You can learn more about MongoDB architecture <a href="https://www.MongoDB.com/MongoDB-architecture" target="_blank" rel="noopener noreferrer">here</a>.

Over the past couple of years there have been numerous reports of exposed MongoDB servers. Some have even been hijacked for ransom. You can read about some of the reports

* <a href="https://www.bleepingcomputer.com/news/security/MongoDB-databases-held-for-ransom-by-mysterious-attacker/" target="_blank" rel="noopener noreferrer">BleepingComputer.com : MongoDB Databases Held for Ransom by Mysterious Attacker</a>
* <a href="https://twitter.com/0xdude/status/813865069218037760" target="_blank" rel="noopener noreferrer">Twitter : 0xdude</a>
* <a href="https://nakedsecurity.sophos.com/2019/05/10/275m-indian-citizens-records-exposed-by-insecure-MongoDB-database/" target="_blank" rel="noopener noreferrer">Naked Security : 275m personal records swiped from exposed MongoDB database</a>.

By default, MongoDB uses port number **27017** for connections. A default configuration on older versions of MongoDB left the database open to external connections via the Internet. Also sometimes developers fail to enable authentication on MongoDB server leaving it exposed. You can still find some of these open MongoDB servers in the wild.

## Searching for MongoDB servers on the Internet

The most simple way to search for MongoDB servers is to query Shodan, BinaryEdge and Censys search engines.

* On Shodan, it is as simple as searching for `port:27017`

    ![MongoDB Servers on Shodan](/img/MongoDB-python/shodan_search1.png)

* As seen in the screenshot, Shodan has indexed over 60k MongoDB servers. However, we need to find the ones which do not have any authentication enabled. In order to find unauthenticated servers modify the query :

    `"MongoDB Server Information" all:"metrics"`

    ![MongoDB Servers on Shodan](/img/MongoDB-python/shodan_search2.png)

Another way of search for MongoDB servers is using **<a href="https://github.com/robertdavidgraham/masscan" target="_blank" rel="noopener noreferrer">masscan</a>**.

* `masscan` the internet `0.0.0.0/0` for port `27017` with a rate of `10000` packets/second and save the result in `MongoDB_server.list` in list `-oL` format.

```bash
masscan 0.0.0.0/0 -p 27017 --rate 10000 -oL MongoDB_servers.list
```

* ***Caution*** : You might want to limit your IP range to a subnet as scanning the internet might take a lot of time and some folks may not be happy with your scanning. **Choose your targets wisely.**

At this moment you can utilize NoSQL Manager for MongoDB or Studio 3T for MongoDB to pilage through the list of MongoDB servers. But the focus of this article is to show off my python kung-fu, so we will build our own python client.

## Python and MongoDB

#### Setting up the environment

I always prefer using a virtualenv whenever I'm working on a project. Its cleaner and simple to manage. If you are unfamiliar about pipenv, you can read more about it <a href="https://docs.pipenv.org/en/latest/" target="_blank" rel="noopener noreferrer">here</a> or read my <a href="#" target="_blank" rel="noopener noreferrer">blog post</a> on how to set it up.

```bash
# create a new directory mongo_parser and 'cd' into it
$ mkdir mongo_parser && cd $_
# use pipenv to create a new python3.7 virtual environment
$ pipenv --python=python3.7
# activate the new virtual environment
$ pipenv shell
# check the python version
(mongo_parser_Fhg43) $ python --version
```

#### PyMongo

**PyMongo** is a Python distribution containing tools for working with MongoDB, and is the recommended way to work with MongoDB from Python.

Install pymongo in the newly created virtual environment.

```bash
(mongo_parser_Fhg43)$ pipenv install pymongo
```

Run a local mongo server using docker

```bash
# pull the latest mongo image from docker hub
$ docker pull mongo:latest
# run the mongo image, name is as local-mongo
$ docker run --name local-mongo -d mongo:latest
# check if the container is up and running
$ docker ps
# get the ip address of the mongo docker container
$ docker inspect local-mongo | grep -i ipaddress
```

Making a connection to a MongoDB server using PyMongo :

```python
from pymongo import MongoClient

ip = '172.17.0.2'
port = 27017
client = MongoClient(ip, port)
```

A MongoDB server can host multiple databases. The **client.list_database_names()** method returns a list of the names of all databases on the connected server.A quick way to check if the MongoDB server has authentication enabled is calling the `client.list_database_names()` method. A `pymongo.errors.OperationFailure` exception is raised if authentication is enabled on the server.

```python
client.list_database_names()
['admin', 'config', 'local']
```

By default there are 3 databases on the server : `admin`, `config` and `local`.

### *Function to check open MongoDB server :*

```python
def check_mongo_server(ip, port):
    client = MongoClient(ip, port, serverSelectionTimeoutMS=1000)
    try:
        databases = client.list_database_names()
        print(f'Open Mongo Server: {ip} --> {databases}')
        return True
    except OperationFailure:
        print(f'Auth required: {ip}')
        return False
```

In MongoDB, databases hold collection of documents. A **collection** is equivalent to a table in relational database.

![MongoDB Collection](/img/MongoDB-python/mongodb_collection.png)

The method `client[database_name].list_collection_names` can be used to list the collections.

```python
client['local'].list_collection_names()
['startup_log']

client['admin'].list_collection_names()
['system.version']

client['config'].list_collection_names()
['system.sessions']
```

### *Function to retrieve collections from a MongoDB database :*

```python
def get_collections(ip, port, database):
    client = MongoClient(ip, port, serverSelectionTimeoutMS=10000)
    collections = client[database].list_collection_names()
    return collections
```

A **document** is a record of data. MongoDB stores data records as BSON. BSON is a binary representation of JSON documents. MongoDB documents are composed of field-and-value pairs as shown below.

![MongoDB Document](/img/MongoDB-python/mongodb_document.png)

The `client[database_name][collection_name].find_one()` method can be used to retrieve a single document from a collection.

```python
client['admin']['system.version'].find_one()
{'_id': 'featureCompatibilityVersion', 'version': '4.2'}
```

### *Function to retrieve a single document from a collection from a database :*

```python
def get_single_document(ip, port, database, collection):
    client = MongoClient(ip, port, serverSelectionTimeoutMS=10000, connect=False)
    return client[database][collection].find_one()
```

The `client[database_name][collection_name].find()` method returns an iterable which can be used to retrieve all documents from a MongoDB database collection.

```python
for document in client['admin']['system.version'].find():
    print(document)

{'_id': 'featureCompatibilityVersion', 'version': '4.2'}
```

## Putting it all together

I have glued together a python script that takes a masscan output in XML format as input and outputs a csv file containing all the open MongoDB servers. You can find it at the below link. It is still a work in progress.

* Github : <a href="https://github.com/zer0ttl/mongo-parser" target="_blank" rel="noopener noreferrer">mongo-parser</a>

## How do I avoid this?

* Enable authentication on MongoDB server. MongoDB has a fantastic security <a href="https://docs.MongoDB.com/manual/administration/security-checklist/" target="_blank" rel="noopener noreferrer">cheatsheat</a> for avoiding such mishaps.

## Other tools to look at :

* <a href="https://github.com/woj-ciech/LeakLooker" target="_blank" rel="noopener noreferrer">`https://github.com/woj-ciech/LeakLooker`</a>

