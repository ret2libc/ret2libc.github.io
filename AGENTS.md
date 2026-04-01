# AGENTS.md

This is a Jekyll-based static site built for GitHub Pages.

## Prerequisites

- [rbenv](https://github.com/rbenv/rbenv) with [ruby-build](https://github.com/rbenv/ruby-build)
- Bundler

## Setup

### 1. Install Ruby via rbenv

```bash
rbenv install 3.2.3
rbenv local 3.2.3
```

This creates a `.ruby-version` file pinning the project to Ruby 3.2.3.

### 2. Install Bundler

```bash
gem install bundler
rbenv rehash
```

### 3. Install dependencies

```bash
bundle install
```

## Running the development server

```bash
bundle exec jekyll serve
```

The site will be available at `http://127.0.0.1:4000/`.

To enable live reload:

```bash
bundle exec jekyll serve --livereload
```

## Project structure

```
.
├── _config.yml     # Jekyll site configuration
├── _layouts/       # HTML templates
├── _includes/      # Reusable HTML partials
├── _posts/         # Blog posts (Markdown)
├── _sass/          # Sass partials
├── _site/          # Generated output (not committed)
├── images/         # Static images
├── static/         # Other static assets
├── style.scss      # Main stylesheet entry point
├── index.html      # Homepage
└── about.md        # About page
```

## Deploying

Push to the `master` branch. GitHub Pages rebuilds the site automatically.
