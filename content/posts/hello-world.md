+++
date = '2026-05-17T21:44:48+08:00'
draft = false
title = 'Hello World'
+++

```
winget install Hugo.Hugo.Extended
hugo new site iamgqr.github.io
cd iamgqr.github.io
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
echo "theme = 'PaperMod'" >> hugo.toml
hugo new posts/hello-world.md
vi content/posts/hello-world.md
hugo server -D
```
