elasticsearch 0.90.13 for ancient apps
===================================

Elasticsearch must **NOT** be exposed to the internet without authentication/authorization in front. We tend to do that using firewall rules that allow only access for specific servers that we control. Ensure that port 9200 and 9300 are closed on the server for anyone except the servers we allow. Ensure that you don't proxy open ports to elasticsearch's ports.

Deployment on dokku
-------------------

We follow the [Dokku image deployment using tags](http://dokku.viewdocs.io/dokku/deployment/methods/images/#deploying-from-a-docker-registry) approach.

```
dokku apps:create elasticsearch-0.90
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