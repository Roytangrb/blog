---
title: "Blogging with Hugo & gh-pages"
date: 2020-10-17T22:47:38+08:00
categories: ['Web']
tags: ['Hugo', 'Github Actions']
---

Having been working as a software developer for over a year, I finally started this blog for the good habit of taking notes along the way of learning.

### Goals
This initial post will walk through the steps of: 
1. Setting up Hugo for local development and post drafting
2. Hosting your source code on Github, and your static site generated on the `gh-pages` branch
3. Use Github Actions to automate your Hugo build and deployment. Surprisingly, it's easy, fast and free!

#### Getting started
To get started, first install Hugo cli with Homebrew:
```bash
brew install hugo
```

To check if the installation succeeds, run: 
```bash
hugo version
```

#### Hugo setup
Create a new Hugo site project by running:
```bash
hugo new site [path/to/<name-of-your-site>]
```
A new directory will be created for you.

You might be using themes for you blog. You may browse the available options on the [theme list](https://themes.gohugo.io/). I personally chose [Notepadium](https://github.com/cntrump/hugo-notepadium).

Hugo themes are usually added to your project and managed as a [git submodule](https://www.atlassian.com/git/tutorials/git-submodule), which means it's not really your source code but the vendor dependency is kept track of. 

```bash
git submodule add https://github.com/cntrump/hugo-notepadium.git themes/hugo-notepadium
```
The theme is now installed under the `themes/` directory. Noted that a `.gitmodules` file is generated. Next time when you clone your project on a different computer, the theme (submodule) will not be pull by default. You need to run either
* `git clone --recursive <your-git-repo.git>` or
* `git clone <your-git-repo.git>` & `git submodule init` & `git submodule update`

Run the development server by using:
```bash
hugo server -D # -D makes the server render draft content
```

Generate you site using the build command: `hugo`. You might want to modify `.gitignore` to ignore the output directory `public/` because we could use Github Actions to automate the build-and-deploy pipeline.

#### Host things on github
Now the project is all setup, we can start writing, styling and play with the markdown porn. We can just push our local git repo to Github. Same old flow of using gh-pages, create a empty git and push the local git up.

Github has changed the default branch from `master` to `main` :). We will use `main` branch as the release branch to trigger deployment later.

Our goal is to push the generated output i.e the `public/` folder to gh-pages. From there Github will host all the html/css/js for us. We can take the simple approach by not to ignore the output from git, and push the `public/` as the git subtree to the `origin/gh-pages` branch. We will need to re-build manually and type out this push command every time:
```bash
# Deploying a subfolder to GitHub Pages
# https://gist.github.com/cobyism/4730490
git subtree push --prefix public origin gh-pages
```

This will do, but we want to automate things.

#### Using Github Actions
The CI/CD pipeline of our blog is just as simple as adding a new workflow on Github (you can find it in the `Actions` tab of you repo).

Replace the `.yml` job config as follow, you blog site will be updated on new code push to `main` branch
```yml
on:
  push:
    branches:
      - main  # Set a branch to trigger deploy

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.75.1'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```
If you need to customize the build/deploy task, find the actions' documentation [here](https://github.com/peaceiris/actions-hugo).

#### Summary
I have been studying the golang a bit recently and learned that Hugo might be the fastest static site generator. I shall have more insights if my content grows, but it seems very promising to me so far.

The Github Actions are nice, especially for people already familiar with Github. The usage and interface are pretty similar to Azure DevOps I am using for work.

The biggest inspiration from setting up this project is the use of git submodule. At work, my team uses NPM/YARN to manage different SDK (node_modules) for a react-native project. Each business unit or vendor feature is cut into a separate git. Different components are assembled as node_modules dependencies in the base app. It took me a whole day just trying to build the app, so it did for other newcomers. It causes some issues: 
* Messy dependencies (not knowing which node_modules the code is importing from), probably caused by the peer dependencies config
* Bad collaboration git flow, people have different project setup, some modify component's code in the `node_modules` folder, some use different js bundler config to maintain a flat folder structure. Different project structures actually affect how node_modules are resolved
* Yarn and NPM fail to pulling the latest commits from git repo, the app just won't build the next day if you pull somebody elses' commits

Git submodule seems to be an alternative approach to study on for a integration project with loads of parties/sub-project/vendor modules and features.

The source code for my blog could be found in this [repo](https://github.com/Roytangrb/blog/tree/main).