---
author: Luis Morales
date: '2016-12-04T00:00:00Z'
linktitle: Makefile is Awesome
title: Makefile is Awesome

---


Make is one of those utilities you don't think about very often, but given that almost every system has it installed by default makes it the ideal hassle free choice to bootstrap and automate your projects.

When we use it in conjunction with modern tools like Docker, means we don't even have to get fancy with it.

I use it it in all my projects to manage my Docker builds, pushes, test runs, and other conveniences.

Main advantages:

* shallow learning curve
* easy customisation, flexibility
* plugins are not needed, cli commands are powerful enough

this is how my Makefile starter template looks like [as part of my golang starter template](https://github.com/lacion/cookiecutter-golang):

```

.PHONY: build build-alpine clean test help default

BIN_NAME={{cookiecutter.app_name}}

VERSION := $(shell grep "const Version " version.go | sed -E 's/.*"(.+)"$$/\1/')

GIT_COMMIT=$(shell git rev-parse HEAD)

GIT_DIRTY=$(shell test -n "`git status --porcelain`" && echo "+CHANGES" || true)

IMAGE_NAME := "{{cookiecutter.docker_hub_username}}/{{cookiecutter.app_name}}"

default: test

help:

@echo 'Management commands for {{cookiecutter.app_name}}:'

@echo

@echo 'Usage:'

@echo '    make build           Compile the project.'

@echo '    make get-deps        runs glide install, mostly used for ci.'

{% if cookiecutter.use_docker == "y" %}@echo '    make build-alpine    Compile optimized for alpine linux.'

@echo '    make build-docker    Build inside an alpine docker container'

@echo '    make package         Build final docker image with just the go binary inside'

@echo '    make tag             Tag image created by package with latest, git commit and version'

@echo '    make test            Run tests on a compiled project.'

@echo '    make push            Push tagged images to registry'{% endif %}

@echo '    make clean           Clean the directory tree.'

@echo

build:

@echo "building ${BIN_NAME} ${VERSION}"

@echo "GOPATH=${GOPATH}"

go build -ldflags "-X main.GitCommit=${GIT_COMMIT}${GIT_DIRTY} -X main.VersionPrerelease=DEV" -o bin/${BIN_NAME}

get-deps:

glide install

{% if cookiecutter.use_docker == "y" %}

build-alpine:

@echo "building ${BIN_NAME} ${VERSION}"

@echo "GOPATH=${GOPATH}"

go build -ldflags '-w -linkmode external -extldflags "-static" -X main.GitCommit=${GIT_COMMIT}${GIT_DIRTY} -X main.VersionPrerelease=VersionPrerelease=RC' -o bin/${BIN_NAME}

build-docker:

@echo "building ${BIN_NAME} ${VERSION}"

docker build -t {{cookiecutter.app_name}}:build -f Dockerfile.build .

docker run --name={{cookiecutter.app_name}} -v $(GOPATH):/gopath/  {{cookiecutter.app_name}}:build

docker rm -f {{cookiecutter.app_name}}

package:

@echo "building image ${BIN_NAME} ${VERSION} $(GIT_COMMIT)"

docker build --build-arg VERSION=${VERSION} --build-arg GIT_COMMIT=$(GIT_COMMIT) -t $(IMAGE_NAME):local .

tag:

@echo "Tagging: latest ${VERSION} $(GIT_COMMIT)"

docker tag $(IMAGE_NAME):local $(IMAGE_NAME):$(GIT_COMMIT)

docker tag $(IMAGE_NAME):local $(IMAGE_NAME):${VERSION}

docker tag $(IMAGE_NAME):local $(IMAGE_NAME):latest

push: tag

@echo "Pushing docker image to registry: latest ${VERSION} $(GIT_COMMIT)"

docker push $(IMAGE_NAME):$(GIT_COMMIT)

docker push $(IMAGE_NAME):${VERSION}

docker push $(IMAGE_NAME):latest

{% endif %}

clean:

@test ! -e bin/${BIN_NAME} || rm bin/${BIN_NAME}

test:

go test $(glide nv)

```

in this case i used it to make 3 steps run docker intermediate container build, make final container and push to docker hub just 1 command most of the "complexities" get abstracted inside Makefile.