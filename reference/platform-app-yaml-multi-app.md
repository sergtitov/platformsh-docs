# Multiple Applications Configuration

Platform.sh supports  multiple applications per project (for example  RESTful web
 services and a front-end, or a main web site and a blog).

All of the applications share a common configuration through the files present in the
`.platform/` directory at the root of the repository. But on many fronts each can
has its own configuration.

To create a multi-application setup put each application in its own directory with
an `.platform.app.yaml` at its root. When Platform.sh deploys this it will create
a service for each application.

The directory layout may resemble something like the following:
```
.platform/
  routes.yaml
  services.yaml
user_api/
  .platform.app.yaml
  public/
    index.php
    [...]
content_api/
  .platform.app.yaml
  public/
    index.php
    [...]
front_end/
  .platform.app.yaml
  public/
    index.php
    assets/
    [...]
```
> **note** note that there is no relationship between the directories names and the 
> applications' names in Platform.sh. The names are going to get declared in the
> configuration files..


Let's imagine that in this case the Front End (front) app depends on two REST services a 
User API (user) and a Content API (content). Lets also imagine both API applications
connect to the same Postgres database. And that the Front-End  does not connect directly to
the database (it does everything through these APIs) but it does use a Redis as a Cache.

In our use case the User API is accessible through  a url like "http://api.example.com/v1/users"
and the Content API is accessible through "http://api.example.com/v1/content". In this case we are not doing http caching on the two APIs, but we are caching on the Front End applications.

The `.profile/routes.yaml` may look like:
````
"http://api.{default}/v1/users":
    type: upstream
    upstream: "user:php"
    cache:
        enabled: false
"http://api.{default}/v1/content":
    type: upstream
    upstream: "content:php"
    cache:
        enabled: false
"http://{default}/":
    type: upstream
    upstream: "front_end:php"
    cache:
        enabled: true
```

> **note** for the moment the upstream is always of this form, ending with ":php" this is 
> because, currently we only support PHP, and in the future Platform.sh will support multiple 
> endpoints per application. Here it simply tells our router to connect, for example to 
> a service we called "front_end" and route http traffic to the PHP service running there.
> Learn more on : [routes.yaml](/refernce/routes.yaml.md)


The `.profile/services.yaml` may look like:
```
common_db:
    type: postgresql
    disk: 4096
redis_cache:
    type: redis
```
Here we say that we have two services that will be potentially available to any application
in the project the, the keys "common_db" and "redis_cache" are names we can freely choose
to describe the role theses services will have in our project; We will use these names in 
each application's `.platform.app.yaml` to link the service to the application.



The User API `user_api/.platform.app.yaml`  will look like the following (only putting here the relevant parts `[...]` representing the stuff we cut out...):
```
name: user
[...]
relationships:
    "database": "common_db:postgresql"
```

> **note** Here the name "database" is freely chosen by us, this will be exposed in the 
> Environment Variables of the application (so it can be different between the different
> application of the same project). The right part "common_db:postgresql" comes from
> what we put in `services.yaml` for the name. The ":postgresql" suffix, which is required,
> is there because in the future we will support multiple endpoints per service (for 
> services that support multiple protocols). Learn more on : [services.yaml](/refernce/routes.yaml.md)

Or example's Content API `cintent_api/.platform.app.yaml` will in this case be very similar
```
name: content
[...]
relationships:
    "database": "common_db:postgresql"
```

> **note** Here we keep the name "database" but this could be anything; This name will only
> appear in the `PLATFORM_RELATIONSHIPS` environment variable of the "Content Api".

And finally the Front End `front_end/.platform.app.yaml` will look like:
```
name: front
[...]
relationships:
    "database": "redis_cache:redis"
    "user_service": "user:php"
    "content_service": "content:php"
```

These relationships allow an application to connect to another.. and will expose in 
its Environment Variables everything that is needed for it to be used dynamically
in its configuration.

> **note** Here the names "database", "user_service" and "content_service"
> are freely chosen by us, often enough its better to keep to a simpler naming scheme.