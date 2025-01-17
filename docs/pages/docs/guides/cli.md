---
title: "Command Line"
description: "Learn how to use Keystone's command line interface (CLI) to develop, build, and deploy your Keystone projects."
---

Keystone's command line interface (CLI) has been designed to help you develop, build, and deploy your Keystone project.

Using the `keystone` command you can start the dev process, build and run your app in production, and control how you migrate the structure of your database as your schema changes.

### Usage

```text
$ keystone [command]

Commands
  dev           start the project in development mode (default)
  postinstall   build the project (for development, optional)
  build         build the project (required by \`keystone start\`)
  start         start the project
  prisma        run Prisma CLI commands safely
  telemetry     sets telemetry preference (enable/disable/status)
{% if $nextRelease %}

Options
  --fix (postinstall) @deprecated
    do build the graphql or prisma schemas, don't validate them

  --frozen (build)
    don't build the graphql or prisma schemas, only validate them

  --no-db-push (dev)
    don't push any updates of your Prisma schema to your database

  --no-prisma (build, dev)
    don't build or validate the prisma schema

  --no-server (dev)
    don't start the express server

  --no-ui (build, dev, start)
    don't build and serve the AdminUI

  --with-migrations (start)
    trigger prisma to run migrations as part of startup
{% /if %}

```

{% hint kind="tip" %}
All the commands expect to find a module called `keystone.js` (or `.ts`) with a default export that returns a Keystone System `config()` from `@keystone-6/core`. See [System Configuration](../config/config) for details.
{% /hint %}

## Setting up package.json

We recommend adding the following scripts to your project's `package.json` file:

```json
{
  "scripts": {
    {% if $nextRelease %}
    "build": "keystone build",
    "dev": "keystone dev",
    "postinstall": "keystone postinstall",
    "generate": "keystone prisma migrate dev",
    "start": "keystone start --with-migrations",
    {% else / %}
    "deploy": "keystone build && keystone prisma migrate deploy",
    "dev": "keystone dev",
    "postinstall": "keystone postinstall",
    "start": "keystone start"
    {% /if %}
  }
}
```

{% hint kind="tip" %}
Note: Depending on where you are deploying the `prisma migrate deploy` step might be better in the `build` or as a separate step altogether.
{% /hint %}

