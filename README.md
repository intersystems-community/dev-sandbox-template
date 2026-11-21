# Developer Sandbox Example TODO

This demo application lets users list, add, and delete TODO items in a web UI.
It is a starting point for a project on
[Developer Cloud](https://cloud.sandbox.developer.intersystems.com/) and
demonstrates:

- persistent application data stored in IRIS at `/data`;
- a static browser UI served by the bundled web server;
- a JSON REST API with list, create, and delete operations;
- reading a variable configured in the Developer Cloud UI;
- an IRIS init method that records every successful start.

## Create a Developer Cloud project

Create a project from this Git repository and use these settings:

| Setting | Value |
| --- | --- |
| IRIS init method | `APP:##class(App.Init).run()` |
| Variable name | `APP_MESSAGE` |
| Variable value | Any non-empty value |

After the project starts, use the application link on its **Project** page in
[Developer Cloud](https://cloud.sandbox.developer.intersystems.com/). The link
uses the domain selected when the project was created.

On the first start of a new persistent database, the init method creates three
sample tasks. Deleted sample tasks are not recreated on later starts.

Runtime details, including `APP_MESSAGE`, are available from `/api/status`:

```json
{
  "variable": {
    "name": "APP_MESSAGE",
    "isSet": true,
    "value": "Hello from Developer Cloud"
  },
  "irisInit": {
    "runCount": 1,
    "lastRunAt": "2026-11-09 09:00:00"
  }
}
```

The status endpoint returns the value of `APP_MESSAGE`. Do not use this pattern
for passwords, tokens, or other secrets.

The demo gives public read/write permission to `%DB_APP`. Production
applications should use dedicated IRIS users, roles, and resource permissions.

## How persistence works

[`iris.cpf`](iris.cpf) creates two databases:

- `APP-CODE` contains compiled classes and is part of the Docker image;
- `APP-DATA` contains globals and persistent objects and has
  `Directory=/data`.

Developer Cloud treats `/data` as persistent storage. Rebuilding the project
replaces `APP-CODE` but retains `APP-DATA`, so the `runCount` returned by the
REST endpoint continues increasing across rebuilds.

TODO items are instances of the persistent `App.Data.Todo` class. After each
successful IRIS start, the init method saves a row in
`App.Data.StartupEvent`. ObjectScript reads the configured variable with:

```objectscript
set value = $system.Util.GetEnviron("APP_MESSAGE")
```

## Run locally

Start the application:

```shell
docker compose up --build -d
```

Open [http://localhost:52773/](http://localhost:52773/).

Developer Cloud invokes the init method automatically. To reproduce this
locally, invoke it once after IRIS starts:

```shell
docker compose exec -T iris iris session IRIS -U APP <<'EOF'
set sc = ##class(App.Init).run()
do:$system.Status.IsError(sc) $system.OBJ.DisplayError(sc)
halt
EOF
curl http://localhost:52773/api/status
```

Use the TODO API directly:

```shell
curl http://localhost:52773/api/todos
curl -X POST http://localhost:52773/api/todos \
  -H 'Content-Type: application/json' \
  --data '{"text":"Try the Developer Sandbox example"}'
curl -X DELETE http://localhost:52773/api/todos/1
```

Stop the local container:

```shell
docker compose down
```

The Compose setup is ephemeral. Developer Cloud provides persistent storage at
`/data`; the local Compose file is for functional testing of the image.

## Project layout

```text
Dockerfile                       Builds a self-contained IRIS image
iris.cpf                         Database, namespace, and web app configuration
iris.script                      Imports and compiles ObjectScript classes
src/App/Init.cls                 Developer Cloud post-start method
src/App/REST.cls                 Status and TODO REST endpoints
src/App/Data/StartupEvent.cls    Persistent startup records
src/App/Data/Todo.cls            Persistent TODO items
web/index.html                   Static TODO page
web/app.css                      Shared frontend styles
web/app.js                       TODO frontend behavior
httpd-local.conf                 Root URL mapping to the static entry point
docker-compose.yml               Optional local development environment
```
