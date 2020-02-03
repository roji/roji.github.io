---
title: Putting commit info in your nuget packages
permalink: /commit-info-in-your-nugets
modified: 2020-02-03
comments: true
---
Here's a little-known feature of nuget: you can integrate information about your source control repo in your package's nuspec. [As the nuspec reference specifies](https://docs.microsoft.com/en-us/nuget/reference/nuspec#repository), you can specify:

* The type of source control (e.g. git)
* The URL for the repo (e.g. a github repo URL)
* The branch
* The commit ID (e.g. git SHA)

The repo URL, for example, is displayed on the nuget.org gallery to link back to the packages repo. But more importantly, including the commit SHA allows your package's users to know which commit the package was built from, and to examine that specific version if they need to.

Of course in these modern times, people don't roll their own nuspec files unless they have a good reason too: we just let `dotnet pack` generate it for us. Luckily, these four nuspec elements can be specified as simple properties in your csproj:

```xml
<PropertyGroup>
  ...
  <RepositoryType>git</RepositoryType>
  <RepositoryUrl>git://github.com/npgsql/npgsql</RepositoryUrl>
</PropertyGroup>
```

The repo type and URL are easy, since they're simple constants - but what about the branch and commit? Assuming you're using a CI solution like Github Actions, this information is typically made available via environment variables. For example, [here's the page detailing environment variables exposed by Github Actions](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/using-environment-variables): the commit and branch/tag are exposed by `GITHUB_SHA` and `GITHUB_REF` respectively.

Since you can easily access environment variables in csproj, all that's needed is the following:

```xml
<PropertyGroup>
  ...
  <RepositoryType>git</RepositoryType>
  <RepositoryUrl>git://github.com/npgsql/npgsql</RepositoryUrl>
  <RepositoryBranch>$(GITHUB_REF)</RepositoryBranch>
  <RepositoryCommit>$(GITHUB_SHA)</RepositoryCommit>
</PropertyGroup>
```

When doing `dotnet pack` on Github Actions, the resulting package's nuspec file will have an element such as the following:

```xml
<repository type="git" url="git://github.com/npgsql/efcore.pg" branch="master" commit="79f1db0d714adffe31292d96aa7681d1a2a98375" />
```

When building locally, these environment variables won't exist, and the `branch` and `commit` attributes in the nuspec will simply be omitted.
