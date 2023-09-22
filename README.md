# Evaluation of variables within docker compose


Tries to make transparent how variable evaluation works in docker compose.
We show the normal case, but also demonstrate the effect when multiple configuration files are used.

In general, Docker merges configuration by respecting a previously defined context.
We are focusing here on using `env_files` declared and the `environment` result of the container to be built.

## Single level (level 0)


First of a



## Multiple levels

