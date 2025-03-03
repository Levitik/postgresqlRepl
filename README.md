# Logical data replication with PostgreSQL

## Configure publisher and subscriber nodes

### Create a private network
```
gcloud compute networks create replnet --subnet-mode=custom
```

### Create two subnets, one for publisher instance and the other for subscriber instance
```
gcloud compute networks subnets create pub-subnet --network=replnet --region="europe-west9" --range=172.16.0.0/24
gcloud compute networks subnets create sub-subnet --network=replnet --region="europe-west1" --range=172.20.0.0/20
```

### Create firewall rule to allow ssh traffic
```
gcloud compute firewall-rules create repl-allow-ssh \
	--direction=INGRESS \
	--priority=1000 \
	--network=replnet \
	--action=ALLOW \
	--rules=tcp:22 \
	--source-ranges=0.0.0.0/0
```

### Create a firewall to accept incoming trafic from subscriber nodeon port 5432
```
gcloud compute firewall-rules create repl-allow-sub \
	--direction=INGRESS \
	--priority=1000 \
	--network=replnet \
	--action=ALLOW \
	--rules=tcp:5432 \
	--source-ranges=172.20.0.0/20
```

### Create Subscriber node
```
gcloud compute instances create subscription-vm \
    --zone="europe-west1-c" \
    --machine-type=e2-medium \
    --subnet=sub-subnet \
    --create-disk=auto-delete=yes,boot=yes,device-name=sub-disk,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20241219,mode=rw,size=10,type=pd-balanced \
    --no-address
```

### Create Publisher node
```
gcloud compute instances create publication-vm \
    --zone="europe-west9-b" \
    --machine-type=e2-medium \
    --subnet=pub-subnet \
    --create-disk=auto-delete=yes,boot=yes,device-name=pub-disk,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20241219,mode=rw,size=10,type=pd-balanced \
    --no-address
```

### Create a router for subscriber (need by NAT)
```
gcloud compute routers create sub-nat-router \
    --network replnet \
    --region europe-west1
```

### Create a router for publisher (need by NAT)
```
gcloud compute routers create pub-nat-router \
    --network replnet \
    --region europe-west9
```

### Create a Nat for subscriber
```
gcloud compute routers nats create sub-nat-config \
    --router-region europe-west1 \
    --router sub-nat-router \
    --nat-all-subnet-ip-ranges \
     --auto-allocate-nat-external-ips
```

### Create a Nat for subscriber
```
gcloud compute routers nats create pub-nat-config \
    --router-region europe-west9 \
    --router pub-nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

## Configure Postgres on publisher and subscriber nodes

## Test and monitor the replication

## Delete all created resources