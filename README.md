# nginxplus-apigateway
<!-- ABOUT THE PROJECT -->
## About The Project

API gateways can be used for both monolithic and microservices-based apps. API gateways perform several key functions:

* Authenticating the requesters making API calls
* Routing requests to appropriate backends
* Applying rate limits to prevent overloading of your systems
* Handling errors and exceptions
* Terminating SSL traffic

<p align="right">(<a href="#readme-top">back to top</a>)</p>


apigw.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

server {
   listen 80;
   server_name apigw.mikelabs.online;
   location / {
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        proxy_pass          http://backend-api;
    }

}
```
