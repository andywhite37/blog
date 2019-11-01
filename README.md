# blog

This repo contains my experimental self-hosted blog source code, powered by [Hugo](https://gohugo.io/).

I am not a Go developer, but Hugo seemed like a nice tool for managing a static site without, so I'm just giving it a shot for now.

The `public/` folder is a submodule to my GitHub pages user site [https://andywhite37.github.io](https://andywhite37.github.io)
so when I run the hugo build, it dumps the static site into that folder, which I then just push to make the site live.

# Theme

I'm currently using the [Ananke theme](https://github.com/budparr/gohugo-theme-ananke)

# Development

## Run local server

```
> hugo server -D
```

## Build to public/

```
> hugo -D
```