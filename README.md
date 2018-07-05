hatenablog
=====

## Description

My blog entries.

## Installation

### blogsync

```shell
$ go get github.com/motemen/blogsync
```

see [motemen/blogsync](https://github.com/motemen/blogsync)

### clone repository

```shell
$ git clone git@github.com:tukaelu/hatenablog.git
```

## Configuration

Create blogsync configuration yaml file at `~/.config/blogsync/config.yaml`.

```yaml
tukaelu.hatenablog.jp:
  username: tukaelu
  password: <API KEY>
default:
  local_root: <PATH TO LOCAL REPO>
```

