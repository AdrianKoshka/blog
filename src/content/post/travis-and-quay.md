---
title: "Using Travis to push container images to Quay.io"
date: 2019-01-24T20:02:00Z
draft: true
---

## Preface

This blog post assumes:

- The user is familiar with:
    - [Travis-ci](https://travis-ci.org)
    - [Docker](https://www.docker.com/) (or [podman](https://podman.io)/[buildah](https://buildah.io))
    - [Git](https://git-scm.com/)
    - [Github](https://github.com)
- The user has an account with:
    - [Travis-ci](https://travis-ci.org)
    - [Github](https://github.com)
    - [Quay](https://quay.io)


For those unfamiliar with the above, here are some [resources](#resources)

# Introduction

I prefer using [Quay.io](https://quay.io) as my container registry instead of
[Dockerhub](https://hub.docker.com), and as such, I had to learn how setup
[Travis](https://travis-ci.org) to push to [quay.io](https://quay.io). This is
something I hadn't done before, as I've only recently started making my own
container images.

### Step One: fork the tutorial repo

I've setup a repo on github that I'll be using as a guide for this tutorial, it
can be found [here](https://github.com/AdrianKoshka/travis-quay-tutorial). All
you need to do is press the `Fork` button:

![fork button](/blog/imgs/fork-button.png)

This will make a copy of my repo for you to use with your account. The next step
is to clone this repo:

`git clone git@github.com:AdrianKoshka/travis-quay-tutorial.git`

You can find the URL to clone from when you press the `Clone or download` button.

![clone or download button](/blog/imgs/clone-button-quay-travis-tut.png)

### Step Two: Initial repo setup

You'll need to edit the `.travis.yml` and `docker_push` files in your repo,
replacing `[yourusername]` with the username of your quay account. 

<script src="https://gist.github.com/AdrianKoshka/52691ea092f24997488b0bcdefd6d0b5.js"></script>

<script src="https://gist.github.com/AdrianKoshka/ee32d9157fd941f98c4eb60afac919ff.js"></script>

### Step Three: Creating the quay repository

When signed into quay, in the upper right-hand corner, you'll see a
`+ Create New Repository` button. Click the button,

![quay plus button](/blog/imgs/quay-plus-button.png)

You'll want to select `Public` instead of `Private`, and enter the repository name.
Then click `Create Public Repository`. You've now created the empty repository
we'll be pushing our container image to later.

![quay new repo](/blog/imgs/quay-new-repo.png)

### Step Four: Creating the "Robot Account"

Robot accounts are "accounts" in quay.io under your user used for automated tasks.

Again, in the upper right-hand corner, you'll see the `+` button, click it, then
click `New Robot Account`.

![New Robot Account](/blog/imgs/new-robot-account.png)

You'll need to enter a name and description for the bot:

![Create Robot Account Step 1](/blog/imgs/create-robot-step-one.png)

Press `Create Robot Account`, now you'll have to give the robot write
permissions to the repo you just created:

![Give Robot privileges](/blog/imgs/robot-give-privs.png)

Then press `Add Permissions`. You'll be taken to your robot account settings
page now.

![Robot Settings Page](/blog/imgs/robot-account-settings.png)

Click the username of your newly created bot, it should bring up a window
with your bots username and password.

![Bot username and password](/blog/imgs/bot-creds.png)

We'll need these creds later.

### Step Five: Adding variables to your repo on travis

On Travis, go to your settings and click the `sync account` button. In the list
of repositories should be your `travis-quay-tutorial` repo you forked earlier,
activate it, and click on the `settings` button.

![Travis Settings](/blog/imgs/travis-settings.png)

Scroll down to the section named `Environment Variables`, and add a variable
name `QUAY_BOT_USERNAME`, put the username of your robot account that you saw
in the previous section, you should be able to copy-paste it.

![Add QUAY_BOT_USERNAME](/blog/imgs/travis-env-var-username.png)

Press the `Add` button. Add another variable called `QUAY_BOT_PASSWORD` and
enter your bots password.

Your `Environment Variables` section should look like this now:

![Travis Environmental Variables After](/blog/imgs/travis-env-var-after.png)

### Step Six: Committing the changes from step two

Now that you've set everything else up, you can commit your changes to
`.travis.yml` and `docker_push`, then push your changes to your fork. This
should trigger a build with travis, which will build the container, and then
push it to the quay repository.

## Resources

- [Travis Core Concepts for Beginners](https://docs.travis-ci.com/user/for-beginners)
- [Docker: What is a Container](https://www.docker.com/resources/what-container)
- [Docker: Get Started with Docker](https://www.docker.com/get-started)
- [Podman: Basic Setup and Use of Podman](https://github.com/containers/libpod/blob/master/docs/tutorials/podman_tutorial.md)
- [Buildah: Tutorials](https://github.com/containers/buildah/tree/master/docs/tutorials)
- [Github Guides](https://guides.github.com/)