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

### Create a firewall to accept incoming trafic from subscriber node on port 5432
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

## Configure Postgres on publisher node

### Install PostgreSQL 17 on Publisher instance

Do not forget to ssh into publisher instance before you proceed
```
sudo apt install curl ca-certificates vim
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt -y install postgresql
``` 

### Start PostgreSQL
```
systemctl status postgresql@17-main.service
sudo systemctl start postgresql@17-main.service
```

### Server settings
```
sudo vim $PGDATA/postgresql.conf
	listen_addresses = 'IP of publisher' #What IP address(es) to listen on
	wal_level = logical                  #Set wal level to logical to logical replical replication
```

### Allow connection from subscriber instance
the below conf will allow any user from any host in the 172.20.0.0/20 subnet (subscriber subnet) to connect to any database
on this PostgreSQL installation using password of type scram-sha-256
```
sudo vim $PGDATA/pg_hba.conf 
	# TYPE		DATABASE		USER		ADDRESS			        METHOD
	  host		all			    all		    '172.20.0.0/20'		    scram-sha-256
```

### Restart the server
```
sudo systemctl restart postgresql@17-main.service
```

### Create pub_db database 
```
sudo su - postgres
psql
CREATE DATABASE pub_db;
\c pub_db
```

### Create and populate the following tables  in pub_db database 
### orders, pizza_types, pizzas, order_details
```
CREATE TABLE orders (
	order_id INTEGER PRIMARY KEY,
	date DATE NOT NULL,
	time TIME NOT NULL

);
INSERT INTO orders SELECT order_id, date, time
FROM (
SELECT generate_series as order_id, 
CAST(Now() AS DATE) + Cast(Floor(2000 * (Random() - 0.5)) AS INTEGER) as date, 
CAST(Now() AS TIME) + (random() - 0.5) * 24 * '1 hours'::interval as time
FROM Generate_series(1,1000)
);

CREATE TABLE pizza_types (
	pizza_type VARCHAR(40) PRIMARY KEY,
	name VARCHAR(50) NOT NULL,
	category VARCHAR(30) NOT NULL,
	ingredients TEXT NOT NULL
);
INSERT INTO pizza_types (pizza_type, name, category, ingredients) VALUES
('bbq_ckn', 'The Barbecue Chicken Pizza', 'Chicken', 'Barbecued Chicken, Red Peppers, Green Peppers, Tomatoes, Red Onions, Barbecue Sauce'),
('cali_ckn', 'The California Chicken Pizza', 'Chicken', 'Chicken, Artichoke, Spinach, Garlic, Jalapeno Peppers, Fontina Cheese, Gouda Cheese'),
('ckn_alfredo', 'The Chicken Alfredo Pizza', 'Chicken', 'Chicken, Red Onions, Red Peppers, Mushrooms, Asiago Cheese, Alfredo Sauce'),
('ckn_pesto', 'The Chicken Pesto Pizza', 'Chicken', 'Chicken, Tomatoes, Red Peppers, Spinach, Garlic, Pesto Sauce'),
('southw_ckn', 'The Southwest Chicken Pizza', 'Chicken', 'Chicken, Tomatoes, Red Peppers, Red Onions, Jalapeno Peppers, Corn, Cilantro, Chipotle Sauce'),
('thai_ckn', 'The Thai Chicken Pizza', 'Chicken', 'Chicken, Pineapple, Tomatoes, Red Peppers, Thai Sweet Chilli Sauce'),
('big_meat', 'The Big Meat Pizza', 'Classic', 'Bacon, Pepperoni, Italian Sausage, Chorizo Sausage'),
('classic_dlx', 'The Classic Deluxe Pizza', 'Classic', 'Pepperoni, Mushrooms, Red Onions, Red Peppers, Bacon'),
('hawaiian', 'The Hawaiian Pizza', 'Classic', 'Sliced Ham, Pineapple, Mozzarella Cheese'),
('ital_cpcllo', 'The Italian Capocollo Pizza', 'Classic', 'Capocollo, Red Peppers, Tomatoes, Goat Cheese, Garlic, Oregano');

CREATE TABLE pizzas (
	pizza_id VARCHAR(50) PRIMARY KEY,
	pizza_type VARCHAR(40) REFERENCES pizza_types,
	size CHAR(1) NOT NULL,
	price DECIMAL(10, 2) NOT NULL
);
INSERT INTO pizzas (pizza_id, pizza_type, size, price) VALUES
('bbq_ckn_s', 'bbq_ckn', 'S', 12.75),
('bbq_ckn_m', 'bbq_ckn', 'M', 16.75),
('bbq_ckn_l', 'bbq_ckn', 'L', 20.75),
('cali_ckn_s', 'cali_ckn', 'S', 12.75),
('cali_ckn_m', 'cali_ckn', 'M', 16.75),
('cali_ckn_l', 'cali_ckn', 'L', 20.75),
('ckn_alfredo_s', 'ckn_alfredo', 'S', 12.75),
('ckn_alfredo_m', 'ckn_alfredo', 'M', 16.75),
('ckn_alfredo_l', 'ckn_alfredo', 'L', 20.75),
('ckn_pesto_s', 'ckn_pesto', 'S', 12.75),
('ckn_pesto_m', 'ckn_pesto', 'M', 16.75),
('ckn_pesto_l', 'ckn_pesto', 'L', 20.75),
('southw_ckn_s', 'southw_ckn', 'S', 12.75),
('southw_ckn_m', 'southw_ckn', 'M', 16.75),
('southw_ckn_l', 'southw_ckn', 'L', 20.75),
('thai_ckn_s', 'thai_ckn', 'S', 12.75),
('thai_ckn_m', 'thai_ckn', 'M', 16.75),
('thai_ckn_l', 'thai_ckn', 'L', 20.75),
('big_meat_s', 'big_meat', 'S', 12),
('big_meat_m', 'big_meat', 'M', 16),
('big_meat_l', 'big_meat', 'L', 20.5),
('classic_dlx_s', 'classic_dlx', 'S', 12),
('classic_dlx_m', 'classic_dlx', 'M', 16),
('classic_dlx_l', 'classic_dlx', 'L', 20.5);

CREATE TABLE order_details (
	order_details_id INTEGER PRIMARY KEY,
	order_id INTEGER REFERENCES orders,
	pizza_id VARCHaR(50) REFERENCES pizzas,
	quantity NUMERIC NOT NULL
);
INSERT INTO order_details SELECT order_details_id, order_id, pizza_id, quantity
FROM (
SELECT generate_series as order_details_id, 
Floor(1000 * Random() + 1) as order_id, 
(Array['bbq_ckn_s','bbq_ckn_m','bbq_ckn_l','cali_ckn_s','cali_ckn_m','cali_ckn_l','ckn_alfredo_s','ckn_alfredo_m',
'ckn_alfredo_l','ckn_pesto_s','ckn_pesto_m','ckn_pesto_l','southw_ckn_s','southw_ckn_m','southw_ckn_l','thai_ckn_s',
'thai_ckn_m','thai_ckn_l','big_meat_s','big_meat_m','big_meat_l','classic_dlx_s','classic_dlx_m','classic_dlx_l'])[Floor(24 * random() + 1)] as pizza_id,
Floor(5 * Random() + 1) as quantity
FROM Generate_series(1,1000)
);
```
### Create a replication user 
```
CREATE ROLE repluser WITH REPLICATION LOGIN PASSWORD 'secretPassword';
GRANT ALL PRIVILEGES ON DATABASE pub_db TO repluser;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO repluser;
```

