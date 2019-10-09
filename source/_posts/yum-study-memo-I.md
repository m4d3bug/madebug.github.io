---
title: Yum Study Memo I
date: 2019-09-14 12:21:48
mathjax: true
copyright: true
comment: true
categories:
- "Ops "
tags:
- "RHEL 7 "
- "Linux "
- "Yum "

---

## 1. [RepoCreate](http://yum.baseurl.org/wiki/RepoCreate.html) -- How to setup your own repositories

***Sometimes you'll find you need to be able to collect a bunch of rpm packages you have together in one place and you want to make them available to your systems running yum. It is pretty easy to do.***

## What is yum

### *A package repository used by yum is simply a directory with one or more RPMs plus some "meta information" used by yum to be able to easily access information (dependencies, file lists, etc.) for the RPMs. yum can then to access this directory over ftp/http or a file URI (including over NFS).*（使用yum的包倉庫只是簡單的目錄帶有多個RPMs plus標有meta information來使得yum可以簡單地訪問依賴，文件列表等RPMs的信息，並且支持訪問ftp/http和文件url甚至NFS）

## Steps

- ### *1. Collect the packages together in one directory. You can make as many sub-directories as you want, but there needs to be a top level directory where they all live. That's where we're going to form our repository.*（收集包到一起，可用子目錄但需要同一根目錄。）

- ### *2. Yum uses a digest of the information stored in each RPM to do its work. This information is created using the 'createrepo' program. If you don't have createrepo installed you can install it with:*（yum使用存在每個RPM中存儲的摘要信息，該信息是通過createrepo生成，你需要createrepo。）

  ```nohighlight
  # yum install createrepo
  ```

  ### *If you are generating your repository on a machine that doesn't use RPMs, you can download createrepo from http://createrepo.baseurl.org/ and build/install it manually.*（甚至支持非RPMs的管理。）

  ### *Once you have createrepo installed you need to run it. It only requires one argument which is the directory in which you would like to generate the repository data. So if the packages directory we made in step 1 is in /srv/my/repo then you would run:*（如果已經安裝ok則直接用以下命令生成repodata。）

  ```nohighlight
  # createrepo /srv/my/repo
  ```

  ### You should see a lot of things fly by but it should finish without an error. In the end you should have a directory named /srv/my/repo/repodata with at least 4 files in it. Maybe more.(至少四個文件生成。)

- ### *3. To make this repository known to yum you need to add a .repo file to your yum configuration. On the systems where you want to use this repo you need to make a new file in /etc/yum.repos.d/. The file can be named anything but the extension on the file has to be .repo. Let's call this one 'myrepo.repo'.*（需要新建.repo後綴文件。）

  ### *In the file you just need to include the following:*（僅需包括以下內容。）

  ```nohighlight
  [myrepo]
  name = This is my repo
  baseurl = url://to/get/to/srv/my/repo/
  ```

  ### *That's all you need in that file. The 'baseurl' line is the path that machine uses to get to the repository. If the machine has direct access to it or mounts it as a filesystem you can use a baseurl line like:*（僅需修改baseurl為你倉庫的路徑。）

  ```nohighlight
  baseurl = file:///srv/my/repo/
  ```

  ### *If you access the file via an http or https server you would use something like:*（http/https如下。）

  ```nohighlight
  baseurl = http://servername/my/repo
  ```

  ### *More details about client-side repo configuration can be found in the yum.conf man page.*	（更多詳情yum.conf。）

- ### *4. Now, every time you modify, remove or add a new RPM package to /srv/my/repo you need to recreate the repository metadata. You do that by running createrepo the same way you did in step 2.* （有任何修改、移除或新增RPM包都要recreate如同第二步一樣。）

## Recommended options

### *While running plain createrepo on it's own will work, and is generally the most compatible, it is recommended that you add extra options if you know you've only got newer yum clients.*（通常只需createrepo，但僅當你擁有更新的yum client推薦createrepo增加以下選項。）

- #### --database, *this creates the .sqlite databases on the server saving time on every client. There is no downside to doing this, as older versions of yum will just ignore the .sqlite files and get the .xml files.*（會在服務器生成.sqlite數據庫為client加速，舊的yum不會理會該文件。）

- #### --unique-md-filenames, *this names all the metadata files be uniquely. This is most useful if you are using mirrors. Versions of yum before 3.2.10 will not clean up correctly.*（這會令所有文件唯一命名，包含checksum在內，幫助http緩存尤其是使用鏡像站。但是3.2.10版本前yum客戶端將不會奏效。）

- #### --changelog-limit CHANGELOG_LIMIT, *limit changelog entries to save on download time.*（限制日誌修改條目以加速。）

- #### --deltas --oldpackagedirs, *this creates delta information, to save on download time. Clients must have yum-presto installed to take advantage of this data.*（生成增量rpms和增量metadata，並且指定舊包的路徑，來加快客戶端下載，同時客戶端需要有yum-presto方能使用該數據。）

## More Advanced options

- ### createrepo --update: *Sometimes you have a lot of packages in your repsitory and regenerating the meta data for each package when only a few packages have been added or changed is just too time consuming. This is where --update comes in handy. You run createrepo just like you did before but you pass the --update flag to it. Like this:*（為新增的包生成metadata的折中方法，直接通過update傳遞。）

  ```nohighlight
  # createrepo --update /srv/my/repo
  ```

- ### *createrepo -x package_file_name: Suppose you have a few packages in your repository directory but you really don't want the unsuspecting world to see them. You can exclude packages easily with createrepo:* （那麼反過來也可以讓某些包不包含進去。）

  ```nohighlight
  # createrepo -x filename -x filename2 -x filename* /srv/my/repo
  ```

- ### *You can sign a repository after it has been created by running:*

  ```nohighlight
  # gpg --detach-sign --armor repodata/repomd.xml 
  ```

  ### *This will create a repomd.xml.asc file in the repodata directory which for modern versions of yum this will allow yum to verify that the repository metadata came from the owner of the gpg key.*(完整性公鑰校驗)

- ### *You can change the internal checksum used.*（或者改用內部checksum為校驗值。）

  ```nohighlight
  # createrepo --checksum /srv/my/repo
  ```

  ### *This should only be needed if you are creating the repodata on a machine with a newer version of python than on the clients (or one with python-hashlib installed).*（當使用了更高版本的python或者pythonhashlib時會被需要。）
