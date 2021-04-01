# PostgreSQL + REST API + Application(s)

- Setup a tutorial PostgreSQL database with docker:

```bash
sudo docker run --name pgdb -p 5433:5432 -e POSTGRES_PASSWORD=p -d postgres
```

## Install [PostgREST]()

- Download `tar.xz` file from: https://github.com/PostgREST/postgrest/releases/latest

```bash
tar xJf postgrest-<version>-<platform>.tar.xz
```

Linux:

```bash
# un-compress into working directory
tar xJf bin/postgrest-v7.0.1-linux-x64-static.tar.xz

# run
./postgrest

Missing: FILENAME

Usage: postgrest FILENAME
  PostgREST 7.0.1 (UNKNOWN) / create a REST API to an existing Postgres database

Available options:
  -h,--help                Show this help text
  FILENAME                 Path to configuration file

Example Config File:
  db-uri = "postgres://user:pass@localhost:5432/dbname"
  db-schema = "public" # this schema gets added to the search_path of every request
  db-anon-role = "postgres"
  db-pool = 10
  db-pool-timeout = 10

  server-host = "!4"
  server-port = 3000

  ## unix socket location
  ## if specified it takes precedence over server-port
  # server-unix-socket = "/tmp/pgrst.sock"
  ## unix socket file mode
  ## when none is provided, 660 is applied by default
  # server-unix-socket-mode = "660"

  ## base url for swagger output
  # openapi-server-proxy-uri = ""

  ## choose a secret, JSON Web Key (or set) to enable JWT auth
  ## (use "@filename" to load from separate file)
  # jwt-secret = "secret_with_at_least_32_characters"
  # secret-is-base64 = false
  # jwt-aud = "your_audience_claim"

  ## limit rows in response
  # max-rows = 1000

  ## stored proc to exec immediately after auth
  # pre-request = "stored_proc_name"

  ## jspath to the role claim key
  # role-claim-key = ".role"

  ## extra schemas to add to the search_path of every request
  # db-extra-search-path = "extensions, util"

  ## stored proc that overrides the root "/" spec
  ## it must be inside the db-schema
  # root-spec = "stored_proc_name"

  ## content types to produce raw output
  # raw-media-types="image/png, image/jpg"
```

## Requirements:

- libpq
- PostgrSQL C Library

## Create a Database for API

```
# execute psql within docker
sudo docker exec -it pgdb psql -U postgres

# create api schema
postgres=# create schema api;

# create table for 'orders' endpoint:
postgres=# create table api.orders (
postgres(# id serial primary key,
postgres(# order_type text not null default 'new',
postgres(# order_status text not null default 'pending',
postgres(# price real not null default 0,
postgres(# datetime timestamptz not null default now() 
postgres(# );

# add some values
insert into api.orders (order_type) values ('new'), ('swap');

# create nologin role
create role web_anon nologin;
grant usage on schema api to web_anon;
grant select on api.orders to web_anon;
```

The `web_anon` role has permission to access things in the `api` schema, and to read rows in the `orders` table.

It’s a good practice to create a dedicated role for connecting to the database, instead of using the highly privileged `postgres` role. 

Name the role `authenticator` and also grant him the ability to switch to the `web_anon` role :

```
create role authenticator noinherit login password 'p';
grant web_anon to authenticator;
```

Now quit out of psql; it’s time to start the API!

```
\q
```

## Run PostgREST

PostgREST uses a configuration file to tell it how to connect to the database. Create a file `tutorial.conf` with this inside:

```
db-uri = "postgres://authenticator:mysecretpassword@localhost:5433/postgres"
db-schema = "api"
db-anon-role = "web_anon"
```

The configuration file has other [options](https://postgrest.org/en/v7.0.0/configuration.html#configuration), but this is all we need. Now run the server:

```
./postgrest tutorial.conf
```

You should see

```
Listening on port 3000
Attempting to connect to the database...
Connection successful
```

It’s now ready to serve web requests. There are many nice graphical API exploration tools you can use, but for this tutorial we’ll use `curl` because it’s likely to be installed on your system already. Open a new terminal (leaving the one open that PostgREST is running inside). Try doing an HTTP request for the todos.

```
curl http://localhost:3000/orders
```

The API replies:

```
[
  {
    "id": 1,
    "order_type": "new",
    "order_status": "pending",
    "price": 0,
    "datetime": null
  },
  {
    "id": 2,
    "order_type": "swap",
    "order_status": "pending",
    "price": 0,
    "datetime": null
  }
]

```

