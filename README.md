# Aidan's Blog
Here you can find the source code for building [my blog](https://aidanstansfield.github.io/), as well as the source for the blog itself under the master branch
## Prerequisites
Before you can start building, you need to install the necessary dependencies. Just run `bundle update`
## Building
I use [jgd](https://github.com/yegor256/jekyll-github-deploy) to automagically build and deploy my pages to the master branch!
Note, since this is a "User site", the only available publishing source is the master branch.

To build, change into the directory (`cd aidanstansfield.github.io`) and run

`bundle exec jgd --branch master --branch-from source`
