# Jonathan Bouzekri's blog

A repository to host my blog built with Jekyll.

## Installation

```shell
npm install
```

## Development

```shell
docker run --rm -p 4000:4000 -p 35729:35729 --volume="$PWD:/srv/jekyll" -it jekyll/jekyll jekyll serve
```

## Deployment

```shell
rm -rf _site
docker run --rm -e JEKYLL_ENV="production" --volume="$PWD:/srv/jekyll" -it jekyll/jekyll jekyll build
```
