# Blog gen src code directory

## Requirements
* `git`
* `hugo version`: v0.76.5/extended

## Development
* Run `brew install hugo` & `hugo version`
* Run `git submodule init` & `git submodule update` to fetch theme dependencies recursively
  * or run `git clone <this-repo.git> -recursive`
* Run dev server: `hugo server -D`
* New post: `hugo new posts/<post-title>.md`

## Deployment
* [Github Actions - Hugo build & deploy](https://github.com/peaceiris/actions-hugo)

## References
* [Hugo setup](https://levelup.gitconnected.com/build-a-personal-website-with-github-pages-and-hugo-6c68592204c7)
* [Notepadium Theme](https://github.com/cntrump/hugo-notepadium)
* [Git submodules](https://www.atlassian.com/git/tutorials/git-submodule)
* [Publish subdir to gh-pages](https://gist.github.com/cobyism/4730490)
* [SEO tags](https://www.skcript.com/svr/perfect-seo-meta-tags-with-hugo/)