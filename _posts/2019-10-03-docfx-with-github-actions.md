---
title: Conceptual and API documentation with Docfx, Github Actions and Github Pages
permalink: /docfx-with-github-actions
modified: 2019-10-03
comments: true
---
A good software project is (among other things!) measured by the quality of its documentation, but setting up a good documentation workflow isn't trivial. There are generally two kinds of documentation: conceptual articles, which are written manually (e.g. in markdown) and API documentation which is generated directly from source code. Docfx is a great tool which knows how to generate a single, seamless site from these two documentation types, but reaching doc nirvana is still hard:

* The whole process should be fully automated: devs shouldn't ever need to run docfx manually. We're better than that. Let's call this a continuous documentation pipeline.
* Conceptual docs are sometimes managed in a separate repo, raising the question of how to bring multiple docs together.
* The Npgsql case is even more complex: the same site has documentation for two projects (both the base Npgsql driver and the EF Core provider).

The post will describe the new documentation pipeline used by Npgsql to solve these challenges, using [docfx](https://dotnet.github.io/docfx/), [Github Actions](https://help.github.com/en/categories/automating-your-workflow-with-github-actions) for automation and [Github Pages](https://pages.github.com/) for hosting. It assumes you're (somewhat) familiar with docfx, and won't go into the details of configuring it.

## Automating Docfx with Github Actions

Let's concentrate on conceptual documentation for now. For people hosting static (or Jekyll) sites on Github Pages, life is simple: edit a file and push, and your changes are automatically live. When a processing system such as docfx is used, we have to run docfx, and the resulting HTML site needs to be hosted somewhere. We can use Github Actions to do this for us.

Create a file called `.github/workflows/build-documentation.yml` in a repo containing some articles and a docfx.json:


```yaml
name: Build Documentation

on:
  push:
    branches:
      - master

jobs:
  build:

    runs-on: ubuntu-18.04

    steps:
    - name: Checkout repo
      uses: actions/checkout@v1
      with:
        path: docs
        fetch-depth: 1

    - name: Get mono
      run: |
        apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
        echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
        sudo apt-get update
        sudo apt-get install mono-complete --yes

    - name: Get docfx
      run: |
        curl -L https://github.com/dotnet/docfx/releases/latest/download/docfx.zip -o docfx.zip
        unzip -d .docfx docfx.zip

    - name: Build docs
      run:  mono .docfx/docfx.exe
```

Let's go over the above. We've created a workflow called "Build Documentation" that will run every time something is pushed to our repo's master branch. It first clones our repo into a directory called docs, and to save time we fetch only 1 commit deep (who needs all that history). Now, since I'm a diehard Linux dude, we'll be running on Ubuntu; this unfortunately means that we need to install mono, since docfx only runs on .NET Framework (guys, it's 2019 and .NET Core 3.0 has just been released...). I won't go into the technicalities of this step, and if you prefer Windows you can skip it entirely.

Once mono is installed, we get the latest version of docfx by fetching it from their Github releases page, and unzip it into some directory. At this point we're ready to go, and can run docfx - hurray! Could it be this simple?

## Repos, repos...

Well, uh, no... What are we going to do with all those HTML files that docfx generated? They need to be hosted somewhere. If you have some external hosting service, at this point you'd pack the outputs into a ZIP and send it off somewhere. But if you use Github Pages for your hosting, you may be tempted to host your site in the same repo which contains the sources. While this may seem like a good idea, it probably isn't: to actually go live, you need to push a new commit containing these HTMLs; creating a new commit in your sources repo would mean you have to pull it the next time you want to make a change. But in any case, who wants a source repo to contain generated HTML artifacts - that's like committing your compiled objects alongside your sources, yuck.

So we'll open a new repo whose sole purpose is to host our static HTML files: this will be our publicly-visible repo. Our workflow will clone that repo, make sure that docfx generates its outputs into it's directory, and finally commit and push the changes to it. Let's add this additional fragment after our repo's checkout:

```yaml
    - name: Checkout live docs repo
      uses: actions/checkout@v1
      with:
        repository: npgsql/livedocs
        ref: master
        fetch-depth: 1
        path: docs/live
    - name: Clear live docs repo
      run: rm -rf live/*
```

This time we need to specify which repo we want to clone, since it isn't the repo where the workflow is running. We also specify to clone it into a `live` directory inside our sources repo; when docfx runs, it will automatically generate HTMLs into that directory. Once that's done, all that's left is to commit and push those changes to the live repo - but unfortunately that's a bit complicated.