Read on below for more details about each command, and see [bringing it all together](#bringing-it-all-together) for more details (including some important caveats) about how that "deploy" command works.

## dev

```bash
$ keystone dev
```

This is the command you use to start Keystone for local development. It will:

- Generate the files required by Keystone's APIs and Admin UI
- Generate and apply database migrations based on your Keystone Schema
- Start the local dev server, which hosts the GraphQL API and Admin UI

{% if $nextRelease %}
### dev flags

- `--no-db-push` - Don't push any updates of your Prisma schema to your database
- `--no-prisma` - Don't build or validate the prisma schema
- `--no-server` - Don't start the express server, will still watch for changes and update the Admin UI and schema files
- `--no-ui` - Don't build and serve the AdminUI
{% /if %}

### About database migrations

{% if $nextRelease %}
When using `keystone dev` the default behaviour is for Keystone to update your database to match your schema using Prisma Migrate. This behaviour is great for the rapid iteration of a schema, but can be modified in the following ways:

- Running `keystone dev --no-db-push` - This will skip the dev migration step and not perform any checks on your database to ensure it matches your schema. This can be useful if you have an existing database or want to handle all migrations yourself. Be aware that this may lead to GraphQL runtime errors if a table or table column is unavailable.

See [`prisma` command](#prisma) below for more information on database migrations.

{% hint kind="tip" %}
Be careful of running `keystone dev` while pointing to a production database as this can cause data loss.
{% /hint %}
{% else /%}
In development, Keystone will migrate the structure of your database for you in one of two ways depending on how you have configured `db.useMigrations`:

- If `db.useMigrations` is `false` (the default), Keystone will use Prisma Migrate to update your database so that it matches your schema. It may lose data while updating your database so you should only use this mode in initial development.
- If `db.useMigrations` is `true`, Keystone will use Prisma Migrate to apply any existing migrations and prompt you to create a migration if there was a change to your schema.

When you are using migrations, the recommended workflow is to have `keystone dev` generate the migrations for you and apply them automatically in development.

Commit the migration files to source control, then when you are hosting your app in production, use the `keystone prisma migrate deploy` command (see below) to deploy your migrations before starting Keystone.

{% hint kind="tip" %}
We strongly recommend enabling migrations if you are going to run your app in production. Not doing so risks data loss.
{% /hint %}
{% /if %}

### Resetting the database

From time to time, in development you may want to reset the database and recreate it from scratch.
You can do this by using Prisma:

```bash
$ keystone prisma db push --force-reset
```

{% hint kind="error" %}
Doing this will destroy your database, including all data
{% /hint %}

This is typically useful early in a project's development lifecycle, when you want to test database initialisation scripts or create a fresh set of test data.

## postinstall

```bash
$ keystone postinstall
```
{% if $nextRelease %}
{% hint kind="tip" %}
Note: `postinstall` is an alias for `keystone build --no-ui --frozen` we recommend switching to this `build` command
{% /hint %}
{% /if %}

Keystone generates several files that your app may depend on, including the Node.js API and TypeScript definitions.

While they are generated by Keystone's `dev` command, we strongly recommend you add the `postinstall` command to your project's `package.json` to ensure the files are always present and up to date; otherwise you may see intermittent errors in your code editor and CI workflows.

### Fixing validation errors

The `postinstall` command will warn you if any of the generated files you should have committed to source control (such as the Prisma and GraphQL Schemas) are out of sync with your Keystone Schema.

While the recommended way to fix this problem is to start your app using `keystone dev` (which will update these files) and commit any changes into source control, if you need to you can use the `--fix` flag to have the `postinstall` command fix the files for you:

```bash
$ keystone postinstall --fix
```
{% if $nextRelease %}
{% hint kind="tip" %}
Note: `postinstall --fix` is an alias for `keystone build --no-ui` we recommend switching to this `build` command
{% /hint %}
{% /if %}

## build

```bash
$ keystone build
```

This command generates the files needed for Keystone to start in **production** mode. You should run it during the build phase of your production deployment.

It will also validate that the generated files you should have committed to source control are in sync with your Keystone Schema.
{% if $nextRelease %}

### build flags
- `--frozen` - Don't update the graphql or prisma schemas, only validate them, exits with error if the schemas don't match what keystone would generate.
- `--no-prisma` - Don't build or validate the prisma schema
- `--no-ui` - Don't build the AdminUI
{% /if %}

## start

```bash
$ keystone start
```

This command starts Keystone in **production** mode. It requires a build to have been generated (see `build` above).

It will not generate or apply any database migrations - these should be run during the **build** or **release** phase of your production deployment.
{% if $nextRelease %}

### start flags
- `--with-migrations` - Trigger prisma to run migrations as part of startup
- `--no-ui` - Don't serve the AdminUI
{% /if %}

## prisma

```bash
$ keystone prisma [command]
```

Keystone's CLI includes a `prisma` command which allows you to run any [Prisma CLI commands](https://www.prisma.io/docs/reference/api-reference/command-reference/) safely within your keystone project.

It will ensure your generated files (importantly, the Prisma Schema) are in sync with your Keystone Schema, and pass the configured database connection string through to Prisma as well.

### Working with Migrations

The primary reason you'll need to use the Prisma CLI in your Keystone project is to work with Prisma Migrate. For example, to deploy your migrations in production, you would run:

```bash
$ keystone prisma migrate deploy
```

This will run the Prisma [`migrate deploy`](https://www.prisma.io/docs/reference/api-reference/command-reference/#migrate-deploy) command.

As you start working with more sophisticated migration scenarios, you'll probably also need to use the [`migrate resolve`](https://www.prisma.io/docs/reference/api-reference/command-reference/#migrate-resolve) and [`migrate status`](https://www.prisma.io/docs/reference/api-reference/command-reference/#migrate-status) commands.

While Keystone abstracts much of the Prisma workflow for you, we strongly recommend you familiarise yourself with the Prisma Migrate CLI commands when you are managing your application in production.

## Bringing it all together

Although there are many subtle ways you might customise your workflow with Keystone based on your project's requirements, this is a fairly typical way of working:

### Postinstall after git clone and pull

Setting up the `postinstall` script will make sure that Keystone's generated files are all present and correct when you get latest from source control and install dependencies.

This will also run in CI, which means your lint rules and tests will be using the latest version of your Keystone build.

### Developing your project

Use `keystone dev` to start Keystone for development. Make sure you commit any changes to generated files, including both the Prisma and GraphQL Schema files, and migrations.

### Deploying to production

Most application hosting platforms allow you to specify a `build` script (for new deployments) and a `start` script (for when the app is started, including new instances when scaling up).

#### Build Script

Install the project dependencies, generate the build and deploy migrations:

```bash
yarn && yarn keystone build && yarn keystone prisma migrate deploy
```

{% hint kind="error" %}
Note: it is only safe to run migrations in the build step if deploys are built synchronously by your hosting provider. Make sure you don't run migrations against your production database from staging / preview builds.
{% /hint %}

#### Start Script

Start Keystone in production mode:

```bash
yarn keystone start
```
{% if $nextRelease %}
{% hint kind="tip" %}
Note: To run migrations before you start Keystone use `keystone start --with-migrations`
{% /hint %}
{% /if %}

#### Notes

- These examples use `yarn` as the package manager, you can use others like `npm` or `pnpm` if you prefer.
- The commands above are included in the `package.json` reference at the top of this page. We recommend using package scripts so your build and start commands are centralised in source control.
- If you promote your build through separate environments in a pipeline (e.g testing → staging → production) you should run migrations during the **promote** step and **not** as part of the build script.
- It is important you do **not run migrations against your production database from staging builds**. If you have staging or preview environments set up in production, make sure they are not pointed to your production database.

## Related resources

{% related-content %}
{% well
heading="Getting Started with create-keystone-app"
href="/docs/getting-started" %}
How to use Keystone's CLI app to standup a new local project with an Admin UI & the GraphQL Playground.
{% /well %}
{% /related-content %}
