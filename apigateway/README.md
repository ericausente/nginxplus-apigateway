etc/
└── nginx/
    ├── api_conf.d/ ………………………………… Subdirectory for per-API configuration
    │   └── warehouse_api.conf …… Definition and policy of the Warehouse API
    ├── api_backends.conf ………………… The backend services (upstreams)
    ├── api_gateway.conf …………………… Top-level configuration for the API gateway server
    ├── api_json_errors.conf ………… HTTP error responses in JSON format
    ├── conf.d/
    │   ├── ...
    │   └── existing_apps.conf
    └── nginx.conf



So you see, from your /etc/nginx/nginx.conf you have the following directives: 

    include /etc/nginx/api_gateway.conf; # All API gateway configuration
    include /etc/nginx/conf.d/*.conf;    # Regular web traffic


This is from the 
## api_gateway.conf 

```
include api_backends.conf;
include api_keys.conf;

limit_req_zone $binary_remote_addr zone=client_ip_10rs:1m rate=1r/s;
limit_req_zone $http_apikey        zone=apikey_200rs:1m   rate=200r/s;

server {
    access_log /var/log/nginx/api_access.log main; # Each API may also log to a 
                                                   # separate file

    listen 443 ssl;
    server_name api.example.com;

    # TLS config
    ssl_certificate      /etc/ssl/certs/api.example.com.crt;
    ssl_certificate_key  /etc/ssl/private/api.example.com.key;
    ssl_session_cache    shared:SSL:10m;
    ssl_session_timeout  5m;
    ssl_ciphers          HIGH:!aNULL:!MD5;
    ssl_protocols        TLSv1.2 TLSv1.3;

    # API definitions, one per file
    include api_conf.d/*.conf;

    # Error responses
    error_page 404 = @400;         # Invalid paths are treated as bad requests
    proxy_intercept_errors on;     # Do not send backend errors to the client
    include api_json_errors.conf;  # API client friendly JSON error responses
    default_type application/json; # If no content-type then assume JSON

    # API key validation
    location = /_validate_apikey {
        internal;

        if ($http_apikey = "") {
            return 401; # Unauthorized
        }
        if ($api_client_name = "") {
            return 403; # Forbidden
        }

        return 204; # OK (no content)
    }

    # Dummy location used to populate $request_body for JSON validation
    location /_get_request_body {
        return 204;
    }
}

js_import json_validation.js;
js_set $json_validated json_validation.parseRequestBody;

server {
    listen 127.0.0.1:10415;
    return 415; # Unsupported media type
    include api_json_errors.conf;
}

# Access to write operations is evaluated by JWT claim 'admin'
map $request_method $admin_permitted_method {
    "GET"     true;
    "HEAD"    true;
    "OPTIONS" true;
    default   $jwt_claim_admin;
}

# vim: syntax=nginx
```


You notice that there is api_json_errors.conf  above included right?: 
## api_json_errors.conf 

```
error_page 400 = @400;
location @400 { return 400 '{"status":400,"message":"Bad request"}\n'; }

error_page 401 = @401;
location @401 { return 401 '{"status":401,"message":"Unauthorized"}\n'; }

error_page 403 = @403;
location @403 { return 403 '{"status":403,"message":"Forbidden"}\n'; }

error_page 404 = @404;
location @404 { return 404 '{"status":404,"message":"Resource not found"}\n'; }

error_page 405 = @405;
location @405 { return 405 '{"status":405,"message":"Method not allowed"}\n'; }

error_page 408 = @408;
location @408 { return 408 '{"status":408,"message":"Request timeout"}\n'; }

error_page 413 = @413;
location @413 { return 413 '{"status":413,"message":"Payload too large"}\n'; }

error_page 414 = @414;
location @414 { return 414 '{"status":414,"message":"Request URI too large"}\n'; }

error_page 415 = @415;
location @415 { return 415 '{"status":415,"message":"Unsupported media type"}\n'; }

error_page 426 = @426;
location @426 { return 426 '{"status":426,"message":"HTTP request was sent to HTTPS port"}\n'; }

error_page 429 = @429;
location @429 { return 429 '{"status":429,"message":"API rate limit exceeded"}\n'; }

error_page 495 = @495;
location @495 { return 495 '{"status":495,"message":"Client certificate authentication error"}\n'; }

error_page 496 = @496;
location @496 { return 496 '{"status":496,"message":"Client certificate not presented"}\n'; }

error_page 497 = @497;
location @497 { return 497 '{"status":497,"message":"HTTP request was sent to mutual TLS port"}\n'; }

error_page 500 = @500;
location @500 { return 500 '{"status":500,"message":"Server error"}\n'; }

error_page 501 = @501;
location @501 { return 501 '{"status":501,"message":"Not implemented"}\n'; }

error_page 502 = @502;
location @502 { return 502 '{"status":502,"message":"Bad gateway"}\n'; }

# vim: syntax=nginx
```



## api_keys.conf 
```
map $http_apikey $api_client_name {
    default "";

    "7B5zIqmRGXmrJTFmKa99vcit" "client_one";
    "QzVV6y1EmQFbbxOfRCwyJs35" "client_two";
    "mGcjH8Fv6U9y3BVF9H3Ypb9T" "client_six";
}

# Infrastructure clients
#
map $api_client_name $is_infrastructure {
    default       0;

    "client_one"  1;
    "client_six"  1;
}

# vim: syntax=nginx
```




## json_validation.js 
```
export default { parseRequestBody };

function parseRequestBody(r) {
    try {
        if (r.variables.request_body) {
            JSON.parse(r.variables.request_body);
        }
        return r.variables.upstream;
    } catch (e) {
        r.error('JSON.parse exception');
        return '127.0.0.1:10415'; // Address for error response
    }
}
```


## warehouse_api.conf 
```
# API definition
#
location /api/warehouse/pricing {
    limit_except GET PATCH {
        deny all;
    }
    error_page 403 = @405; # Convert response from '403 (Forbidden)' 
                           # to '405 (Method Not Allowed)'
    set $upstream pricing_service;
    rewrite ^ /_warehouse last;
}

location /api/warehouse/inventory {
    limit_except GET {
        deny all;
    }
    error_page 403 = @405;
    set $upstream inventory_service;
    rewrite ^ /_warehouse last;
}

# Policy section
#
location = /_warehouse {
    internal;
    set $api_name "Warehouse";

    limit_req zone=client_ip_10rs;
    limit_req_status 429;

    proxy_pass http://$upstream$request_uri;
}

# vim: syntax=nginx
```



## warehouse_api_apikeys.conf  ---> I think if got API Key Authentication!

```
# Warehouse API
#
location /api/warehouse/pricing {
    set $upstream pricing_service;
    rewrite ^ /_warehouse last;
}

location /api/warehouse/inventory {
    set $upstream inventory_service;
    rewrite ^ /_warehouse last;
}

location = /api/warehouse/inventory/audit {
    if ($is_infrastructure = 0) {
        return 403; # Forbidden (not infrastructure)
    }
    set $upstream inventory_service;
    rewrite ^ /_warehouse last;
}

# Policy section
#
location = /_warehouse {
    internal;
    set $api_name "Warehouse";

    if ($http_apikey = "") {
        return 401; # Unauthorized (please authenticate)
    }
    if ($api_client_name = "") {
        return 403; # Forbidden (invalid API key)
    }

    proxy_pass http://$upstream$request_uri;
}

# vim: syntax=nginx
```

## warehouse_api_bodysize.conf 
```
# Warehouse API
#
location /api/warehouse/ {
    # Policy configuration here (authentication, rate limiting, logging...)
    #
    access_log /var/log/nginx/warehouse_api.log main;
    client_max_body_size 16k;

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

# vim: syntax=nginx
```


## warehouse_api_jsonbody.conf 
```
# Warehouse API
#
location /api/warehouse/ {
    # Policy configuration here (authentication, rate limiting, logging...)
    #
    access_log /var/log/nginx/warehouse_api.log main;

    # URI routing
    #
    location /api/warehouse/inventory {
        proxy_pass http://warehouse_inventory;
    }

    location /api/warehouse/pricing {
        set $upstream warehouse_pricing;
        mirror /_get_request_body;        # Force early read
        client_body_in_single_buffer on;  # Minimize memory copy operations 
                                          # on request body
        client_body_buffer_size      16k; # Largest body to keep in memory 
                                          # (before writing to file)
        client_max_body_size         16k;
        proxy_pass http://$json_validated$request_uri;
    }

    return 404; # Catch-all
}

# vim: syntax=nginx
```



## warehouse_api_methods.conf 
```
# Warehouse API
#
location /api/warehouse/ {
    # Policy configuration here (authentication, rate limiting, logging...)
    #
    access_log /var/log/nginx/warehouse_api.log main;
    auth_jwt "Warehouse API";
    auth_jwt_key_file /etc/nginx/idp_jwks.json;

    # URI routing
    #
    location /api/warehouse/inventory {
        limit_except GET {
            deny all; 
        }
        error_page 403 = @405; # Convert deny response from '403 (Forbidden)'
                               # to '405 (Method Not Allowed)'
        proxy_pass http://warehouse_inventory;
    }

    location /api/warehouse/pricing {
        limit_except GET PATCH {
            deny all;
        }
        if ($admin_permitted_method != "true") {
            return 403;
        }
        error_page 403 = @405; # Convert deny response from '403 (Forbidden)'
                               # to '405 (Method Not Allowed)'
        proxy_pass http://warehouse_pricing;
    }

    return 404; # Catch-all
}

# vim: syntax=nginx
```

## warehouse_api_ratelimit.conf 
```
# Warehouse API
#
location /api/warehouse/ {
    # Policy configuration here (authentication, rate limiting, logging...)
    #
    access_log /var/log/nginx/warehouse_api.log main;
    limit_req zone=client_ip_10rs;
    limit_req_status 429;

    # URI routing
    #
    location /api/warehouse/inventory {
        limit_except GET {
            deny all;
        }
        error_page 403 = @405; # Convert deny response from '403 (Forbidden)'
                               # to '405 (Method Not Allowed)'
        proxy_pass http://warehouse_inventory;
    }

    location /api/warehouse/pricing {
        limit_except GET PATCH {
            deny all;
        }
        error_page 403 = @405;
        proxy_pass http://warehouse_pricing;
    }

    return 404; # Catch-all
}

# vim: syntax=nginx
```


## warehouse_api_secured.conf 
```
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

    location = /api/warehouse/inventory/audit {
        if ($is_infrastructure = 0) {
            return 403; # Forbidden (not infrastructure)
        }
        proxy_pass http://warehouse_inventory;
    }

    location /api/warehouse/pricing {
        proxy_pass http://warehouse_pricing;
    }

    return 404; # Catch-all
}

# vim: syntax=nginx
```
