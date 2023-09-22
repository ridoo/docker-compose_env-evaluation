# Evaluation of variables within docker compose


Pre-defined containers are meant to run in different environments (test, stage, prod, or your CI/CI pipeline) to provide configuration [which is strictly separated](https://12factor.net/config) from both the code base and between environments.
Docker merges configuration [by respecting a preceding context](https://docs.docker.com/compose/environment-variables/envvars-precedence/) which might get confusing sometimes.

This repository tries to make transparent how variable evaluation works in docker compose.
We show the normal case, but also demonstrate the effect when multiple configuration files are used.

Next, we focus on the effect using `env_files` and the resulting `environment` of a container to be built.

## Single level (level 0)

Single level is straight.
These two files are important:

```sh
.
├── docker-compose.yaml
└── level_0.env
```

File `level_0.env` is referenced in the [`docker-comopse.yaml` as `env_file`](./docker-compose.yaml#L6).
It looks like:

```sh
ENV_LEVEL="level_0"
REPLACEMENT="env level: ${ENV_LEVEL}"
```

Running `docker-compose config` give us the following environment:

```sh
environment:
  ENV_LEVEL: level_0
  REPLACEMENT: 'env level: level_0'
```

Replacement `"env level: ${ENV_LEVEL}"` have been done [within `level_0.env` file](https://github.com/ridoo/docker-compose_6741/blob/main/level_0.env).

Now, rename the `_.env` to `.env`.
Contents look similar:

```sh
ENV_LEVEL="host env"
REPLACEMENT="env level: ${ENV_LEVEL}"
```

> :bulb: **Note:**
>
> You can set environment variables via `.env`-file.
> If present, the `.env` is read always by Docker before any `docker-compose.yaml`.
> However, the variables are available via build time.
> They do not become part of the container environment (pass it as `env_file` if you really want to).

Now run `docker-compose config`.
The environment looks like:

```sh
environment:
  ENV_LEVEL: level_0
  REPLACEMENT: 'env level: host env'
```

You can see, that `ENV_LEVEL="host env"` from the `.env` takes precedence.

Play around with other possibilities to get a feeling about which environment variable weighs more.
For example, what do you think, how `REPLACEMENT` will look like, when executing the following:

```sh
export ENV_LEVEL=shell ; docker compose config
```


## Multiple levels

Docker compose offers to chain multiple `docker-compose.yaml` configurations.
Here, [level 1](./level_1) and [level 2](./level_2) are use to show what happens when configuration gets chained.

`level_1/level_1.env`
```sh
ENV_LEVEL=level_1
# stays the same, as ENV_LEVEL is taken from preceding level
#REPLACEMENT="env level: ${ENV_LEVEL}"
```

Run `docker-compose -f docker-compose.yaml -f level_1/docker-compose.yaml`:

```sh
environment:
  ENV_LEVEL: level_1
  REPLACEMENT: 'env level: level_0'
```

Note, that `REPLACEMENT` variable of level 0 will not get re-revaluated!
Also, even if uncommented, it also would stay the same as `ENV_LEVEL` from the preceding context would be used.

> :bulb: **Note:**
>
> If you still get `env level: host env` make sure to unset the variable by running `unset ENV_LEVEL` afterwards.
> [Shell variables have higher weight](https://docs.docker.com/compose/environment-variables/envvars-precedence/) than variables declared in `.env` or `evironment` variables.


Now, comment out `ENV_LEVEL` from `level_0.env` and re-run `docker-compose -f docker-compose.yaml -f level_1/docker-compose.yaml`.
You will receive a warning `WARN[0000] The "ENV_LEVEL" variable is not set. Defaulting to a blank string.`
Pretty obvious, as the variable is not declared anymore.
Commenting in `REPLACEMENT="env level: ${ENV_LEVEL}"` in `level_1/level_1.env` will re-evaluate `REPLACEMENT` again with the variable found within the same file.


Now, have a look at

`level_2/level_2.env`
```sh
ENV_LEVEL=level_2
# re-evaluates with ENV_LEVEL from preceding level
REPLACEMENT="env level: ${ENV_LEVEL}"
```

Run `docker-compose -f docker-compose.yaml -f level_1/docker-compose.yaml -f level_2/docker-compose.yaml`:

```sh
environment:
  ENV_LEVEL: level_1
  REPLACEMENT: 'env level: level_1'
```

Now, `level_1` is really used, as `REPLACEMENT` gets re-evaluated.



However, the replacement logic follows these rules:

- if variable exists from a preceding or higher ranked environment context, use it for replacement
- if such variable not exist use the variable from within the file for replacement
- otherwise use empty string and print out a warning


