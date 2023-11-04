# nginxplus-apigateway
<!-- ABOUT THE PROJECT -->
## About The Project

API gateways can be used for both monolithic and microservices-based apps. API gateways perform several key functions:

* Authenticating the requesters making API calls
* Routing requests to appropriate backends
* Applying rate limits to prevent overloading of your systems
* Handling errors and exceptions
* Terminating SSL traffic

**This is the base config:** apigw.conf
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
We will implement rate limiting based on source-ip address, together with error handling

**This is the config with rate limiting and error handling:** apigw-ratelimiting.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

limit_req_zone $binary_remote_addr zone=ratelimit:20m rate=1r/s;
limit_req_status 429;

server {
   listen 80;
   server_name apigw.mikelabs.online;
   location / {
        limit_req zone=ratelimit;
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        proxy_pass          http://backend-api;
        error_page 429 /error429.json;
    }
    location = /error429.json {
        internal;
        return 429 '{"status": "error", "message": "429 Too many requests !"}';
    }
}
```

**This is the config with authentication via apikey:** apigw-authentication.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

limit_req_zone $binary_remote_addr zone=ratelimit:20m rate=1r/s;
limit_req_status 429;

server {
   listen 80;
   server_name apigw.mikelabs.online;
   location / {
        limit_req zone=ratelimit;
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        auth_request /_validate_apikey;
        proxy_pass          http://backend-api;
        error_page 429 /error429.json;
        error_page 401 /error401.json;
        error_page 403 /error403.json;
    }
    location = /error429.json {
        internal;
        return 429 '{"status": "error", "message": "429 Too many requests !"}';
    }
    location = /error401.json {
        internal;
        return 401 '{"status": "error", "message": "401 customized Authorization Required !"}';
    }
    location = /error403.json {
        internal;
        return 403 '{"status": "error", "message": "403 Customized Forbidden !"}';
    }
}
```
