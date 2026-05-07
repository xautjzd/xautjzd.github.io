# New Blog based on Hugo

Personal blog built with Hugo static site generator.

## Create new post

```bash
hugo new posts/<post-name>.md
```

## Run in local environment

```bash
hugo server -D
```

## Deploy

This blog is automatically deployed to GitHub Pages via GitHub Actions.

### Manual Deployment

```bash
# Build the site
hugo --minify

# The site will be generated in the public/ directory
# Push to main branch to trigger automatic deployment
```

### Prerequisites

- Hugo (Extended version recommended)
- Git
