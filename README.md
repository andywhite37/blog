# blog

This repo contains my experimental self-hosted blog source code, powered by [Hugo](https://gohugo.io/).

I am not a Go developer, but Hugo seemed like a nice tool for managing a static site without, so I'm just giving it a shot for now.

The `public/` folder is a submodule to my GitHub pages user site [https://andywhite37.github.io](https://andywhite37.github.io)
so when I run the hugo build, it dumps the static site into that folder, which I then just push to make the site live.

# Theme

~~I'm currently using the [ananke theme](https://github.com/budparr/gohugo-theme-ananke)~~

I'm currently using the [hugo-coder theme](https://themes.gohugo.io/hugo-coder/)

# First time setup

```sh
# Clone the repo
> git clone git@github.com:andywhite37/blog
> cd blog

# Themes and the public/ folder are submodules
> git submodule update --init --recursive
```

# Run local server

```
# Run he local server (with drafts included)
> hugo server -D
```

# Build

```
> hugo
```

# Build and deploy

This builds the site to `public/`, then commits and pushes the `public/`
folder (which is a submodule to my GitHub pages repo https://github.com/andywhite37/andywhite37.github.io.  Then it commits and pushes the submodule change (and any other changes) to this `blog` repo.

```sh
> ./deploy
```