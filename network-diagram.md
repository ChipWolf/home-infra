# Network diagram

## https/443 traffic ingressing to the network from internet

### [haproxy](http://www.haproxy.org/)

* all 443 traffic is port-forwarded (from edgerouter) the HAProxy VIP (10.0.7.30)
* the HAProxy VIP is shared between the raspberry pi devices (`lb` & `pihole` running haproxy via docker)
* `lb` and & `pihole` leverage [`keepalived`](https://www.keepalived.org/) to 'float' the 10.0.7.30 VIP to whichever device is the leader, with a notice sent to slack whenever the leadership changes (e.g. a device is rebooted or haproxy is stopped on one device)
* keepalived configuration example:

`/etc/keepalived/keepalived.conf`:

```conf
vrrp_script chk_haproxy {      # Requires keepalived-1.1.13
  script "/usr/bin/killall -0 haproxy"  # cheaper than pidof
  interval 2 # check every 2 seconds
  weight 2 # add 2 points of priority if OK
}

vrrp_instance VI_1 {
  interface eth0
  state MASTER
  virtual_router_id 151
  priority 101 # 101 on primary, 100 on secondary
  virtual_ipaddress {
    10.0.7.30
  }
  track_script {
    chk_haproxy
  }
  notify /etc/keepalived/notify.sh
}
```

`/etc/keepalived/notify.sh`:

```shell
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3
HOST=$(hostname)

curl -X POST --data-urlencode "payload={\"channel\": \"#general\", \"username\": \"$HOST\", \"text\": \":exclamation: keepalived on *$HOST* is now in $STATE state\", \"icon_emoji\": \":skull:\"}" <slack incoming webhook URL>

case $STATE in
        "MASTER") /usr/bin/docker kill -s HUP haproxy
                  ;;
esac
```

* example slack notification via keepalived:

![haproxy notification](https://i.imgur.com/UhPTjeg.png)

* haproxy is running as a docker container on each host via:

```shell
IMAGE=haproxy:alpine
NAME=haproxy

if ! docker pull $IMAGE | tee /dev/stderr | grep -q "Image is up to date"
then
  echo "removing old $NAME for $IMAGE"
  docker stop $NAME
  docker rm -f $NAME
fi

if ! docker ps --filter=name="$NAME" --filter=status="running" | grep $NAME
then
  echo "running $NAME"

docker run \
    -d \
    --name $NAME \
    --restart always \
    --net=host \
    -v /mnt/appdata/haproxy/:/usr/local/etc/haproxy/:ro \
    $IMAGE
fi
```

* haproxy is still used instead of sending all https/443 traffic directly to nginx (running in the k8s cluster) because the SNI header is inspected to route traffic to other workloads that also expect to ingress on port 443 (e.g. stunnel or some other test nginx instance)
* example snippet from the haproxy configuration for 443 ingress:

```haproxy
frontend https_frontend
    bind 10.0.7.30:443
    option tcplog
    mode tcp
    option clitcpka
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }

    use_backend https_stunnel_backend if { req_ssl_sni -i www.mydomain.com }
    use_backend https_stunnel_backend if !{ req_ssl_sni -m found }
    use_backend https_k8s_nginx if { req_ssl_sni -m end .mydomain.com }
    use_backend https_k8s_nginx-test if { req_ssl_sni -m end .test.mydomain.com }
```

### [nginx ingress](https://github.com/kubernetes/ingress-nginx)

* nginx (running as a k8s deployment) has an 'external' IP obtained via [MetalLB](https://metallb.universe.tf/):

```shell
NAME                                          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
nginx-ingress-controller                      LoadBalancer   10.43.22.99     10.2.0.150    80:31582/TCP,443:30321/TCP   3d7h
```

* haproxy forwards all appropriate https/443 to this external IP and nginx routes as required via the various k8s ingress configurations
* nginx runs with multiple replicas and leverages [`cert-manager`](https://github.com/jetstack/cert-manager) to handle certificates from letsencrypt instead of nginx itself requesting certs

## other incoming traffic

* all other traffic ingressing from the internet are port-forwarded, as needed, to the MetalLB-bound IP addresses for internet-exposed k8s services
* examples of this would be things like minecraft, unifi controller, torrent, etc
