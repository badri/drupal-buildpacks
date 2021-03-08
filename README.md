
# Cloud native continuous delivery using Paketo buildpacks

Doing a "git push" and be done with deployment has been a developer utopia ever since Heroku popularized it.  The good news is, we can do all of this within our Kubernetes cluster. Buildpacks provide a rich ecosystem where you can build your application's container images without messing up with writing Dockerfiles and worrying about security.

In this article, I'll walk you through how to build and deploy a Drupal 8 application from the git push stage to shipping to production.


## Why buildpacks?


## First cut

Prerequisites:

1.  Drupal 8 project/codebase
2.  Pack cli. Instructions to download [here](https://paketo.io/docs/)
3.  Docker

Let's get the source code first.

    composer create-project "drupal/recommended-project:^8" d8-buildpack

Pack needs answers to 2 questions to build a container image.

What are you going to build? - The answer's mostly your application source code.

How are you going to build it?
This is not a one-size-fits-all. Case in point, our Drupal 8 is a PHP application will be built by running `composer install` to manage the dependencies. A Drupal 7 doesn't use composer, but it's still a PHP application. For some PHP applications, we might use Nginx, for others Apache etc.

The buildpack specification provides a set of images which can be assembled in a required sequence. This allows us to tailor our build process down to the tiniest detail.

Now, think of each of these buildpacks as a part of a build "pipeline", where 4 processes are run, in that order:

1.  Detection - check if this buildpack is relevant to building the app
2.  Analysis - see how this buildpack run can be optimzed
3.  Build - the actual build process, in our case, running composer install
4.  Export - The final OCI-compatible image of this buildpack.

This is termed "Lifecycle" in the Buildpack spec document.

Finally, we have a "stack", which specifies what image to use for building the app, and what image to use for running the app. Now, why would we have separate images for each? There might be some dependencies which will be relevant while building the application, but unnecessary for the built application to run. This also keeps the runtime image as lean as possible without any extra libraries and unwanted dependencies.

### A note about OCI-compatible images

OCI(Open Container Initiative) compatible images are container images which are compatible with any container runtime that's available out there. This is a minimum guarantee stating that the container image you build using Paketo buildpacks can be run on any container runtime, like ContainerD and CRI-O.

### Build image vs runtime image

A builder is a combination of all the available buildpacks, the lifecycle and a stack.
Armed with this concept, let's take a stab at how we will build our code. First, we pick and choose what builder image we want to use.

    $ pack builder suggest
    Suggested builders:
            Google:                gcr.io/buildpacks/builder:v1      Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python
            Heroku:                heroku/buildpacks:18              heroku-18 base image with buildpacks for Ruby, Java, Node.js, Python, Golang, & PHP
            Paketo Buildpacks:     paketobuildpacks/builder:base     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Ruby, NGINX and Procfile
            Paketo Buildpacks:     paketobuildpacks/builder:full     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, PHP, Ruby, Apache HTTPD, NGINX and Procfile
            Paketo Buildpacks:     paketobuildpacks/builder:tiny     Tiny base image (bionic build image, distroless-like run image) with buildpacks for Java Native Image and Go

The `paketobuildpacks/builder:full` builder looks like a good bet, as it contains all the required libraries for running PHP and most of its extensions.

We can set that as the default builder image.

    $ pack config default-builder paketobuildpacks/builder:full

## Defining our buildpacks

At the time of writing this, the PHP buildpack was undergoing a [major refactor](https://github.com/paketo-buildpacks/php/blob/main/rfcs/0001-restructure.md) and I wasn't able to build a Drupal 8 application. This was a great excuse to try my hands at creating a custom buildpack.
Let's dive straight into how to create a buildpack. The first rule is, a buildpack is a highly composable entity. It is not atypical for your application to be built using 3-4 buildpacks.
Each buildpack will have an entry criteria defined by an executable `detect`. If this entry criteria fails, that buildpack will not be used to build your image and will be passed on.
Ex: The entry criteria for a PHP composer buildpack is the presence of a `composer.json` in the top level directory, failing which the composer buildpack won't feature while building your application  container.
The next important thing is a `build` executable, which does the actual job of the buildpack. In the composer example, it will run `composer install`. It will also do a few other things before and after this, like installing composer, doing a post-install clean up etc. Running `composer install` everytime the buildpack "builds" is fine, but installing composer everytime the buildpack is run seems unintuitive. Hence, Packeto gives buildpack authors the provision to cache build artifacts, so that they can be reused in the next build. This notion of caching the build dependencies is there in most programming languages in some shape or form. Paketo buildpacks leverage this and cache them appropriately. In our case, we will cache the composer executable and the vendor directory, so that they're not downloaded in every build.

There is a `buildpack.toml` file which contains meta information about a buildpack, such as a uniquely identifiable name of the buildpack, what version of the buildpack spec it is using, which builder images it is compatible with etc. That is most of what's needed about creating your own buildpack.

For building a PHP application, we need the following buildpacks:
1. A PHP buildpack which installs PHP 7.4(latest stable at the time of writing this)
2. A composer buildpack which installs the dependencies into the source directory(most modern PHP applications will need this)
3. PHP FPM, a FastCGI process manager, used for running PHP alongside a webserver like Nginx and Apache
4. Nginx or Apache, this could be optional depending on how you run your application.

### The webserver part

Some PHP applications just run a custom script in PHP. Those probably don't need the webserver buildpack and even the PHP FPM buildpack. Drupal needs a webserver, and for this example we choose Nginx. PHP doesn't understand HTTP and Nginx doesn't understand PHP. To bridge both, we use PHP FPM. **NOTE** that this approach would equally work well with Apache as well.

## Writing our buildpacks

The buildpack specification doesn't dictate how you write your buildpacks, as long as it has an executable `detect` and `build`. To that extent, we will be using bash scripts to create our buildpacks.

### The detect stage
The PHP buildpack kicks in by default and assigns 7.4 as the default PHP version. If you want to override this, you can provide a `buildpack.yml` which will specify the PHP version.

    if [[ ! -f buildpack.yml ]]; then
      version="7.4"
    else
      version=$(cat buildpack.yml | yj | jq -r ".php .version")
    fi

    versions=("7.3" "7.4" "8.0")

    if ! [[ " ${versions[@]} " =~ " ${version} " ]]; then
      echo "$version not supported"
      exit 1
    fi

This information is passed on to the next stage by creating a ["build plan"](https://github.com/buildpacks/spec/blob/main/buildpack.md#build-plan-toml). This build plan is passed as an argument to both the detect and build stages.

    plan=$2

    echo "provides = [{ name = \"php\" }]" > "$plan"
    echo "requires = [{ name = \"php\", version = \"$version\" }]" >> "$plan"

### The build stage

In the detect stage, we download the version of PHP we "detected" in the previous stage.

    echo "---> PHP Buildpack"

    # Get args
    layersdir=$1
    plan=$3

    declare -A dl_urls=(
      ["7.3"]="https://buildpacks.cloudfoundry.org/dependencies/php/php_7.3.27_linux_x64_cflinuxfs3_267c03f0.tgz"
      ["7.4"]="https://buildpacks.cloudfoundry.org/dependencies/php/php_7.4.15_linux_x64_cflinuxfs3_4015c116.tgz"
      ["8.0"]="https://buildpacks.cloudfoundry.org/dependencies/php/php_8.0.2_linux_x64_cflinuxfs3_c6a58cc0.tgz"
    )

    # Download PHP
    phplayer="$layersdir"/php
    php_version=$(cat "$plan" | yj -t | jq -r '.entries[] | select(.name == "php") | .version')

We setup and install PHP and cache it so that we needn't do it unless
1. We're running the first time
2. The PHP version is changed by the developers

```shell
# clean up if current version is not same as installed version
if [[ -f "$phplayer/version" ]]; then
  installed_version=$(cat "$phplayer/version")
  if [[ "$installed_version" != "$php_version" ]]; then
    rm -rf "$phplayer"
  fi
fi

# Download
if [[ ! -f "$phplayer/bin/php" ]]; then
  echo "---> Downloading and setting up PHP $php_version"
  mkdir -p "$phplayer"
  php_url="${dl_urls[$php_version]}"
  wget -q -O - "$php_url" | tar -xzf - -C "$phplayer"
  echo "$php_version" > "$phplayer/version"
fi
```

Every buildpack build stage is ephemeral, in the sense that it doesn't add any file to the final image to be shipped unless we say so.
We want the PHP executable to be available in the final image. The buildpack way of saying this is to create a [`launch.toml`](https://github.com/buildpacks/spec/blob/main/buildpack.md#launchtoml-toml).

    # Make PHP available during launch
    echo -e 'launch = true\nbuild=true\ncache=true' > "$phplayer.toml"

On a similar vein, we can create the other buildpacks as well.
The composer buildpack with a
1. detect stage which will look out for a `composer.json` in the top level directory.
2. build stage which will download composer, run `composer install` and cache dependencies

The PHP FPM buildpack,
1. a dummy detect stage which will always pass.
2. a build stage which will install PHP FPM and run it as a process.

The Nginx buildpack,
1. another dummy detect stage.
2. the build stage which will look out for the docroot location in `buildpack.yml`, install and configure Nginx, run both Nginx and PHP FPM as a [supervisord](http://supervisord.org/) script.

## Building and testing our image

The next step is to take the buildpacks for a spin and use them for what they're intended to do, i.e. build images. We use the pack cli to do this.

    pack build -b ../php-cnb -b ../composer-cnb -b ../php-fpm-cnb -b ../nginx-cnb drupal-8-cnb

Notice the order of the buildpacks specified, from left to right. First, each buildpack's detect phase will run, figuring out whether the buildpack is participating in the build or otherwise. Then each buildpack is going to form a [layer](https://github.com/buildpacks/spec/blob/main/buildpack.md#build-layers) where it will download the dependencies required for that layer and execute the build script. Your output should look something like this:


    ===> DETECTING
    acme-buildpacks/php      0.0.1
    acme-buildpacks/composer 0.0.1
    acme-buildpacks/php-fpm  0.0.1
    acme-buildpacks/nginx    0.0.1
    ===> ANALYZING
    Previous image with name "drupal-8-cnb" not found
    ===> RESTORING
    ===> BUILDING
    ---> PHP Buildpack
    ---> Downloading and setting up PHP 7.4
    ---> PHP Composer Buildpack
    ---> Downloading and extracting Composer
    All settings correct for using Composer
    Downloading...

    Composer (version 2.0.11) successfully installed to: /layers/acme-buildpacks_composer/composer/composer
    Use it: php /layers/acme-buildpacks_composer/composer/composer

    ---> Running composer install
    ....
    ---> PHP Fpm Buildpack
    ---> Creating Fpm config
    ---> Nginx Buildpack
    ---> Downloading and extracting Nginx
    ---> Installing Supervisord
    ===> EXPORTING
    Adding layer 'acme-buildpacks/php:php'
    Adding layer 'acme-buildpacks/composer:composer'
    Adding layer 'acme-buildpacks/php-fpm:php-fpm'
    Adding layer 'acme-buildpacks/nginx:nginx'
    Adding 1/1 app layer(s)
    Adding layer 'launcher'
    Adding layer 'config'
    Adding layer 'process-types'
    Adding label 'io.buildpacks.lifecycle.metadata'
    Adding label 'io.buildpacks.build.metadata'
    Adding label 'io.buildpacks.project.metadata'
    Setting default process type 'web'
    *** Images (f5cda43eeee5):
          drupal-8-cnb
    Adding cache layer 'acme-buildpacks/php:php'
    Adding cache layer 'acme-buildpacks/composer:composer'
    Adding cache layer 'acme-buildpacks/nginx:nginx'
    Successfully built image drupal-8-cnb


Let's make sure our new image works.




This image can be published to the docker registry using `pack publish`.

## Shipping our buildpacks

It's time to take your buildpacks outside your laptop to the external world, so that your team members can use it and it can be incorporated into your CI/CD systems. This is where the `buildpack.toml` and `package.toml` files of your buildpack come into play.

The `package.toml` specifies where to find the buildpack that's being packaged. In all our buildpacks, it's the current directory. The `buildpack.toml` contains metadata about the buildpack. We can use the pack cli to package a buildpack as an OCI image.


    pack buildpack package registry.acme-corp.com/my-buildpack --config ./package.toml
    docker push registry.acme-corp.com/my-buildpack

So that everyone else can build their images from your buildpack.

    pack build -b registry.acme-corp.com/my-buildpack-1 -b registry.acme-corp.com/my-buildpack-2 my-app-image:v1.0


## Building buildpacks in the cluster

TODO kpack introduction

## Why not use dockerfiles?

TODO What benefits does CNB bring to the table

## How to update builder images

TODO layer update

## Conclusion

TODO other stacks

TODO Using packit
