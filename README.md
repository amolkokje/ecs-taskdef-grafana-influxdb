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
- Create a cluster, with a single container instance. You may have multiple container instances if you need for other
applications, but for this deployment, a single instance is sufficient. Please note that you will only be able to
deploy a single task, because if you need multiple tasks, you will have to manage the port contention.

- Register the task definition using CLI:
```
aws ecs register-task-definition --cli-input-json file://grafana-influxdb-taskdef.json
```

- From the console, deploy a service using the task def. When the service is deployed, you can go in the task and
get the IP address and the port to connect to the components.


## Architecture

