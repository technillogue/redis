
# Redis

Deploying Redis using Fly.

[Read this guide on Fly.](https://fly.io/docs/app-guides/redis/)

<!---- cut here --->

## Rationale

A key/value database like Redis is useful to have to support caching, session management and, well, anything which needs simple fast storage. That's why there's Redis already built into Fly. But there are some things that Redis configuration doesn't do, like Publish and Subscribe, and that's when you want to deploy your own Redis on Fly.

For this example, we are going to customize a Redis docker image to tune it for running on Fly and deploy it with persistent disk storage for Redis to save its data on.

## Preparing

There's a couple of components to this example. We're going to use the official Redis image, `redis:alpine`, but we want to change some system settings before Redis starts running. To do that, we'll use a script, `start-redis-server.sh`

```shell
#!/bin/sh
sysctl vm.overcommit_memory=1
sysctl net.core.somaxconn=1024
redis-server --requirepass $REDIS_PASSWORD  --dir /data/ --appendonly yes
```

The two `sysctl` calls set up the environment so that Redis doesn't throw warnings about memory and connections. The script then starts up the Redis server, giving it a password to require and a directory to persist into using . Now we need to make those changes apply to a new Redis deployment. For that we use this `Dockerfile`:

```dockerfile
FROM redis:alpine

ADD start-redis-server.sh /usr/bin/
RUN chmod +x /usr/bin/start-redis-server.sh

CMD ["start-redis-server.sh"]
```

It adds our new shell script to the image, makes it executable, and makes the image start by running that shell script.

With these two files in place we're ready to put Redis onto Fly.

## Configuring

First, we need a configuration file - and a slot on Fly - for our new application. We'll use the `fly launch` command. The parameter is the app name we want - names have to be unique so choose a new unique one or omit the name in the command line and let Fly choose a name for you.

```cmd
fly launch --name a-unique-app-name
```

By default, this Redis installation will only accept connections on the private IPv6 network. If you want internet access, uncomment the `[[services]]` section in `fly.toml`. 

## Keeping a secret

Our script takes a password from an environment variable to secure Redis. We need to set that now using the `fly secrets` command. This encrypts the value so it can't leak out; the only time it is decoded is into the Fly application as an environment variable.

```
flyctl secrets set REDIS_PASSWORD=<a password>
```

And of course, remember that password because you won't be able to get it back.

## Persisting Redis

The last step is to create a disk volume for Redis to save its state on. Then the Redis can be restarted without losing data. For Fly apps, the volume needs to be in the same region as the app. We saw that region when we initialized the app; here it's `ord`. We'll give the volume the name `redis_server`. 

```cmd
flyctl volumes create redis_server --region ord
```
```out
      Name: redis_server
    Region: ord
   Size GB: 10
Created at: 02 Nov 20 19:55 UTC
```

To connect this volume to the app, `fly.toml` includes a `[[mounts]]` entry.

```
[[mounts]]
source      = "redis_server"
destination = "/data"
```

When the app starts, that volume will be mounted on /data. 

## Deploy

We're ready to deploy now. Run `fly deploy` and the Redis app will be created and launched on the cloud. Once complete you can connect to it using the `redis-cli` command or any other redis client from another application on the internal network.

## Notes

* This configuration sets Redis with mostly default settings. 

## Discuss

* You can discuss this example on the [community.fly.io](https://community.fly.io/t/new-redis-example/366) topic.

