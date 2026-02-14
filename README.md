# rustybower.com

Source for [rustybower.com](https://www.rustybower.com), a blog about security engineering, systems, and infrastructure.

Built with [Hugo](https://gohugo.io) using the [Stack](https://github.com/CaiJimmy/hugo-theme-stack) theme.

## Local development

```bash
git clone --recursive git@github.com:RustyBower/rustybower.com.git
cd rustybower.com
hugo server -D
```

## Deployment

Pushes to `master` are automatically picked up by a [git-sync](https://github.com/kubernetes/git-sync) sidecar running in Kubernetes, which rebuilds the site with Hugo and serves it via nginx.
