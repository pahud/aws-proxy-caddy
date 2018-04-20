# aws-proxy-caddy



**aws-proxy-caddy** is a reference architecture built on top of AWS cross-regional VPC peering to accelerate cross-regional HTTPS traffic and ensure the traffic go through AWS backbone. It has some benefits:

- All traffic go through AWS regional VPC peering with underlying HA and encryption
- Internal facing http proxy service with [Caddy](https://caddyserver.com/) - a very robust web server written with Golang.
- Reduced HTTPS RTT overhead - all HTTPS RTT within the same AWS region. No cross-regional HTTPS traffic.



## Demo

cURL the API Gateway regional endpoint from Singapore region to Oregon region could take up to 720ms:

```
$ curl https://8ucdb5oms4.execute-api.us-west-2.amazonaws.com/prod/greetings -w '\n==> %{time_total} spent'
<h1>Hello, AWS!</h1>
==> 0.720968 spent
```

However, if you cURL with aws-proxy-caddy via the regional VPC-peering, it would be around 331ms, which is  **54%** improvement in speed.

```
$ curl http://10.0.1.23:2015/prod/greetings -w '\n==> %{time_total} spent'
<h1>Hello, AWS!</h1>
==> 0.331145 spent
```



## Sample and Howto

create your Caddyfile like this and store in your current directory in EC2 from Oregon VPC.

```
0.0.0.0
browse
proxy /prod/greetings https://8ucdb5oms4.execute-api.us-west-2.amazonaws.com {
	header_upstream Host 8ucdb5oms4.execute-api.us-west-2.amazonaws.com
	keepalive 20
}
log stdout
errors stdout
```



In the EC2, run the Caddy with docker like this

```
$ docker run -d -v $PWD/Caddyfile:/etc/Caddyfile -p 80:2015 --name caddy abiosoft/caddy
```

This will create a caddy service with docker and bind the TCP port 80.

**<u>Please note</u>** - if your EC2 has public IP interface, you need to <u>**create a security group**</u> to filter the inbound TCP 80 request from the internet or just allow internal traffic to TCP 80, otherwise this host will be vulnerable to become an open proxy.