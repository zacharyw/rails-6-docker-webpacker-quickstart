# Rails 6 Quick Start with Docker and Webpacker

This project contains all you need to start developing web apps with Rails 6 using [Webpack](https://webpack.js.org/) to manage your CSS and javascript assets, via the [webpacker](https://github.com/rails/webpacker) gem.

We'll use docker to run our code, webpack server, and database, so this development environment can be used in any OS environment. If you mess up or something goes wrong, feel free to delete your containers and try again - that's what they're there for.

If you want to use [Nuxt](https://nuxtjs.org) for your front end instead of Rails, see [my other boilerplate repo](https://github.com/zacharyw/nuxt-rails-docker-boilerplate).

## Prerequisites

* [Git](https://git-scm.com/downloads)
* [Docker](https://www.docker.com/products/docker-desktop)

## Getting Started

Download the contents of this repo (except for this README) to a directory on your
computer that you want to contain your new Rails app.

For example: `~/Projects/myapp/`

You should name your directory whatever you want your Rails app to be named.

* **Dockerfile** - Defines the dependencies of the container that our Rails app will run
in. We start with a base image, `FROM`, that includes Ruby, Node, and Yarn.
* **docker-compose.yml** - Defines the services that make up our application. To get
started all we need is a database, a web server where Rails will run, and a 
webpack server that will serve and hot reload our assets. The postgres db is created
using a pre-made image, while the webpack and web services are built using our
**Dockerfile**.
* **Gemfile** - A basic gemfile that just describes what version of Rails we want.
When we create and install our app this file will be overwritten with all of
Rails dependencies.

### Create database volume

Run

```bash
docker volume create --name=pgdata
```

Docker containers are ephemeral. You can destroy and recreate them as often as you
want. This volume will allow our DB data to survive such purges.

#### DB Volume Alternative

*Doesn't work on Windows*

Instead of creating a Docker volume, you can
use the `volumes` attribute under the `db` service, and mount a local
directory (./tmp/db):

```yaml
db:
  image: postgres
  volumes:
    - ./tmp/db:/var/lib/postgresql/data
```

You could then delete the top level `volumes` attribute:

```yaml
volumes:
  pgdata:
    external: true
```

However, due to file ownership issues, this approach won't work on Windows and
you'll need to stick with using a Docker volume.

### Create rails application

Run:
```bash
docker-compose run --no-deps web rails new . --force --database=postgresql --skip-sprockets --skip-coffee --skip-test --webpack
```

This will run the `rails new` command on our `web` service defined in docker-compose.yml.

Flag explanations:
* **--no-deps** - Tells `docker-compose run` not to start any of the services in `depends_on`.
* **--force** - Tells rails to overwrite existing files, such as Gemfile.
* **--database=postgresql** - Tells Rails to default our db config to use postgres.
* **--skip-sprockets** - Since we're using webpacker we don't need sprockets or the asset pipeline.
* **--skip-coffee** - We're going to be writing ES6 JS, so we don't need coffeescript.
* **--skip-test** - We're going to install Rspec, so we don't need the unit test framework that comes with rails.
* **--webpack** - Tells Rails to go ahead and install webpack.

If everything went well you should have a directory full of Rails boilerplate.

### Re-build web image

Run
```bash
docker-compose build
```

Since we now have a new Gemfile with all of our Rails gem dependencies defined, we need to rebuild our web image.
This will re-execute the Dockerfile to build our image. Anytime you change your gemfile or tweak your Dockerfile,
you'll need to rebuild.


### Configure database

Next open `config/database.yml`.

It's already configured for postgres, but we need some additional configuration to get it to work with our `db`
container.

Change the `&default` config to match the following:
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # http://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: db
  username: postgres
  password:
```

This uses the default username and password (which is empty) for the postgres image. It also changes `host` to point
to `db`, the name of our service.

### Configure Webpacker

Similar to the database, we need to point our webpacker config at our running webpack service. In `config/webpacker.yml`, change
```
host: localhost
```
and
```
public: localhost:3035
```
to
```
host: webpack
```
and
```
public: webpack:3035
```

### Start the application

Run
```bash
docker-compose up -d
```

This will start all of our services in the background:

```bash
Creating railspacker-test_db_1      ... done
Creating railspacker-test_webpack_1 ... done
Creating railspacker-test_web_1     ... done
```

Run:

```bash
docker ps
```

You should see all three defined services running.

### Create database

Run:

```bash
docker-compose run web rake db:create
```

This will build our development and test databases.

### Visit application

In your browser, go to `http://localhost:3000`. You should be greeted by the Rails splash image.

### Install a CSS library (optional)

In this example we are using [Bulma](https://bulma.io/), but you could use Twitter bootstrap or any other CSS framework.

Run

```bash
docker-compose run web yarn add bulma
```

Next, create a new file, `./app/javascript/packs/application.scss`, and add the following code:

```javascript
@import '~bulma/bulma';
```

Then, in `./app/views/layouts/application.html.erb`, change `stylesheet_link_tag` to
`stylesheet_pack_tag`.

### Install rspec

Follow these directions to install Rspec: https://github.com/rspec/rspec-rails#installation

Remember that you'll need to `docker-compose build` after adding the gem to your Gemfile, and you'll need to use
`docker-compose run web` to run the `rails generate` command.

## Build Your Application

You're now ready to take your application in whatever direction you choose.

To shut down your containers, simply run `docker-compose down`.
