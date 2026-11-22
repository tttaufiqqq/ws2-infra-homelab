# app-build

The container build for the Animal Shelter Workshop app, moved here from the application repository so
that repository holds application code only. Paths inside (`COPY composer.json`, `resources/`, and so on)
refer to the app repository's root, so this does not build on its own: it is a reference.

- `Dockerfile`: multi-stage build with a `runtime` target (php-fpm) and an `nginx` target (static assets).
- `docker/`: the nginx site config and PHP-FPM tuning used by the image.
- `docker-compose.yml`: a local practice environment wired to the five dev databases.
- `.dockerignore`: files kept out of the build context.

See [Architecture](../docs/architecture.md#4-delivery-pipeline) for how the images were built and rolled out.
