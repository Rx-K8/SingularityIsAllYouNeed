# SingularityIsAllYouNeed

🚧 作成中

⚠️ 動かない可能性大

## Create SIF image

```sh
$ singularity build --fakeroot --build-arg python_version="3.11.9" miyalab.sif CreateSIFImage.def
```

## Create container

```sh
$ singularity build --sandbox miyalab miyalab.sif
```

## Run a shell within a container

```sh
$ singularity shell --fakeroot --shell /bin/bash -w --no-home miyalab
```
