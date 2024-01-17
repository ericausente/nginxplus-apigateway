## 1. Script for API Key Generation and Store in Keyval (generate_api_key.sh):

Generate a key and use curl to add it to the NGINX Keyval store

```
#!/bin/bash

API_KEY=$(openssl rand -base64 18)
curl -iX DELETE http://localhost/api/7/http/keyvals/apikey
curl -iX POST -d "{\"$API_KEY\":\"allowed\"}" http://localhost/api/7/http/keyvals/apikey

```

## 2. NGINX Keyval Store Configuration:

```
### To mimic a backend

upstream warehouse_inventory {
  server 127.0.0.1:8001;
}

upstream warehouse_pricing {
  server 127.0.0.1:8002;
}

server {
   listen 8001;
   location /api/warehouse/inventory {
   return 200 "inventory";
   }
}

server {
   listen 8002;
   location /api/warehouse/pricing {
   return 200 "pricing";
   }
}


### Create a folder under /etc/nginx/
### Use the keyval directive to check the 'apikey' HTTP header against the keyval zone.
### Then a corresponding value indicating if it's allowed (e.g., allowed).

keyval_zone zone=apikey:64k state=/etc/nginx/state-files/apikey.json;
keyval $http_apikey $is_valid zone=apikey;

server {
    access_log /var/log/nginx/api_access.log main; # Each API may also log to a
                                                   # separate file

    #listen 443 ssl;
    listen 80;
    server_name api.example.com;
    # TLS config
    #ssl_certificate      /etc/ssl/certs/api.example.com.crt;
    #ssl_certificate_key  /etc/ssl/private/api.example.com.key;
    #ssl_session_cache    shared:SSL:10m;
    #ssl_session_timeout  5m;
    #ssl_ciphers          HIGH:!aNULL:!MD5;
    #ssl_protocols        TLSv1.2 TLSv1.3;

    # Warehouse API
    #
    location /api/warehouse/ {
    # Policy configuration here (authentication, rate limiting, logging...)
    #
    access_log /var/log/nginx/warehouse_api.log main;
    auth_request /_validate_apikey;

    # URI routing
    #
    location /api/warehouse/inventory {
        proxy_pass http://warehouse_inventory;
    }

    location /api/warehouse/pricing {
        proxy_pass http://warehouse_pricing;
    }

    return 404; # Catch-all
}

        location = /_validate_apikey {
        internal;

        if ($is_valid != "allowed") {
        return 401; # Unauthorized if API key is not allowed
    }
        return 204; # OK (no content)
    }

    location /api/ {
        api write=on;
        allow 127.0.0.1;
        deny all;
    }

    # Error responses
    error_page 404 = @400;         # Invalid paths are treated as bad requests
    proxy_intercept_errors on;     # Do not send backend errors to the client
    include api_json_errors.conf;  # API client friendly JSON error responses (save this under /etc/nginx/
    default_type application/json; # If no content-type then assume JSON
}
```

## 3. You'll need to add the key  to the key-value store using an HTTP request (the script will take care of this for you).

```
$ bash generate_api_key.sh
HTTP/1.1 204 No Content
Server: nginx
Date: Wed, 17 Jan 2024 01:35:28 GMT
Connection: keep-alive

HTTP/1.1 201 Created
Server: nginx
Date: Wed, 17 Jan 2024 01:35:29 GMT
Content-Length: 0
Location: http://localhost/api/7/http/keyvals/apikey/
Connection: keep-alive
```

## 4. Validate the keyval contents:
```
$curl http://localhost/api/7/http/keyvals/apikey/
{"Ie5LkZ5JYxe8epfnPAfg2/wD":"allowed"}
```

## 5. Now validate the functionality:

XXXX
No API Key:
XXXX

```
$ curl http://api.example.com/api/warehouse/pricing
{"status":401,"message":"Unauthorized"}
```

XXXX
With apikey header but wrong value.
```
$ curl http://api.example.com/api/warehouse/pricing -H "apikey: q28398q2u332q989203"
{"status":401,"message":"Unauthorized"}
```

XXXX
Wrong header but with correct apikey value
XXXX
```
$ curl http://api.example.com/api/warehouse/pricing -H "apike: Ie5LkZ5JYxe8epfnPAfg2/wD"
{"status":401,"message":"Unauthorized"}
```

/ / /
Correct header and value (desired)
/ / / 
```
$ curl http://api.example.com/api/warehouse/pricing -H "apikey: Ie5LkZ5JYxe8epfnPAfg2/wD"
pricing
```
