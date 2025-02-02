# Building from Compose file

## Specification

Bake uses the [compose-spec](https://docs.docker.com/compose/compose-file/) to
parse a compose file and translate each service to a [target](file-definition.md#target).

```yaml
# docker-compose.yml
services:
  webapp-dev: 
    build: &build-dev
      dockerfile: Dockerfile.webapp
      tags:
        - docker.io/username/webapp:latest
      cache_from:
        - docker.io/username/webapp:cache
      cache_to:
        - docker.io/username/webapp:cache

  webapp-release:
    build:
      <<: *build-dev
      x-bake:
        platforms:
          - linux/amd64
          - linux/arm64

  db:
    image: docker.io/username/db
    build:
      dockerfile: Dockerfile.db
```

```console
$ docker buildx bake --print
```
```json
{
  "group": {
    "default": {
      "targets": [
        "db",
        "webapp-dev",
        "webapp-release"
      ]
    }
  },
  "target": {
    "db": {
      "context": ".",
      "dockerfile": "Dockerfile.db",
      "tags": [
        "docker.io/username/db"
      ]
    },
    "webapp-dev": {
      "context": ".",
      "dockerfile": "Dockerfile.webapp",
      "tags": [
        "docker.io/username/webapp:latest"
      ],
      "cache-from": [
        "docker.io/username/webapp:cache"
      ],
      "cache-to": [
        "docker.io/username/webapp:cache"
      ]
    },
    "webapp-release": {
      "context": ".",
      "dockerfile": "Dockerfile.webapp",
      "tags": [
        "docker.io/username/webapp:latest"
      ],
      "cache-from": [
        "docker.io/username/webapp:cache"
      ],
      "cache-to": [
        "docker.io/username/webapp:cache"
      ],
      "platforms": [
        "linux/amd64",
        "linux/arm64"
      ]
    }
  }
}
```

Unlike the [HCL format](file-definition.md#hcl-definition), there are some
limitations with the compose format:

* Specifying variables or global scope attributes is not yet supported
* `inherits` service field is not supported, but you can use [YAML anchors](https://docs.docker.com/compose/compose-file/#fragments) to reference other services like the example above

## Extension field with `x-bake`

Even if some fields are not (yet) available in the compose specification, you
can use the [special extension](https://docs.docker.com/compose/compose-file/#extension)
field `x-bake` in your compose file to evaluate extra fields:

```yaml
# docker-compose.yml
services:
  addon:
    image: ct-addon:bar
    build:
      context: .
      dockerfile: ./Dockerfile
      args:
        CT_ECR: foo
        CT_TAG: bar
      x-bake:
        tags:
          - ct-addon:foo
          - ct-addon:alp
        platforms:
          - linux/amd64
          - linux/arm64
        cache-from:
          - user/app:cache
          - type=local,src=path/to/cache
        cache-to:
          - type=local,dest=path/to/cache
        pull: true

  aws:
    image: ct-fake-aws:bar
    build:
      dockerfile: ./aws.Dockerfile
      args:
        CT_ECR: foo
        CT_TAG: bar
      x-bake:
        secret:
          - id=mysecret,src=./secret
          - id=mysecret2,src=./secret2
        platforms: linux/arm64
        output: type=docker
        no-cache: true
```

```console
$ docker buildx bake --print
```
```json
{
  "group": {
    "default": {
      "targets": [
        "aws",
        "addon"
      ]
    }
  },
  "target": {
    "addon": {
      "context": ".",
      "dockerfile": "./Dockerfile",
      "args": {
        "CT_ECR": "foo",
        "CT_TAG": "bar"
      },
      "tags": [
        "ct-addon:foo",
        "ct-addon:alp"
      ],
      "cache-from": [
        "user/app:cache",
        "type=local,src=path/to/cache"
      ],
      "cache-to": [
        "type=local,dest=path/to/cache"
      ],
      "platforms": [
        "linux/amd64",
        "linux/arm64"
      ],
      "pull": true
    },
    "aws": {
      "context": ".",
      "dockerfile": "./aws.Dockerfile",
      "args": {
        "CT_ECR": "foo",
        "CT_TAG": "bar"
      },
      "tags": [
        "ct-fake-aws:bar"
      ],
      "secret": [
        "id=mysecret,src=./secret",
        "id=mysecret2,src=./secret2"
      ],
      "platforms": [
        "linux/arm64"
      ],
      "output": [
        "type=docker"
      ],
      "no-cache": true
    }
  }
}
```

Complete list of valid fields for `x-bake`:

* `cache-from`
* `cache-to`
* `contexts`
* `no-cache`
* `no-cache-filter`
* `output`
* `platforms`
* `pull`
* `secret`
* `ssh`
* `tags`