### Setting up publication
```
CREATE PUBLICATION mypub;
ALTER PUBLICATION mypub ADD TABLE orders, order_details, pizzas, pizza_types;
```

## Configure Postgres on Subscriber node

### Install PostgreSQL 17 on Subscriber instance

Do not forget to ssh into subscriber instance before you proceed
```
sudo apt install curl ca-certificates vim
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt -y install postgresql
```

### Start PostgreSQL
```
systemctl status postgresql@17-main.service
sudo systemctl start postgresql@17-main.service
```

### Create sub_db database 
```
sudo su - postgres
psql
CREATE DATABASE pub_db;
\c sub_db
```

### To allow replication to work we need to create table schema into subscription db
```
CREATE TABLE pizza_types (pizza_type VARCHAR(40) PRIMARY KEY, name VARCHAR(50) NOT NULL, category VARCHAR(30) NOT NULL, ingredients TEXT NOT NULL);
CREATE TABLE pizzas (pizza_id VARCHAR(50) PRIMARY KEY, pizza_type VARCHAR(40) REFERENCES pizza_types, size CHAR(1) NOT NULL, price DECIMAL(10, 2) NOT NULL);
CREATE TABLE orders (order_id INTEGER PRIMARY KEY, date DATE NOT NULL, time TIME NOT NULL);
CREATE TABLE order_details (order_details_id INTEGER PRIMARY KEY, order_id INTEGER REFERENCES orders, pizza_id VARCHaR(50) REFERENCES pizzas, quantity NUMERIC NOT NULL);
```

### The created tables are empty
```
SELECT * FROM orders;
SELECT * FROM order_details;
SELECT * FROM pizza_types;
SELECT * FROM pizzas;
```

### Create subscription 
```
psql
\c sub_db
CREATE SUBSCRIPTION mysub CONNECTION 'host=172.16.0.2 dbname=pub_db port=5432 password=secretPassword user=repluser' PUBLICATION mypub;
```
As soon as the postmaster process identify the new subscription, it attach it the to the replication and ant launcher and replication workers copy initial data into the sub_db
see screen shot below

## Test and monitor the replication

### Testing replication
- Insert operation on pizza_types table
>
> insert data from the publisher instance 
> ```
> SELECT * FROM pizza_types  ;
> INSERT INTO pizza_types (pizza_type, name, category, ingredients) VALUES
> ('five_cheese', 'The Five Cheese Pizza', 'Veggie', 'ozzarella Cheese, Provolone Cheese, Smoked Gouda Cheese, Romano Cheese, Blue Cheese, Garlic'),
> ('four_cheese', 'The Four Cheese Pizza', 'Veggie', 'Ricotta Cheese, Gorgonzola Piccante Cheese, Mozzarella Cheese, Parmigiano Reggiano Cheese, Garlic'),
> ('green_garden', 'The Green Garden Pizza', 'Veggie', 'Spinach, Mushrooms, Tomatoes, Green Olives, Feta Cheese');
> ```
> check the new inserted data on the subscriber instance 
> ```
> SELECT * FROM pizza_types  ;
> ```

Update operation on pizzas table
update data from the publisher instance
```
UPDATE pizzas SET price = 18 WHERE pizza_id = 'classic_dlx_l';
```
check the new updated data on the subscriber instance 
```
SELECT * FROM pizzas ;
```

delete operation on order_details table
delete data from the publisher instance 
```
SELECT COUNT(*) FROM order_details ;
DELETE FROM order_details WHERE order_details_id > 900;
```
check the deleted data on the subscriber instance 
```
SELECT * FROM order_details ;
```

truncate operation on order_details table
truncate order_details from the publisher instance 
```
SELECT COUNT(*) FROM order_details ;
TRUNCATE TABLE order_details;
```
check order_details is truncated on the subscriber instance 
```
SELECT * FROM order_details ;
```

## Delete all created resources