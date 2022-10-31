# Istio TCP Egress Gateway

## Introduction

This repository demonstrates how use the Istio Egress Gateway for TCP and TCP/TLS traffic to external services.  This demonstration uses an external RabbitMQ server as the message bus for the [`ecommerce-microservices`](https://github.com/brijimerson/ecommerce-microservices)  application.  Both non-secure TCP and secure TCP egress scenarios are demonstrated.

This demonstration assumes that you have a Kubernetes cluster in AWS and are going to install RabbitMQ as a standalone EC2 instance in the same VPC.  Other environments should be similar to set up.

## RabbitMQ Setup

### EC2 Setup

First, create an EC2 instance based on Ubuntu 20.04.  A t2.xlarge instance type with a 60GB volume was used for this demonstration.  Also, make sure you assign a public IP address to the instance so that you can access the RabbitMQ Management Console.  If you don't, you'll have forward ports for the console.  The security group for the instance should have the following inbound rules:

 * 5671/TCP from at least the subnet for your Kubernetes cluster (secure RabbitMQ protocol)
 * 5672/TCP from at least the subnet for your Kubernetes cluster (non-secure RabbitMQ protocol)
 * 15672/TCP from everything (HTTP port for the Management Console)
 * 22/TCP from everything (SSH port)

### RabbitMQ Installation

Once the EC2 instance is available, establish an SSH session to the instance.  Install RabbitMQ with the Management Console enabled with the following commands:

```
sudo apt-get update
sudo apt install rabbitmq-server
sudo rabbitmq-plugins enable rabbitmq_management
```

You should see the RabbitMQ server and Management Console listening on their respective ports:

```
sudo apt install net-tools # for netstat

sudo netstat -anp | grep 5672

<omitted>

sudo netstat -anp | grep 15672

<omitted>
```

By default the guest user can only access the Management Console from `localhost`.  Add an admin user that can access the Management Console from anywhere:

```
sudo rabbitmqctl add_user admin <password>
sudo rabbitmqctl set_user_tags admin administrator
```

You should be able to access the Management Console at `http://<ec2-instance-public-address>:15672` with the admin user created.

### TLS Setup

You can create a self-signed certificate for TLS encryption to use in this demonstration.  First, create the `ca-certificate`, `server-certificate`, and `server-key` keys.  First set up the directories for the certificates:

```
mkdir testca
cd testca
mkdir certs private
chmod 700 private
echo 01 > serial
touch index.txt
```
Then create a file called `openssl.cnf` with the following contents:

```
[ ca ]
default_ca = testca

[ testca ]
dir = .
certificate = $dir/ca_certificate.pem
database = $dir/index.txt
new_certs_dir = $dir/certs
private_key = $dir/private/ca_private_key.pem
serial = $dir/serial

default_crl_days = 7
default_days = 365
default_md = sha256

policy = testca_policy
x509_extensions = certificate_extensions

[ testca_policy ]
commonName = supplied
stateOrProvinceName = optional
countryName = optional
emailAddress = optional
organizationName = optional
organizationalUnitName = optional
domainComponent = optional

[ certificate_extensions ]
basicConstraints = CA:false

[ req ]
default_bits = 2048
default_keyfile = ./private/ca_private_key.pem
default_md = sha256
prompt = yes
distinguished_name = root_ca_distinguished_name
x509_extensions = root_ca_extensions

[ root_ca_distinguished_name ]
commonName = hostname

[ root_ca_extensions ]
basicConstraints = CA:true
keyUsage = keyCertSign, cRLSign

[ client_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.2

[ server_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.1
```

Then generate the root ca and server certificates:

```
openssl req -x509 -config openssl.cnf -newkey rsa:2048 -days 365 \
    -out ca_certificate.pem -outform PEM -subj /CN=MyTestCA/ -nodes
openssl x509 -in ca_certificate.pem -out ca_certificate.cer -outform DER

mkdir server
cd server
openssl genrsa -out private_key.pem 2048
openssl req -new -key private_key.pem -out req.pem -outform PEM \
    -subj /CN=$(hostname)/O=server/ -nodes
cd ..
openssl ca -config openssl.cnf -in server/req.pem -out server/server_certificate.pem -notext -batch -extensions server_ca_extensions
cd server
openssl pkcs12 -export -out server_certificate.p12 -in server_certificate.pem -inkey private_key.pem \
    -passout pass:password
```

Copy the needed certificates and keys to a RabbitMQ directory, and set the proper owner and permissions:

```
sudo mkdir /etc/rabbitmq/certs

sudo cp ~ubuntu/testca/ca_certificate.pem /etc/rabbitmq/certs/ca_certificate.pem
sudo cp ~ubuntu/testca/server/server_certificate.pem /etc/rabbitmq/certs/server_certificate.pem
sudo cp ~ubuntu/testca/server/private_key.pem /etc/rabbitmq/certs/server_key.pem

sudo chmod 400 /etc/rabbitmq/certs/*
sudo chown rabbitmq:rabbitmq /etc/rabbitmq/certs/*
```

Finally, add a `rabbitmq.conf` file to configure TLS:

```
sudo vim /etc/rabbitmq/rabbitmq.conf
```

and add these configuration entries:

```
# SSL options
listeners.ssl.default = 5671
ssl_options.cacertfile = /etc/rabbitmq/certs/ca_certificate.pem
ssl_options.certfile = /etc/rabbitmq/certs/server_certificate.pem
ssl_options.keyfile = /etc/rabbitmq/certs/server_key.pem
ssl_options.verify  = verify_peer
ssl_options.fail_if_no_peer_cert = false
```

If you don't want to allow any non-secure connections to RabbitMQ, add this line to the `rabbitmq.conf` file as well:

```
# No non-ssl connections
listeners.tcp = none
```

Restart RabbitMQ to enable the TLS settings:

```
sudo service rabbitmq-server restart
```

To test that RabbitMQ is configured and listening for both secure and non-secure (if non-secure isn't disabled above) connections, use the `busybox` pod:

```
kubectl apply -f busybox.yaml
kubectl exec -it -n no-injection busybox -- sh

/ # telnet ip-172-20-58-9.us-east-2.compute.internal 5672 # Use the private IP address of the RabbitMQ instance
Connected to ip-172-20-58-9.us-east-2.compute.internal
/ # telnet ip-172-20-58-9.us-east-2.compute.internal 5671 # Use the private IP address of the RabbitMQ instance
Connected to ip-172-20-58-9.us-east-2.compute.internal
```
## Istio Installation

Install Istio as you normally would.  Make sure that you install the ingress and egress gateways too.  From the Istio release directory:

```
export AM_VALUES=<path-to-this-repository>/istio-values.yaml # Edit this file as needed

kubectl create ns istio-system
helm install istio-base manifests/charts/base --namespace istio-system
helm install istiod manifests/charts/istio-control/istio-discovery/ -n istio-system --values $AM_VALUES
helm install istio-ingress manifests/charts/gateways/istio-ingress/ -n istio-system --values $AM_VALUES
helm install istio-egress manifests/charts/gateways/istio-egress/ -n istio-system --values $AM_VALUES
```

## Application Installation

There is a separate folder called [`ecommerce-microservices`](https://github.com/istio/se-workshop-demo-library/ecommerce-microservices) that contains a demonstration of microservices on Istio.  One of the microservices, `payment-service`, requires an egress gateway to make an API call to Stripe, a credit card processing API. Two of the microservices, `payment-service` and `payment-history` communicate through the RabbitMQ server as well.

First, install all of the services without any egress setup.  For more details on configuration see the [`README`](https://github.com/brijimerson/ecommerce-microservices/README.md) in the `ecommerce-microservices` repository.

```
cd ecommerce-microservices/deploy
./install-all.sh
./deploy-all.sh
```

You also need to patch the `payment-history` and `payment-service` deployments to override the default RabbitMQ connection settings.  Edit `patch-payment-service.yaml`and `patch-payment-history.yaml `, update the host, username, port, and SSL settings, and then apply the patches:

```
kubectl patch deployment payment-service -n ecommerce --patch "$(cat patch-payment-service.yaml)"
kubectl patch deployment payment-history -n ecommerce --patch "$(cat patch-payment-history.yaml)"
```
 
If you view the logs in the `payment-history` pod, you should see an error when it tries to establish a connection to the RabbitMQ server.

If you're using a non-secure connection you should see something like this:

```
2021-07-15 15:40:07.177  INFO 1 --- [    container-3] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [ip-172-20-58-9.us-east-2.compute.internal:5672]
2021-07-15 15:40:07.181 ERROR 1 --- [    container-3] o.s.a.r.l.SimpleMessageListenerContainer : Failed to check/redeclare auto-delete queue(s).
```

And if you're using a secure connection you should see something like this:

```
2021-07-15 15:32:17.108  INFO 1 --- [   container-24] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [ip-172-20-58-9.us-east-2.compute.internal:5671]
2021-07-15 15:32:17.110 ERROR 1 --- [   container-24] c.r.client.impl.SocketFrameHandler       : TLS connection failed: Remote host terminated the handshake
```

## Configure Egress

The final step is to configure the egress gateway for RabbitMQ and the Stripe API.  

If you didn't use the istio-values.yaml file for the Istio installation, you will need patch the `istio-egressgateway` to add ports `5671` and `5672` to the listener (the istio-values.yaml in this repository has these ports too):

```
kubectl patch svc istio-egressgateway -n istio-system --patch "$(cat patch-egress-gateway.yaml)"
```

Then, depending on whether you're using a secure or non-secure connection to RabbitMQ, apply the `ServiceEntry`, `Gateway`, `DestinationRule`, and `VirtualService` objects for the Istio Egress Gateway:

_Secure_

```
kubectl apply -f rabbitmq-egress-gateway-tls.yaml
```
_Non-secure_

```
kubectl apply -f rabbitmq-egress-gateway-notls.yaml
```

If you view the logs in the `payment-history` pod, you should see a successful connection established to RabbitMQ:

```
2021-07-15 15:55:45.899  INFO 1 --- [    container-5] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [ip-172-20-58-9.us-east-2.compute.internal:5671]
2021-07-15 15:55:46.047  INFO 1 --- [    container-5] o.s.a.r.c.CachingConnectionFactory       : Created new connection: rabbitConnectionFactory#1be59f28:9/SimpleConnection@46c27a8f [delegate=amqp://admin@172.20.58.9:5671/, localPort= 42908]
``` 

Finally, apply the egress gateway for the Stripe API:

```
kubectl apply -f stripe-egress-gateway-tls.yaml
```

To verify that the Stripe API traffic is going through the egress gateway, submit a couple of orders, and then look at the access logs for the egress gateway:

```
kubectl logs -l istio=egressgateway -n istio-system
```

You should see entries for https calls to api.stripe.com, which means that all external calls are being routed through the egress gateway.

