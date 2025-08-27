# install hugo
see [hugo installation](https://gohugo.io/installation/)
```bash
# mac
brew install hugo
```
# add theme
```bash
git submodule add https://github.com/pseudoyu/pure themes/PaperMod
git submodule update --init --recursive
git submodule update --remote
```
# create new site
only need to run it once
```bash
hugo new site xxxx
```
# create new post
```bash
hugo new posts/xxx.md
```
