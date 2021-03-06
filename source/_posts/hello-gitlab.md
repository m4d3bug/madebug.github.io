---
title: Hello GitLab
categories:
- "DevOps "
tags:
- "Git " 
- "CI/CD " 
- "Hexo "
- "Ubuntu "
- "Linux "
- "Nodejs " 
date: 2019-08-08 23:59:59
---

*In this post, I am going to markdown how I build up this blog with CI/CD and custom my own domain in [GitLab](https://www.gitlab.com) .*

## *Configure blog environment in local*

### *Install the package*

``` bash
~]# apt update -y && apt install -y nodejs npm git
```

### *Confirm the version of tools*

``` bash
~]# nodejs -v
v8.10.0
~]# npm -v
3.5.2
~]# git --version
git version 2.17.1
```

### *Install the hexo*

``` bash
~]# npm install hexo -g
```

### *Confirm the version of hexo*

``` bash
~]# hexo -v
hexo-cli: 2.0.0
os: Linux 4.15.0-20-generic linux x64
http_parser: 2.7.1
node: 8.10.0
v8: 6.2.414.50
uv: 1.18.0
zlib: 1.2.11
ares: 1.14.0
modules: 57
nghttp2: 1.30.0
openssl: 1.0.2n
icu: 60.2
unicode: 10.0
cldr: 32.0.1
tz: 2017c
```

### *Initialize the directory*

``` bash
~]# mkdir m4d3bug.gitlab.io
~]# cd m4d3bug.gitlab.io/
~/m4d3bug.gitlab.io# hexo init
npm WARN optional Skipping failed optional dependency /chokidar/fsevents:
npm WARN notsup Not compatible with your operating system or architecture: fsevents@1.2.9
INFO  Start blogging with Hexo!
```

## *Configure git repository*

### *Create the remote repository*

*Please fill up your repository name such as: [ custom name ]+gitlab.io.*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200410191701.png?raw=true)

### *Create the local repository*

``` bash
~/m4d3bug.gitlab.io# git init
Initialized empty Git repository in /root/m4d3bug.gitlab.io/.git/
~/m4d3bug.gitlab.io# git remote add origin git@gitlab.com:m4d3bug/m4d3bug.gitlab.io.git
~/m4d3bug.gitlab.io# git remote -v
origin  git@gitlab.com:m4d3bug/m4d3bug.gitlab.io.git (fetch)
origin  git@gitlab.com:m4d3bug/m4d3bug.gitlab.io.git (push)
```

## *Configure the access right in the GitLab*

### *Generate local SSH keys and public keys*

``` bash
~/m4d3bug.gitlab.io# ssh-keygen -t rsa -b 4096 -C "m4d3bug@ubuntu" -N ""
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:dGucfhOAOXbHYm@@@@@@@@@@@@@@@/L@@@@@@@@@@@@ m4d3bug@ubuntu
The key's randomart image is:
+---[RSA 4096]----+
|          .++.   |
|         + .  .  |
|        B * oo . |
|       =E@ *= O  |
|       .S.=..* * |
|       o.oo o.=  |
|         ..+oB   |
|          o.=.o  |
|        .o o..   |
+----[SHA256]-----+
~/m4d3bug.gitlab.io# cat /root/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1y.../iiF@@@@@@@@@@@@@@@@@@@@@== m4d3bug@ubuntu
```

### *Upload the public key*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20200410192701.png?raw=true)

### *Update the policy of yourself*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/9NeG8lWjYHadt4b.jpg?raw=true)

####  *Delete origin rule and setup like this.*

![](https://img.madebug.net/m4d3bug/images-of-website/master//blog/jL7n1h3KBOywvZg.png?raw=true)

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/UGSMXCbTFYlA1yh.png?raw=true)

## *Start up the first commit*

``` nohighlight
~/m4d3bug.gitlab.io# git checkout -b beta
Switched to a new branch 'beta'
~/m4d3bug.gitlab.io# git config --global user.email "m4d3bug@gmail.com"
~/m4d3bug.gitlab.io# git config --global user.name "m4d3bug"
~/m4d3bug.gitlab.io# git add .
~/m4d3bug.gitlab.io# git commit -m "Init Commit"
~/m4d3bug.gitlab.io# ssh -T git@gitlab.com
~/m4d3bug.gitlab.io# git push --set-upstream origin beta
```

*Now you can see that the remote repository has created a new branch called beta and includes the above files.*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/OvHGo1j3MutW7rR.png?raw=true)

## *Configure the pipeline*

### *Create the .gitlab-ci.yml in local*

``` yaml
image: node:6.10.2

cache:
    paths:
        - node_modules/

variables:
    GIT_SUBMODULE_STRATEGY: recursive

before_script:
    - npm install hexo-cli -g
    - npm install

pages:
    script: 
        - hexo generate
    artifacts:
        paths:
            - public
    only:
#       - master
        - beta
```

### *Confirm the status of local repository*

``` bash
~/m4d3bug.gitlab.io# git status
On branch beta
Your branch is up to date with 'origin/beta'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        .gitlab-ci.yml

nothing added to commit but untracked files present (use "git add" to track)
```

### *Push the change to the remote repository*

``` nohighlight
~/m4d3bug.gitlab.io# git add .
~/m4d3bug.gitlab.io# git commit -m "Init Commit"
~/m4d3bug.gitlab.io# git push --set-upstream origin beta
```

### *Confirm the pipelines is working*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/U9xHfalLTC3WpA1.png?raw=true)

*After that, you can get the assigned domain name in Settings >> Pages .*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012224018.png?raw=true)

## *Custom my own domain*

### *GitLab generates domain verification text*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012213613.png?raw=true)

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012213701.png?raw=true)

### *Domain name resolution is set by cloudflare*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012213810.png?raw=true)

### *Make sure the https working on custom domain*

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012213854.png?raw=true)

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20191012213928.png?raw=true)



## *Done*

*Now the custom domain name and CI/CD are in effect, and a clean master branch is kept for rollback. When everything is good with the commit, you can choose merge beta to the master branch for backup, by executing the following command.*

``` nohighlight
~/m4d3bug.gitlab.io# git checkout master
~/m4d3bug.gitlab.io# git merge beta --allow-unrelated-histories
~/m4d3bug.gitlab.io# git add .
~/m4d3bug.gitlab.io# git commit -m "Merge Beta"
~/m4d3bug.gitlab.io# git push --set-upstream origin master
```





