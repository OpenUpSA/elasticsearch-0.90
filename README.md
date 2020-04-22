elasticsearch 0.90.13 for ancient apps
===================================


Deployment on dokku
-------------------

We follow the [Dokku image deployment using tags](http://dokku.viewdocs.io/dokku/deployment/methods/images/#deploying-from-a-docker-registry) approach.

```
dokku apps:create elasticsearch-0.90
```

DISABLE PROXYING PUBLIC REQUESTS: Elasticsearch must **NOT** be exposed to the internet without authentication/authorization in front.

```
dokku proxy:disable elasticsearch-0.90
```

```
docker pull openup/elasticsearch-0.90:latest
```

```
docker tag openup/elasticsearch-0.90:latest dokku/elasticsearch-0.90:latest
```

Configure elasticsearch to use enough RAM - configure this according
to the server and your needs, and leave some room for the operating system. E.g. on a
machine with 4GB RAM, try minimum 1g, maximum 3G.

```
dokku config:set elasticsearch-0.90 ES_MIN_MEM=1g ES_MAX_MEM=3g
```

Map out the data directory to the host to persist across container instances

```
dokku docker-options:add elasticsearch-0.90 deploy,run -v /var/elasticsearch-0.90/data:/elasticsearch/data
```

Now you can depoy the app

```
dokku tags:deploy elasticsearch-0.90 latest
```

If you run an app on the same server that needs to use elasticsearch, you can
set up a consistent hostname for elasticsearch by linking the containers.
**NOTE** this means your app won't start unless elasticsearch is running.

```
dokku docker-options:set myapp deploy,run --link elasticsearch.web.1:elasticsearch
```

Assuming your app takes an environment variable ELASTICSEARCH_URL

```
dokku config:set myapp ELASTICSEARCH_URL=elasticsearch:9200
```

In a single-node cluster, replicas will never be assigned, so status will always be yellow.

To disable replicas and get status green, set number of replicas to zero:

```
docker exec -ti elasticsearch-0.90.web.1 bash
root@812adf00a96e:/# curl -XPUT localhost:9200/_settings -d '{ "index": { "number_of_replicas" : 0 } }'
```

Troubleshooting
---------------

### Unallocated shards

elasticsearch 0.90 seems to fail to allocate shards wen it restarts. Perhaps because of corruption. This is especially going to be an issue whe we run it as a single-node cluster.

The quickest fix is just to rebuild your index, which will replace and allocate all chards when the index is recreated.

You can also force those shards to be allocated to the one node, but their data will be lost. If you can build the index without destroying and recreating it, this is the approach with the least disruption.

Check the node name - (a string like `fjuZTiNVQQSWX4Olf7rmBg1`) and which shards are unassigned (an integer betwee 0 and 5)

```
curl -XGET 'http://localhost:9200/_cluster/state?pretty=true'
```

Assign each unassigned shard manually, replacing the node ID and shard ID in the following command

```
curl -XPOST 'localhost:9200/_cluster/reroute' -d '{"commands": [{"allocate": {"index": "pombola","shard": 1,"node": "fjuZTiNVQQSWX4Olf7rmBg1","allow_primary": true}}]}'
```