To push changes to another repo, we're going to have to have an access token with the proper permission, so let's head over to the site and generate one, with `repo` permissions, by following [these instructions](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line). Now, I know we want to go fast, but we absolutely *cannot* insert that access token inside our workflow YAML: this is a public file, and putting our token there would give the world write access to our repo - not cool (don't do this even for private repos!). Fortunately, Github Actions has a [secrets management feature](https://help.github.com/en/articles/virtual-environments-for-github-actions#creating-and-using-secrets-encrypted-variables), which allows you to store your access token and reference it safely from your YAML. Follow the instructions in that page to create a token called DOC_MGMT_TOKEN, and insert the following fragment at the bottom of your workflow:


```yaml
    - name: Commit and push
      run: |
        cd live
        git config --global user.email "noreply@npgsql.org"
        git config --global user.name "Automated System"
        git add .
        git commit -m "Automated update" --author $GITHUB_ACTOR
        header=$(echo -n "ad-m:${{ secrets.DOC_MGMT_TOKEN }}" | base64)
        git -c http.extraheader="AUTHORIZATION: basic $header" push origin HEAD:master
```

We unfortuately have to jump through some hoops - this should ideally be simpler. We:

* Enter the live repo's directory, where our HTMLs have been generated
* Configure our name and email with git, as these will appear in the commit we're about to create
* Add all files
* Create the commit
* Git push the commit to the live doc, doing some magic to include our access token in the HTTP header for proper authentication

... and we have a fully-working, automated documentation pipeline - just push any changes to see it appearing live! Now we're done, right?

## API documentation

I know... I promised we'd also do API documentation here. It's not so hard after what we've already been through. After [configuring your docfx.json](https://dotnet.github.io/docfx/tutorial/walkthrough/walkthrough_create_a_docfx_project_2.html) appropriately, if your (conceptual) doc repo is separate from your actual project(s), you will simply need to add workflow step to clone it into a directory where docfx will look for it:

```yaml
    - name: Checkout Npgsql
      uses: actions/checkout@v1
      with:
        repository: npgsql/npgsql
        ref: master
        fetch-depth: 1
        path: docs/Npgsql
```

Note that Npgsql follows [Gitflow](https://datasift.github.io/gitflow/IntroducingGitFlow.html), which means that the latest released version can always be found in the master branch - so that's where we generate API docs from. Your git workflow may be different, adjust accordingly.

At this point, a doc rebuild is triggered whenever something is pushed to our *conceptual* repo, which is great, but we also want our project repo to trigger a rebuild! So we add another trigger at the beginning of our doc repo's workflow file:

```yaml
on:
 repository_dispatch:
 push:
    branches:
      - master
```

The added [*repository dispatch*](https://help.github.com/en/articles/events-that-trigger-workflows#external-events-repository_dispatch) is basically an event that can be triggered externally via a simple HTTP POST request. All that's left is to drop the following workflow in our project repo, under `.github/workflows/trigger-doc-build.yml`:

```yaml
name: Trigger Documentation Build

on:
  push:
    branches:
      - master

jobs:
  build:

    runs-on: ubuntu-18.04

    steps:
    - name: Trigger documentation build
      run: |
        curl -X POST \
             -H "Authorization: token ${{ secrets.DOC_MGMT_TOKEN }}" \
             -H "Accept: application/vnd.github.everest-preview+json" \
             -H "Content-Type: application/json" \
             --data '{ "event_type": "Npgsql push to master" }' \
             https://api.github.com/repos/npgsql/doc/dispatches
```

Note that we also need to use an access token here as well, and to configure it on our *project* repo, since that is where the workflow runs. Once this is done, every push to your repo's master branch will result in a doc rebuild (Npgsql even has two repos with this trigger to the same site). 

## Nirvana

Once this is all properly set up, you hopefully never have to think about syncing docs ever again...! If you think the above is useful (or hate it), please drop me a comment below - any improvement suggestions would be welcome as well!

Oh, and here are the full files so you can see it all put together. Feel free to wander around [the doc repo](https://github.com/npgsql/doc) or [the Npgsql project repo](https://github.com/npgsql/npgsql) to see how it all fits together.

* [build-documentation.yml](/assets/2019-10-03-docfx-with-github-actions/build-documentation.yml)
* [trigger-doc-build.yml](/assets/2019-10-03-docfx-with-github-actions/trigger-doc-build.yml)
* [docfx.json](/assets/2019-10-03-docfx-with-github-actions/docfx.json) (in case it floats your boat)
