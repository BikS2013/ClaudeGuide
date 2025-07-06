I want to start a new solution here.
The solution must have a backend project and a frontend project.
Both of them must be nodejs, typescript implementations.
The front end must be based on React and Vite.

I want you to start creating the complete configuration layer including:
- the config.json endpoint at client.
- the configuration repository, and database cache patterns as refered in instructions.

**Very Important** The use of any kind of default values is strictly prohibited .

The assets owner name must be "github-monitor" and the assets owner class must be "monitoring-app".
You sould use the following .env variables to support the configuration implementation 

``` 
NODE_ENV=
PORT=
HOST=

DATABASE_URL=

GITHUB_REPO=
GITHUB_TOKEN=
GITHUB_BRANCH=

ASSET_OWNER_NAME=
ASSET_OWNER_CLASS=

ENV_SETTINGS_ASSET_KEY=

CORS_ORIGIN=
CORS_CREDENTIALS=true
CORS_MAX_AGE=86400

```

I don't want you to implement any kind of external caching (like REDIS or similars).
In cases of assets retrieved from the registry, the preffered approach is the application to keep the resources in local memory variables 
after the resource has beed retrieved to avoid to retrieve it again and again.
It is also recomended to implement a dedicated request to refresh from the resiistry an asset when needed.
This asset-refresh endpoint must be generic and get as parameter the assey_key.
