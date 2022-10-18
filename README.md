# Content delivery network

Run ```docker-compose up -d``` to launch the environment. 

## Geo-balancing

In addition to DNS, balancers and nodes containers I added two test containers to emulate traffic from different locations. 

IP of the ```app-us``` container is added to the ```l11/bind/bind/etc/GeoIP.acl``` MaxMind library to the ```acl US {}``` section making it be served by the ```balancer-1```. 

Requests from the ```app-all``` container is serverd by ```balancer-2``` as defined in the bind9 views: 

```shell
view "united-states" {
    match-clients { US; };
    recursion yes;
    zone "cdn.example.com" {
        type master;
        file "/etc/bind/zones/us.cdn.example.com";
    };
};

view "all" {
    match-clients {"any"; };
    recursion yes;
    zone "cdn.example.com" {
        type master;
        file "/etc/bind/zones/all.cdn.example.com";
    };
};
```

## Balancing approaches

I used the ```for i in `seq 1 20`; do curl cdn.example.com; echo; done``` command to check the balancer behavior. 

### ip_hash

```ini
upstream node {
    ip_hash;
    server node-1:8000;
    server node-2:8000;
}
```

Output: 

Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2

### least_conn

```ini
upstream node {
    least_conn;
    server node-1:8000;
    server node-2:8000;
}
```

Output:

Node 1\
Node 1\
Node 1\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1\
Node 2\
Node 1

### leat_conn with weight 

```ini
upstream node {
    least_conn;
    server node-1:8000 weight=3;
    server node-2:8000;
}
```
Output:

Node 1\
Node 2\
Node 1\
Node 1\
Node 1\
Node 2\
Node 1\
Node 1\
Node 1\
Node 2\
Node 1\
Node 1\
Node 1\
Node 2\
Node 1\
Node 1\
Node 1\
Node 2\
Node 1\
Node 1

### hash

```ini
upstream node {
    hash $scheme$request_uri;
    server node-1:8000 weight=3;
    server node-2:8000;
}
```

Output: 

Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\
Node 2\


## Cache

Implemented in the balancer node config ```config/balancer/balancer-1/default.conf``` using the proxy_cache directives

```ini
proxy_cache_path /var/cache/nginx/static levels=1:2 keys_zone=p11:32m inactive=1d max_size=1g;
```

```ini
server {
    listen       80;
    listen  [::]:80;
    server_name balancer-1;
    root /var/www;
    index index.html index.htm;

    error_log /var/log/nginx/localhost.error.log;
    access_log /var/log/nginx/localhost.access.log;

    proxy_cache p11;
    proxy_cache_valid 200 302 1h;
    proxy_cache_valid 404     1m;

    location / {
        proxy_pass http://node;
    }
}
```