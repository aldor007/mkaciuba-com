+++
title = "Caching Pages With Login Form"
date = 2018-09-09T11:59:58+01:00
images = []
tags = ["devops", "lua", "nginx]
categories = ["projets"]
draft = true
type = "post"
+++

Usually, when websites are made that are to be available to a specific group of recipients (with a login and password), the proxying server does not allow it to be cached. Caching such a page could lead to users who should not have access to the content to have it

In this post I will show that it is possible to prepare such a proxy server configuration, to cache the page per user (fortunately in this case we have only 2 groups: logged in, not logged in. With more group this approach would not make sense.)

NGINX was used as a proxy server.


#  Caching in NGINX

At the beginning information on how caching works in NGINX. It has a _proxy_cache_key_ directive, which by default is composed of _$scheme$proxy_host$ request_ur_i;

So the key to the cache is the host and URL protocol. For example, the site https://mkaciuba.pl/uri will be saved using the httpsmkaciuba.pluri key. This approach does not give much opportunity. But thanks to the fact that in NGINX you can use the LUA language, the problem of static configuration disappears, variables and almost the entire configuration can be modified dynamically.

# NGINX and LUA

Usually, the NGINX configuration is contained in a static configuration file, however, thanks to the [nginx-Lua ](https://github.com/openresty/lua-nginx-module)module, we can extend the support service by asking scripts operating on requests. LUA in NGINX can run on several phases.

![](https://lh3.googleusercontent.com/-vmsGcKFZHNIkalvTSXwUARj2vRj8KL61OzS_stcsJAirs0IQMz1gD4IMpe2NgQ_aJgiUE9C2IL2RYimC1dNGDXRmFFnrkQM92dh2dL514hUY9RFTOFUB_Np1z-ucvr824cE-4x0)

We will be interested in the rewrite by lua phase.

The use of LUA in NGINX gives a lot of possibilities to handle queries. Thanks to it, we no longer need a static approach (everything defined in the configuration) but we can perform HTTP queries, connect to redis, etc.

# Modification of the cache key

We already know how NGINX generates a cache key and that you can use LUA to modify NGINX variables. It's time to put it all together and prepare a configuration that will modify the cache key depending on whether the user should have access or not.

A few initial assumptions:

1.  The logged in user will have a cookie in which we will store the data needed to verify whether he has access to the site
2.  Nginx will communicate with CacheAPI to calculate the cache key. The key will be modified if the user has access to the site

The diagram below shows the proposed control flow for the backend application.

![](https://lh5.googleusercontent.com/16yKh8fGouXtcI7qVqQqUV9_6_pvi8Jb2NF7TDwx5q_UjXtjRNvztnTV6R9-jAXJH3P78erAKeg6OirueeaC_Y9_j1Z1JXjRY_sard56kUIIBWZZ2AZeX0tKBa0fPINcKnWAx5_X)

If you serve a page from the cache

![](https://lh3.googleusercontent.com/t3r0TAv-J1oOoUMFRRAQL07YpW4jNPjPs7sdXeoyXvmSnDcjA758r-K2KgS01kB-YYonrLK7ICT1g-Wagay6hHsxhYEbcTeseY4c-PhieTg3MOLq9lLHLW-eU2QycV2oIgqwU290)

And when we do not have a page in the Nginx

![](https://lh3.googleusercontent.com/rrEhFHqTLYASeqAlueaWOUQ8S_gx8AY6PC-juLYuBjTCc7RVaNFXusJdVh2nIfP6cBj6COOHI0xjsrd4QgWuHI6cKLO3srRV-fHitSDih7JwRAqwkqsxTgEEnn8dlz_wZnhOIVlz)

# NGINX configuration

At the beginning, we should define several variables in nginx so that we can not cave out answers in case of an error with CacheAPI

<pre class="brush:php;">set $ no_cache 0;
  set $ bypass_cache 0;

$ The cache_version variable in it will be saved by the cach version
set $ cache_version "";

# Use of variables to change cacha and not caching the opdis
proxy_cache_bypass $ bypass_cache;
proxy_no_cache $ no_cache;

# Our main secret for handling requests is too large to be included in the nginx configuration. More conveniently edit it in a separate file
rewrite_by_lua_file /var/lib/nginx/lua/rewrite.lua;

# The name of our bucket
proxy_cache dynamic;

# Our cache key
set $ cache_key "$ method! $ host! $ uri! $ is_args $ args! $ content_encoding! $ cache_version";
proxy_cache_key $ cache_key;</pre>

The script itself looks as follows

<pre class="brush:python;">ngx.var.no_cache = '0'
local user_id = ngx.var.cookie_session_cookie
if user_id == nil then
   return
end

local path = ngx.var.uri
-- dziala na okreslonych sciezkac
if not re.match(path, '^/(path1|path)/(.*)', 'ijo') then
   return
end

local http = require "resty.http"
local httpc = http.new()

-- timeouts do api
httpc:set_timeouts(50, 450, 450)
httpc:connect('unix:/var/run/nginx.sock')

local res, err = httpc:request({
   path = '/api/v1/lambda/?userid=' .. user_id .. '&path=' .. path,
   headers = {
       ["Host"] = "mkaciuba.com",
       ['x-cache-api-user-id'] = user_id,
       ['x-cache-api-path'] = path
   },
})

if not res then
-- in case of error no cache response
   ngx.log(ngx.ERR, 'request to lambda failed ' .. err )
   ngx.var.no_cache = '1'
   return
end

if res.status ~= 200 then
   ngx.log(ngx.ERR, 'request to lambda invalid status ' .. res.status)
   ngx.var.no_cache = '1'
   return
end

local info = res.headers['x-cache-api-info']
if info == 'no-cache' then
   ngx.var.no_cache = '1'
   return
end

if res.headers['x-cache-api-cache-param'] == nil then
   return
end

if info == 'no-change-cache-key' then
   return
end

-- update cache key
local cache_key = ngx.var.cache_key
ngx.var.cache_key = cache_key .. '!' .. res.headers['x-cache-api-cache-param']</pre>

# Summary

Caching pages per user segment is not a simple task, but once you've done them, you can speed up the page's performance with a slow backend.

PS. Example application with described here logic can be found [here](https://github.com/aldor007/nginx-user-cache-poc)