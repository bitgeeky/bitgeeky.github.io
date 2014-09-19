---
layout: default
title: Git and GitHub workflow
comments: true
---
{% highlight bash %}
Workflow for working on GIT and Github

Set Git Env variables:
git config --global user.name "Full Name"
git config --global user.email "user@example.com"
git config --global core.editor vim
git config --global merge.tool vimdiff

Proxy Settings:
git config --global http.proxy http://proxyuser:proxypwd@proxy.server.com:8080
git config --global https.proxy https://proxyuser:proxypwd@proxy.server.com:8080

1) Fork the orgs repo in Github

2) Clone the repo from your Github Account
git clone <clone url of repo in your github account> 

3) git remote add upstream <clone url for orgs repo on github >

4) origin is our repo(the url to repo in our github account) by default 
check by : git remote show origin

5) we have a master branch by default
check by : git branch

6) create a new working branch ( do not work in master branch )
git checkout -b working
-make changes on this branch

7) update master branch everytime you start working
(since the org code might have changed while you were working on it)
git checkout master
git pull --rebase upstream master

8) rebase the working branch to master branch(to apply you changes on latest code)
git checkout working
git rebase master

// not necessary for first time users
9) squash the commits // if you have more than one 
// commits combine them into single commit
git rebase -i HEAD~x 

11) And for publishing the code 
push code to working branch
git push origin working 

12) Send a pull request (change the default master branch to working)
branch from where u want the code to be pulled 
{% endhighlight %}
