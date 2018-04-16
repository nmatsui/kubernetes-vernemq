# kubernetes-vernemq
A kubernetes configuration yaml to construct VerneMQ Cluster.

## Usage
### Register password to Kubernetes Secret
1. create password file
    * create two user as `inner` and `outer` in order to access from Kubernetes and from Internet

    ```bash
    $ mkdir -p secrets
    $ touch secrets/vmq.passwd
    $ docker run --rm -v $(PWD)/secrets:/mnt -it erlio/docker-vernemq vmq-passwd /mnt/vmq.passwd inner
    $ docker run --rm -v $(PWD)/secrets:/mnt -it erlio/docker-vernemq vmq-passwd /mnt/vmq.passwd outer
    ```
1. register password file as Kubernetes Secret

    ```bash
    $ kubectl create secret generic vernemq-passwd --from-file=./secrets/vmq.passwd
    ```

### Register self-signed certificate files to Kubernetes Secret
1. create self-signed CA

    ```bash
    $ openssl req -new -x509 -days 365 -extensions v3_ca -keyout secrets/ca.key -out secrets/ca.crt
    ```
1. create server certificate signed by self-signed CA
    * you have to spacify the FQDN of VerneMQ's MQTTS endpoint

    ```bash
    $ openssl genrsa -out secrets/server.key 2048
    $ openssl req -out secrets/server.csr -key secrets/server.key -new
    $ openssl x509 -req -in secrets/server.csr -CA secrets/ca.crt -CAkey secrets/ca.key -CAcreateserial -out secrets/server.crt -days 365
    ```
1. register certifcation files as Kubernetes Secret

    ```bash
    $ kubectl create secret generic vernemq-certifications --from-file=./secrets/ca.crt --from-file=./secrets/server.crt --from-file=./secrets/server.key
    ```

### start VerneMQ Cluster on Kubernetes
1. start VerneMQ Cluster on Kubernetes like below:
    * start 3 pods using StatefulSet
    * create and join a VerneMQ cluster automatically
    * listen MQTT (1883/tcp) as ClusterIP which can be accessed from Kubernetes only
    * listen MQTT over TLS (8883/tcp) as LoadBalancer which can be accessed from Kubernetes and Internet

    ```bash
    $ kubectl apply -f vernemq-cluster.yaml
    ```
1. confirm cluster status

    ```bash
    $ kubectl exec vernemq-0 -- vmq-admin cluster show
    +---------------------------------------------------+-------+
    |                       Node                        |Running|
    +---------------------------------------------------+-------+
    |VerneMQ@vernemq-0.vernemq.default.svc.cluster.local| true  |
    |VerneMQ@vernemq-2.vernemq.default.svc.cluster.local| true  |
    |VerneMQ@vernemq-1.vernemq.default.svc.cluster.local| true  |
    +---------------------------------------------------+-------+
    ```

1. check MQTTS external IP

    ```bash
    $ kubectl get services -l app=mqtts
    ```

1. register the FQDN of VerneMQ's MQTTS endpoint to DNS Server in order to resolve MQTTS external IP

## Connecting to VerneMQ from 'inner' and 'outer'
1. start 'inner' subscriber
    * 'inner' subscriber subscribe VerneMQ from inside Kubernetes using 1883/tcp (no encrypt)

    ```text
    $ kubectl run inner-sub --rm -it --image efrecon/mqtt-client /bin/ash
    / # mosquitto_sub -h mqtt -p 1883 -t /foo/bar -d -u inner -P <password of 'inner'>
    ```

1. start 'outer' subscriber
    * 'outer' subscriber subscribe VerneMQ through Internet using 8883/tcp (mqtt over tls)

    ```text
    $ mosquitto_sub -h <FQDN of VerneMQ's MQTTS endpoint> -p 8883 --cafile ./secrets/ca.crt -t /foo/bar -d -u outer -P <password of 'outer'>
    ```

1. publish message from 'inner' publisher
    * 'inner' publisher publish message to VerneMQ from inside Kubernetes using 1883/tcp (no encrypt)
    * When 'inner' publisher pulish message, its message push back to 'inner' subscriber and 'outer' suscriber

    ```text
    $ kubectl run inner-pub --rm -it --image efrecon/mqtt-client /bin/ash
    / # mosquitto_pub -h mqtt -p 1883 -t /foo/bar -d -u inner -P <password of 'inner'> -m "Message from inner"
    ```

1. publish message from 'outer' publisher
    * 'outer' publisher publish message to VerneMQ through Internet using 8883/tcp (mqtt over tls)
    * When 'outer' publisher pulish message, its message push back to 'inner' subscriber and 'outer' suscriber

    ```text
    $ mosquitto_pub -h <FQDN of VerneMQ's MQTTS endpoint> -p 8883 --cafile ./secrets/ca.crt -t /foo/bar -d -u outer -P <password of 'outer'> -m "Message from outer"
    ```

## See also

[VerneMQ](https://vernemq.com/)
[erlio/docker-vernemq](https://github.com/erlio/docker-vernemq)

## License

[MIT License](/LICENSE)

## Copyright
Copyright (c) 2018 Nobuyuki Matsui <nobuyuki.matsui@gmail.com>
