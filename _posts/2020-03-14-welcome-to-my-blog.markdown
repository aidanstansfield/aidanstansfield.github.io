---
layout: post
title:  "Welcome to my blog!"
date:   2020-03-14 14:26:58 +1000
---
G'day! Today marks the day that I finally became *one of those guys* who has his own blog. The main reason for making this blog is so that I can show it off to other people (emplyees, friends, etc.). In this blog, I plan on documenting various things I do related to IT that I find interesting and worth sharing. You can expect this to include some home projects on my raspberry pi, walkthroughs of [hackthebox][hackthebox]/ctf's, and generally anything else I find cool. Also, I find myself funny, so please try not to cringe to hard at what I write/take offence from anything I say.

For my first post, I figured I'd document how I made *this*, the blog itself. The process I went through to set it up, and how I made it look so damn pretty.

# How I setup my blog
## Intro to GitHub Pages

To start with, if you didn't already know, this is running on a service called [GitHub Pages][gh-pages]. Essentially, the quickstart is as follows:

1. You create a GitHub repo with the name `$USERNAME.github.io` where `$USERNAME` is your own GitHub username.

2. Next, you clone your repo.

    ```
    git clone https://github.com/$USERNAME/$USERNAME.github.io
    ```

3. Add whatever HTML you want

    ```
    echo "Hello World" > index.html
    ```

4. Push it

    ```
    git add -A
    git commit -m 'hello world'
    git push -u origin master
    ```

5. GitHub will automagically see your new commit, and in a short time, build and publish your code available for you to visit at your very own website https://$USERNAME.github.io

**How cool!**

## Intro to Jekyll 

Ok so having GitHub build and host a plain old HTML site for you is kinda cool I guess, but nothing to write home about. What *is* cool however is the combination of Jekyll and GitHub Pages. But what is Jekyll you ask?

Jekyll is a static site generator with with built-in support for GitHub pages. Jekyll takes special Markdown and HTML files, and builds out a completely static website based upon them. For more about what Jekyll is and how it works, check out their [website][jekyll].

To use Jekyll with your own website:

1. First install Ruby, Bundler and Jekyll. For Debian based systems:

    ```
    sudo apt update && apt install ruby-full && gem install bundler jekyll
    ```

2. Create your new Jekyll site inside your repositories folder

    ```
    bundle exec jekyll new /path/to/repository
    ```

3. If you want to have GitHub Pages automagically build your Jekyll site for you, open up the Gemfile and follow the instructions within the comments of that file regarding GitHub Pages.

4. I'd make sure all the dependencies are up to date with

    ```
    cd /path/to/repository && bundle update
    ```

5. Push your new code and wait for it to build.
6. Congrats, you now have a cooler looking website that runs on Jekyll. To see all the benefits of using Jekyll, such as themes, templating, syntax highlighting and more, I'd encourage you again to check out their [website][jekyll]. I'd also recommend checking out how to build & serve your site locally for development purposes, as documented [here][jekyll-local].

## Making Jekyll Sexy
If you're following along with this post, you might be wondering why my Jekyll site looks cooler than the black and white default of Jekyll. The answer is *themes*. After trolling through the interwebs for about an hour or so, I stumbled upon this theme that I really liked, called [Jekyll Dash][jekyll-dash]. I decided this would be the theme for my blog, so I went about setting it up. **This is where the fun begins.**

Now in it's current state, GitHub Pages is not capable of accepting user supplied plugins/gems to be installed as part of the build process, meaning that you cannot supply a theme (unless it's one of the 13 supported themes) as follows in the `_config.yml` file.

```yaml
theme: jekyll-dash
```

GitHub's workaround for this is to use the `remote_theme` option which allows you to use any theme available in a GitHub repository as shown.

```yaml
remote_theme: bitbrain/jekyll-dash
```

This solved a lot of my problems, however the theme didn't quite work as it did on [Bitbrain's blog][bitbrain]. So I forked the the [Jekyll Dash][jekyll-dash] repo and started customizing the theme to make it look the way it was supposed to.

This solved the majority of problems, except for the fact that you still couldn't supply your own plugins to be used as part of the build process for the site. In this case, the jekyll-dash theme used a `liquid-md5` plugin to help support Gravatar image URL's.

So after a bit of researching, I decided the best approach would be to build the site locally (that way I could use whatever plugins I pleased), and then push the result to GitHub for free hosting. Here's how I achieved this:

1. Create a new branch called 'source'. Since the repository is what GitHub terms a "user" site and not a "project" site, the only publishing source available is the master branch. This means that the code GitHub will build and publish will be taken from the master branch, so we will need another branch to store the source code for building our blog!
    ```
    git checkout -b source
    ```
2. Install [jgd][jgd]. Jekyll-Github-Deploy (jgd) is a tool that will build your Jekyll site and push it to GitHub for you.
    ```
    gem install jgd
    ```
3. Build your site from the source branch and push it to the master branch!
    ```
    bundle exec jgd --branch master --branch-from source
    ```
4. Enjoy the freedom of plugins with Jekyll!

## Summary
So, welcome to my blog! I briefly described how I set up my own blog with Jekyll and GitHub Pages, using whatever plugins/themes I wanted. Hope you enjoyed!

[hackthebox]: https://hackthebox.eu
[gh-pages]: https://pages.github.com/
[jekyll-local]: https://help.github.com/en/github/working-with-github-pages/testing-your-github-pages-site-locally-with-jekyll
[bitbrain]: https://bitbrain.github.io
[jekyll-dash]: https://github.com/bitbrain/jekyll-dash
[jekyll]: https://jekyllrb.com/
[jgd]: https://github.com/yegor256/jekyll-github-deploy
