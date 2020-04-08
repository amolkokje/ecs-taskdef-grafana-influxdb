# ecs-taskdef-grafana-influxdb
ECS task definition to launch grafana and influxdb containers using the EC2 launch type

## Considerations
- Network mode used is 'bridge' (default). This is because 'awspc' network mode provides an ENI with a private IP
which cannot be assigned a public DNS/IP. Thus, with 'awsvpc' you would need to launch the service with Load Balancer
to allow public endpoint accesss. Besides that, 'awsvpc' does not allow custom host->container port mapping - this
complicates and requires more stuff to be deployed to enable clients in corporate network to reach the endpoint as
seldom corporate networks do not allow outbound traffic on several ports.

- Launch type is 'EC2', and not 'Fargate'. Fargate makes the deployment easier as with that there is no need to manage
the cluster and container instances. But Fargate does not allow mounting volumes for persistent storage. Persistent
storage with something like EBS volume is very useful when working with large amount of data, as it allows to scale
the size of storage without having to scale-up the deployment/instance. Also EBS allows automated backups, recovery
and all the other cool stuff.


## Steps:
- Create a S3 bucket(MyServerConfigBucket) and copy the contents of the folder ServerConfigFiles to the bucket. This
bucket will be used to source all the server config files.
```
aws s3 sync ServerConfigFiles s3://MyServerConfigBucket
```

- Create a cluster, with a single container instance. You may have multiple container instances if you need for other
applications, but for this deployment, a single instance is sufficient. Please note that you will only be able to
deploy a single task, because if you need multiple tasks, you will have to manage the port contention.
Launch the container EC2 instance with the following user data to configure it to pull configuration from S3:
```
#!/bin/bash
echo 'INIT: update server'
yum -y update
sudo yum -y install awscli

echo 'INIT: config to connect instance to cluster'
sudo touch /etc/ecs/ecs.config
sudo chmod 777 /etc/ecs/ecs.config
sudo echo "ECS_CLUSTER=MyCluster" > /etc/ecs/ecs.config

echo 'INIT: get the server config files from S3'
sudo mkdir /data
sudo chmod 777 /data
cd /data
aws s3 sync s3://MyServerConfigBucket .

echo 'INIT: create data drive for InfluxDB data mount'
sudo mkdir /data/influxdbdata

echo 'INIT: update the ECS container agent, and restart'
docker pull amazon/amazon-ecs-agent:latest
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
sudo docker run --name ecs-agent --detach=true --restart=on-failure:10 --volume=/var/run:/var/run --volume=/var/log/ecs/:/log --volume=/var/lib/ecs/data:/data --volume=/etc/ecs:/etc/ecs --net=host --env-file=/etc/ecs/ecs.config amazon/amazon-ecs-agent:latest
```

- Register the task definition using CLI:
```
aws ecs register-task-definition --cli-input-json file://grafana-influxdb-taskdef.json
```

- From the console, deploy a service using the task def. When the service is deployed, you can go in the task and
get the IP address and the port to connect to the components.


## Architecture
- Grafana and InfluxDB containers pulled from public docker registry are deployed on a single container instance.
- Some configuration parameters are passed to the containers through environment variables.
- Configuration files for the containers and grafana dashboards are pulled from a S3 bucket onto the container instance.
These configuration files are deployed on container instance using Bind volumes. InfluxDB data is also mounted to a
bind volume. Following volumes are present in the deployment.
    - grafana-dashboards: Mounted to '/var/lib/grafana/dashboards' on the grafana container. This is the location
    where all the grafana dashboards are stored as files. These files are loaded using the provisioning file on
    grafana-provisioning volume 'provisioning/dashboards/dashboard.yml'
    - grafana-config: Mounted to '/etc/grafana/custom.ini' and is the custom config file for grafana server.
    - grafana-provisioning: Mounted to '/etc/grafana/provisioning' and contains .yml file used for provisioing the
    dashboards
    - influxdb-data: Mounted to '/var/lib/influxdb' and stores all the data that is indexed to influxdb. This drive
    can be mounted to a EBS volume to enable backup and recovery.
