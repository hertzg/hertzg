# Using buildx to build docker images for foreign architectures separately using qemu and publishing as one multi-arch image to docker hub.

Steps involved in words:

1. Build image for each architecture and push to temp registry
2. Create a manifest list grouping them togeather in the temp registry
3. Use scopeko to copy from temp registry to public registry

These steps are easier said then done, few things need to happen first.

## Example project

Let's image a case where we have a project that runs on docker. We would like to build images for the following
platforms.

* linux/amd64
* linux/arm64/v8
* linux/arm/v7
* linux/arm/v6
* linux/ppc64le
* linux/s390x

The build should happen in parallel for each platform, but only publish one "multi-arch" image (in other words a
manifest list).

Here's a sample app

```javascript
// app.js
const http = require('http');

const port = 3000;

const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.end('Hello World');
});

server.listen(port, () => {
    console.log(`Server running at %j`, server.address());
});
```

And it's complementing (not very good) Dockerfile

```
FROM node:14-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
WORKDIR /app
COPY ./app.js ./app.js
CMD [ "node", "/app/app.js" ]
EXPOSE 3000
```

## Step 1.1: Setup

To perform the first step of we need to set-up a few things:

* registry
* qemu - to emulate different cpus for building
* binfmt
* buildx builder that has access to all above

### Step 1.1.1: registry

First start a v2 registry and expose as an **INSECURE** `localhost:5000`.

```
docker run --rm --name registry -p 5000:5000 registry:2
```

### Step 1.1.2:  qemu, binfmt & buildx

Now setup `qemu`, `binfmt` configuration to use that `qemu` and create a special `buildx` container which has access to
host network.

```
sudo apt-get install qemu-user-static

docker run --privileged --rm tonistiigi/binfmt --install all

docker buildx create \
                --name builder \
                --driver docker-container \
                --driver-opt network=host \
                --use

docker buildx inspect builder --bootstrap
```

The `tonistiigi/binfmt --install all` is a docker container "with side-effects" that will set up `binfmt`
configuration **on your host**.

The `--driver-opt network=host` will allows the `buildx` container to reach the `registry` running on host
at `localhost:5000`.

The `buildx inspect --bootstrap` will kickoff the contianer and print it's information for us.

## Step 1.2: Build

**NOTE**: Buildx by itself runs the builds in parallel if you provide a comma separated list of platforms
to `buildx build` command as `--platform` flag.

The problem for me and the whole reason of writing this post is that if the build with multiple `--platforms`
fails for **one of the platforms** then the whole build is marked as failed and you get nothing.

Another use case can also be that together with one multi-arch image maybe you want to push arch specific repositories (
eg: `docker.io/app/app`, `docker.io/arm64v8/app` and `docker.io/amd64/app`).

The other case is that I do builds on multiple actual machines that natively have `arm/v6`, `arm/v7` and `arm64/v8`
cpus (a cluster of different Pis and similar).

There are probably even more reasons why you would want to build them this way 🤷.

Now we are ready to start building for different architectures with our `buildx` builder for this example.

The base `alpine` image supports the following architectures.

* linux/amd64
* linux/arm/v6
* linux/arm/v7
* linux/arm64/v8
* linux/ppc64le
* linux/s390x

lets target all of them 😎

```
docker buildx build \
        --tag localhost:5000/app:linux-amd64 \
        --platform linux/amd64 \
        --load \
        --progress plain \
        . > /dev/null 2>&1 &
docker buildx build \
        --tag localhost:5000/app:linux-arm-v6 \
        --platform linux/arm/v6 \
        --load \
        --progress plain \
        .> /dev/null 2>&1 &
docker buildx build \
        --tag localhost:5000/app:linux-arm-v7 \
        --platform linux/arm/v7 \
        --load \
        --progress plain \
        .> /dev/null 2>&1 &
docker buildx build \
        --tag localhost:5000/app:linux-arm64-v8 \
        --platform linux/arm64/v8 \
        --load \
        --progress plain \
        .> /dev/null 2>&1 &
docker buildx build \
        --tag localhost:5000/app:linux-ppc64le \
        --platform linux/ppc64le \
        --load \
        --progress plain \
        .> /dev/null 2>&1 &
docker buildx build \
        --tag localhost:5000/app:linux-s390x \
        --platform linux/s390x \
        --load \
        --progress plain \
        .> /dev/null 2>&1 &
wait

```

Once this is done, the images will be loaded and visible with `docker images` command

```
$ docker images

...
localhost:5000/app   linux-arm64-v8    e3ec56e457e6   About a minute ago   115MB
localhost:5000/app   linux-arm-v7      ab770e5be5d1   About a minute ago   106MB
localhost:5000/app   linux-ppc64le     3a328d516acf   About a minute ago   126MB
localhost:5000/app   linux-s390x       73e064c0c3d4   About a minute ago   119MB
localhost:5000/app   linux-amd64       f6260fedf498   About a minute ago   116MB
localhost:5000/app   linux-arm-v6      5a1fb75b0a45   2 minutes ago        110MB
...
```

There is no need to `--load` the images to your local docker, you can make `buildx` directly push to our local registry.

For this example you have to push these images as an extra step

```
docker push --all-tags -q localhost:5000/app
```

## Step 2: Manifest List

Now we only need to group those images together into one big manifest list.

```
docker manifest create --insecure
    localhost:5000/app:1.0.0 \
        localhost:5000/app:linux-amd64 \
        localhost:5000/app:linux-arm-v6 \
        localhost:5000/app:linux-arm-v7 \
        localhost:5000/app:linux-arm64-v8 \
        localhost:5000/app:linux-ppc64le \
        localhost:5000/app:linux-s390x

docker manifest create localhost:5000/app:1.0.0
```

## Step 3.1: Skopeo

The one last step is to copy the manifest list and only the blobs that are linked by it. For this we need `skopeo`, an
amazing tool for working with registries.

You can either build form source or for ubuntu 20.04 we can use the prebuilt `kubic` packages.

**NOTE**: You don't have to install it with this script, just follow the install guide
at https://github.com/containers/skopeo/blob/master/install.md

```
OS="x$(lsb_release --id -s)_$(lsb_release --release -s)"
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/ /" > /etc/apt/sources.list.d/skopeop.kubic.list
wget -qO- "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/Release.key" | apt-key add -
apt-get update
apt-get install -y skopeo
```

Now because our local registry is insecure `skopeo` will complain when we try to copy from it, so we need to explicitly
configure it to allow insecure connections to our temp registry.

```
[[registry]]
location = 'localhost:5000'
insecure = true
```

Create this file in `/etc/containers/registries.conf.d/localhost-5000.conf`

## Step 3.2: Copy

The final step is to copy only the `localhost:5000/app:1.0.0` to lets say `hertzg/example:app-1.0.0`.

First you might need to authenticate with your target registry

```
skopeo login docker.io
```

Now we can finally copy the image

```
skopeo copy \
        --all \
        docker://localhost:5000/app:1.0.0 \
        docker://docker.io/hertzg/example:app-1.0.0
```

This might take some time but once it's finished you can check the docker hub or just pull and run the image on target
architectures

```
docker run --rm -it hertzg/example:app-1.0.0
```
