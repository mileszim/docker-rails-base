![Build images](https://github.com/mileszim/docker-rails-base/workflows/Build%20images/badge.svg)

# DockerRailsBase

Building Docker images usually takes a long time. This repo contains base images with preinstalled dependencies for [Ruby on Rails](https://rubyonrails.org/), so building a production image will be **2-3 times faster**.


## What?

When using the official Ruby image, building a Docker image for a typical Rails application requires lots of time for installing dependencies - mainly OS packages, Ruby gems, Ruby gems with native extensions (Nokogiri etc.) and Node modules. This is required every time the app needs to be deployed to production.

In order to reduce this time, we can use these base images that contain most of the dependencies used in an applications.

As much as possible is moved from the app-specific Dockerfile into the base image by using [ONBUILD](https://docs.docker.com/engine/reference/builder/#onbuild) triggers. This makes the Dockerfile small and simple.


## Performance

Compared building times using a typical Rails application. This is the result on a local machine:

- Based on official Ruby image: **4:50 min**
- Based on DockerRailsBase: **1:57 min**

As you can see, using DockerRailsBase is more than **2 times faster** compared to the official Ruby image. It saves nearly **3min** on every build.

Note: Before the author started timing, the base image was not available on their machine, so it was downloaded first, which took some time. If the base image is already available, the building time is only 1:18min (**3 times faster**).


## How?

This repo is based on the following assumptions:

- Your app is compatible with [Ruby 2.7.2 for Alpine Linux](https://github.com/docker-library/ruby/blob/master/2.7/alpine3.12/Dockerfile)
- Your app uses Ruby on Rails 6.0 or 6.1
- Your app uses PostgreSQL
- Your app installs Node modules with [Yarn](https://yarnpkg.com/)
- Your app compiles JS with [Webpacker](https://github.com/rails/webpacker) and/or [Asset pipeline (Sprockets)](https://github.com/rails/sprockets-rails)

It uses [multi-stage building](https://docs.docker.com/develop/develop-images/multistage-build/) to build a very small production image. There are two Dockerfiles in this repo, one for the first stage (called `Builder`) and one for the resulting stage (called `Final`).

### Builder stage

The `Builder` stage installs Ruby gems and Node modules. It also includes Git, Node.js and some build tools - all we need to compile assets.

- Based on [ruby:2.7.2-alpine](https://github.com/docker-library/ruby/blob/master/2.7/alpine3.12/Dockerfile)
- Adds packages needed for installing gems and compiling assets: Git, Node.js, Yarn, PostgreSQL client and build tools
- Adds some standard Ruby gems (Rails 6.1 etc., see [Gemfile](https://github.com/mileszim/docker-rails-base/blob/master/Builder/Gemfile))
- Adds some standard Node modules (Vue.js etc., see [package.json](https://github.com/mileszim/docker-rails-base/blob/master/Builder/package.json))
- Via ONBUILD triggers it installs missing gems and Node modules, then compiles the assets

See [Builder/Dockerfile](https://github.com/mileszim/docker-rails-base/blob/master/Builder/Dockerfile)


### Final stage

The `Final` stage builds the production image, which includes just the bare minimum.

- Based on [ruby:2.7.2-alpine](https://github.com/docker-library/ruby/blob/master/2.7/alpine3.12/Dockerfile)
- Adds packages needed for production: postgresql-client, tzdata, file, libsodium
- Via ONBUILD triggers it mainly copies the app and gems from the `Builder` stage

See [Final/Dockerfile](https://github.com/mileszim/docker-rails-base/blob/master/Final/Dockerfile)


### Staying up-to-date

Using [Dependabot](https://dependabot.com/), every updated Ruby gem or Node module results in an updated image.


### How to use for your Rails application

#### Build Docker image

```Dockerfile
FROM mileszim/rails-base-builder:2.7.2-alpine AS Builder
FROM mileszim/rails-base-final:2.7.2-alpine
USER app
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

Yes, this is the complete Dockerfile of the Rails app. It's simple because the work is done by ONBUILD triggers.


#### Continuous integration (CI)

Example to build the application's image with GitHub Actions and push it to the GitHub Container Registry:

```yaml
deploy:
  runs-on: ubuntu-latest

  steps:
    - uses: actions/checkout@v2

    - name: Login to GitHub Container Registry
      run: echo ${{ secrets.CR_PAT }} | docker login ghcr.io -u $GITHUB_ACTOR --password-stdin

    - name: Build the image
      run: |
        export COMMIT_TIME=$(git show -s --format=%ci ${GITHUB_SHA})
        export COMMIT_SHA=${GITHUB_SHA}
        docker build --build-arg COMMIT_TIME --build-arg COMMIT_SHA -t ghcr.io/user/repo:latest .

    - name: Push the image
      run: docker push ghcr.io/user/repo:latest
```


## FAQ

### Why not simply use layer caching?

Docker supports layer caching, so for building images it performs just the needed steps: If there is a layer from a former build and nothing has changed, it will be used. But for dependencies, this means: If a single Ruby gem in the application was updated or added, the step with `bundle install` is run again, so **all** gems will be installed again.

Using a prebuilt image improves installing dependencies a lot, because only the different/updated dependencies will be installed - all existing ones will be reused.

### What if my app requires slightly different dependencies?

This doesn't matter:

- A missing Alpine package can be installed with `apk add` inside your app's Dockerfile.
- A missing Node module (or version) will be installed with `rails assets:precompile` via the ONBUILD trigger.
- A missing Ruby gem (or version) will be installed with `bundle install` via the ONBUILD trigger.

### There are gems included that my app doesn't need. Will they bloat the resulting image?

No. In the build stage there is a `bundle clean --force`, which uninstalls all gems not referenced in the app's Gemfile.
