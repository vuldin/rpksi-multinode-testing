# rpksi in a multi-node, multi-partition cluster

The following steps show how to configure a local environment for testing `rpksi` and tiered storage.
The environment is a three-node cluster kicked off with a docker compose script, that includes minio for local S3 storage.

## Prerequisites
Install docker, docker-compose, and mc (more details on mc: https://docs.min.io/minio/baremetal/reference/minio-mc.html)

## Setup

Setup the local `volumes` directory. Each container will mount their own sub-directory here:
```shell
mkdir -p volumes/{redpanda0,redpanda1,redpanda2,minio}
sudo chown -R 101:101 volumes
```

Start the containers (1 Minio and 3 Redpanda):
```shell
docker compose up
```

Configure the local S3 bucket:
```shell
mc mb local/redpanda
mc alias set local http://localhost:9000 minio minio123 --api s3v4
```

Create a topic with three partitions, each replicated three times (ensuring each node has one copy of each partition), with remote read/writes enabled:
```shell
rpk topic create atopic -c retention.bytes=30000 -c segment.bytes=10000 -c redpanda.remote.read=true -c redpanda.remote.write=true -p 3 -r 3
```

Verify each Redpanda node is part of the cluster:
```shell
rpk cluster health
rpk cluster info
```

Verify partition and replicas for the topic:
```shell
rpk topic describe -p atopic
```

Watch local segments in the Redpanda nodes in one terminal:
```shell
cd volumes && watch find redpanda* -type f -name "*.log" -path "*atopic*"
```

Watch remote segments in minio another terminal:
```shell
cd volumes && watch find minio -type f -path "*kafka/atopic*"
```

Each segment for this topic is 10K bytes, and local retention is set to 30K bytes. Run the following command to start generating data into the topic:
```shell
BATCH=$(date) ; printf "$BATCH %s\n" {1..1000} | rpk topic produce atopic
```

After running the above command ~12 times you will see the earliest local segments have been deleted and only appear as remote segments.
The exact number of times command needs to be ran depends on a number of variables, including which partition the command produces data to.

# rpksi

> _Note: rpksi is still being worked on, and these instructions will be updated soon_

`rpksi` is a CLI for managing tiered storage. It can be found [here](https://github.com/redpanda-data/rpksi).
The following commands show how to use `rpksi` to find and delete segments:

```shell
rpksi ls -t atopic
rpksi ls -at atopic
rpksi ls -at atopic -o 4500
rpksi del -at atopic -o 4500 --dry-run
rpksi del -at atopic -o 4500
```

# cleanup

```shell
docker compose down
sudo rm -r volumes
```
