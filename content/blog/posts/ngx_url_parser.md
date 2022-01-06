+++
title = "Ngx_url_parser"
date = 2016-12-04T12:15:45+01:00
images = ["https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy9uZ3hfdXJsX3BhcnNlcjJfMTNlN2I3M2M1MS5wbmc/photo_ngxUrlParser_big.jpg"]
tags = ["c", "nginx"]
categories = ["projects"]
draft = false
type = "posts"
+++
A non RFC strict URL parser that is derived from NGINX url parser. Main reason to create such parser was need to have parser that is not so strict like uriparser and  is fast.

Example usage:

```cpp
const char * str = "https://user:password@mkaciuba.pl:555/path/?query#fragment";
// structure in with result will be stored
ngx_http_url url;

// run parser
int status = ngx_url_parser(&url, str);
if (status != NGX_URL_OK) {
    printf("Error processing url!\n");
    return 1;
}

printf("Url = %s\n", str);
printf("\nParse status %d", status);
printf("\n scheme = %s", url.scheme);
printf("\n Host = %s", url.host);
printf("\n Port = %s", url.port);
printf("\n Path = %s", url.path);
printf("\n Query = %s", url.query);
printf("\n Fragment = %s", url.fragment);
printf("\n Auth = %s", url.auth);
printf("\n");

// free memory
ngx_url_free(&url);
```

It can parse path only too
```cpp
const char * str = "/path/?query#fragment";
// structure in with result will be stored
ngx_http_url url;

// run parser
int status = ngx_url_parser(&url, str);
if (status != NGX_URL_OK) {
    printf("Error processing url!\n");
    return 1;
}

printf("Url = %s\n", str);
printf("\nParse status %d", status);

printf("\n Path = %s", url.path);
printf("\n Query = %s", url.query);
printf("\n Fragment = %s", url.fragment);
printf("\n");

// free memory
ngx_url_free(&url);
```

# **Performance comparison**

Both parser were used in mode where their mark where is searched element of URI.

Code used for tests:
```cpp
#include <stdio.h>
#include "ngx_url_parser.h"

int main(int argc, char *argv[])
{
    const char * str = "https://user:password@mkaciuba.pl:555/path/?query#fragment";
    for (int i = 0; i < 2000000; i++) {
        ngx_http_url_meta url_meta;
        ngx_url_parser_meta(&url_meta, str);
    }
    return 0;
}
```

uriparser:
```cpp
#include <stdio.h>
#include <Uri.h>

int main(int argc, char *argv[])
{

   const char * str = "https://user:password@mkaciuba.pl:555/path/?query#fragment";

   for (int i = 0; i < 2000000; i++) {
           UriUriA m_uri;
           UriParserStateA m_state;
           m_state.uri = &m_uri;
           uriParseUriA(&m_state, str) ;
                   uriFreeUriMembersA(&m_uri);
    }
    return 0
}
```

Average ngx_url_parser 0.25

Average uriparser 1.08

Source code: [https://github.com/aldor007/ngx_url_parser](https://github.com/aldor007/ngx_url_parser)
