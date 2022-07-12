---
title: Moving container images from docker.io to quay.io
date: 2019-09-30
url: /blog/moving-container-images-from-docker-io-to-quay-io/
categories:
  - containers
  - skopeo
  - quay
summary: A short memo on how we can move container images from the docker.io registry to quay.io.
weight: 1
cover:
  image: "images/moving.jpg"
---

Have a look at how we can move container images from the docker.io registry to quay.io.

<!--more-->

Thanks to [Skopeo](https://github.com/containers/skopeo#copying-images), we can copy container images from one registry to another.

In this post, we'll copy images from [docker.io](https://hub.docker.com/) to [quay.io](https://quay.io/), a container registry which has a lot of features that docker.io doesn't provide. Two of them that I really like are:

- List and manage image vulnerabilities and other security information
- Manage the manifests of an image

If you want more information, checkout their [documentation](https://docs.quay.io/).

I wrote a small script that one can use to automate the copy of images.

Before running the script:

- Get OAuth token from: https://quay.io/organization/[your-org]?tab=applications
- Change  the token, namespace, containers and tag (if needed)
- If your docker.io registry requires authentication, you'll need to run `podman login docker.io` (`--src-creds` option could also be used with Skopeo)
- You'll need to authenticate against your quay.io registry with `podman login quay.io` (`--dest-creds` option could also be used with Skopeo)

```bash
#!/bin/sh
set -ex

# get OAuth token from https://quay.io/organization/[your-org]?tab=applications
token='secrete'
namespace=yourorg
containers='app1 app2'
tag=latest

retry() {
    local -r -i max_attempts="$1"; shift
    local -r cmd="$@"
    local -i attempt_num=1
    until $cmd
    do
        if ((attempt_num==max_attempts))
        then
            echo "Attempt $attempt_num failed and there are no more attempts left!"
            return 1
        else
            echo "Attempt $attempt_num failed! Trying again in $attempt_num seconds..."
            sleep $((attempt_num++))
        fi
    done
}

for container in $containers; do
    # create empty public repo first otherwise skopeo will create the image as private
    curl -X POST https://quay.io/api/v1/repository \
        -d '{"namespace":"'$namespace'","repository":"'$container'","description":"Container image '$container'","visibility":"public"}' \
        -H 'Authorization: Bearer '$token'' -H "Content-Type: application/json"
    # workaround if quay.io returns 500 error, likely due to an internal bug when using skopeo against docker.io
    copy="skopeo copy docker://docker.io/$namespace/$container:$tag docker://quay.io/$namespace/$container:$tag"
    retry 5 $copy
done
```

As you can see, there are 2 unusual things in this script:

- The `curl` creates an empty public image otherwise quay.io would create a private image by default when copying the image with Skopeo. As far as I know, there is no option in quay.io to change the default policy. Of course, remove it if you don't want your image to be public by default.
- The retry mechanism is to workaround the 500 error that you might get when it provisions a new repository, and it says it already exists (sounds specific to how the registry receives authentication from Skopeo vs Docker CLI).

Enjoy Skopeo & quay.io!
