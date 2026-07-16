# rustybower.com

Source for [rustybower.com](https://www.rustybower.com), a blog about security engineering, systems, and infrastructure.

Built with [Hugo](https://gohugo.io) using the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## Local development

```bash
git clone --recursive git@github.com:RustyBower/rustybower.com.git
cd rustybower.com
hugo server -D
```

## Deployment

The site runs on a K3s cluster, deployed via ArgoCD and Kustomize. The pod uses a three-container pattern:

1. **git-sync** — polls this repo every 30 seconds for new commits
2. **hugo** — rebuilds the static site when the working tree changes
3. **nginx** — serves the built output

Push to `master` and the site is live in about 30 seconds. No CI/CD pipeline, no build step outside the cluster — git-sync and Hugo handle everything inside the pod.
