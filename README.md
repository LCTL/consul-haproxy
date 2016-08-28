# Consul HAproxy
Automatic service discovery with client side load balancing

## Required
1. Consul Server
2. Run gliderlabs/registrator on your docker machine - https://github.com/gliderlabs/registrator

## Example

### Call service via consul-proxy
```
version: "2"
services:
  web-service:
    image: jwilder/whoami
    ports:
      - "8000"
    environment:
      - SERVICE_NAME=whoami
  internal-service-proxy:
    image: lawrence0819/consul-haproxy
    command: -consul consul.service.consul:8500
    ports:
      - "8000"
    environment:
      - CONSUL_SERVICE_NAME_REGEX=.*whoami.*
  call-web-service-via-consul-haproxy:
    image: bmoussaud/ubuntu-wget
    container_name: call-web-service-via-consul-haproxy
    command: bash -c 'for ((i=0;i<5;i++)); do wget -qO- "http://internal-service-proxy:8000"; done'
    links:
      - internal-service-proxy
    depends_on:
      - internal-service-proxy
```

Result:
```
docker logs call-web-service-via-consul-haproxy
I'm cc074dd2618c
I'm cc074dd2618c
I'm cc074dd2618c
I'm cc074dd2618c
I'm cc074dd2618c
```

You can create more web-service instance on different machine, It will automatic discovery by consul-haproxy:
```
# node0
docker run -d -P -e "SERVICE_NAME=whoami" jwilder/whoami
# node1
docker run -d -P -e "SERVICE_NAME=whoami" jwilder/whoami
```

Restart:
```
docker-compose restart
docker logs call-web-service-via-consul-haproxy
I'm cc074dd2618c
I'm d0d827722641
I'm 640449cda65b
I'm cc074dd2618c
I'm d0d827722641
```

### Bind virtual host for web service
```
version: "2"
services:
  web-service:
    image: jwilder/whoami
    ports:
      - "8000"
    environment:
      - SERVICE_NAME=whoami
  internal-services-proxy:
    image: lawrence0819/consul-haproxy
    command: -consul consul.service.consul:8500
    ports:
      - "8000"
    environment:
      - VIRTUAL_HOST=whoami.example.com
      - CONSUL_SERVICE_NAME_REGEX=.*whoami.*
  public-nginx-proxy:
    image: jwilder/nginx-proxy
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
```

You can create more web-service instance on different machine, It will automatic discovery by consul-haproxy:
```
# node0
docker run -d -P -e "SERVICE_NAME=whoami" jwilder/whoami
# node1
docker run -d -P -e "SERVICE_NAME=whoami" jwilder/whoami
```

## Configuration
| Environment Variables | Default Value                                                                              | Description |
| --------------------- |--------------------------------------------------------------------------------------------|-------------|
| HAPROXY_MAXCONN       | `1024`                                                                                     |HAProxy maxconn  |
| HAPROXY_MODE          | `tcp`                                                                                      |HAProxy mode |
| HAPROXY_BALANCE       | `roundrobin`                                                                               |HAProxy balance |
| HAPROXY_OPTIONS       | `timeout connect 3s,timeout server 3s,timeout client 3s,option redispatch,option tcp-check`|`,` as separator|
| CONSUL_SERVICE_NAME_REGEX | `.*`                                                                                   |for filter unnecessary service in proxy| 
