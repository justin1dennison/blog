---
title: "GitHub Actions With Go"
date: 2020-02-05T09:54:08-05:00
draft: false
tags: 
  - automation
  - github
  - go
  - security
---


Recently, I was working on a Go project (see [GoCat and Remote Commands](/blog/posts/gocat-and-remote-commands/)) in order to utilize Go's wonderful cross platform support. However, when I think about rebuilding the binaries everytime that I make changes to the project, I just get tired. A little voice in my head says...

![Buzz telling Woody that he should automate everything](/blog/images/automate-everything.png)


Before we move forward on how I automated the build process, I will warn you that sometimes this voice can create work for you that is not meaningful. If you can accomplish a task one time and be done, then please do not set up automation. However, if you know that you are going to repeat that same task over and over again, then automation will save you some time in the long run.


With that out of the way, let's get to the fun part: **Automation!**


I have used various continuous integration platforms before such as Travis CI, but recently _GitHub Actions_ have become publicly available so I figured that now is the time to give them a try. I have been excited about GitHub actions because I don't have to remember or choose other services. Everything that I need for a project is in a centralized location. So I dove in trying to get GitHub Actions to build Go binaries for 64-bit Linux, Windows, and MacOS. 

I got a little deep in the weeds a little too fast...

![Spongbob Squarepants getting very confused and tangling his arms together.](/blog/images/confused-spongebob.gif)

I was having some issues just getting GitHub Actions to build the binaries and upload the artifacts to GitHub Releases for the project. A few different things went wrong:
1. I was trying to accomplish everything in a single build job. This seemed like it would work for me, but I ran into issues with the artifacts being 30 kilobytes in size. That should be okay, right? Well, when I built the Go project locally, each of the binaries were ~3MB in size. So I downloaded the artifacts from GitHub Releases and found that it was a text file with a warning message contained within. 
2. I didn't understand how to parameterize builds when using Actions. As a result, I tried to lean on the fact that the build environment was a Linux machine and I could script the compilation of the project with Bash. (Hey! That seemed like a good idea.) However, the build process was very opaque with errors or issues that occurred so debugging was a nightmare. Additionally, this was combined with the issue from #1.
3. Testing changes sometimes caused contention for resources to build so there would be a failure that had nothing to do with the project build setup. (I was making changes and pushing to the repository a great deal as that seemed to be the only way to test the changes I made to the build process).

I was getting frustrated...So I took a break. I thought, "Okay, I need to take a step back. Let me head to the documentation!".

Upon my inspection of the GitHub Actions documentation ([here](https://help.github.com/en/actions/automating-your-workflow-with-github-actions)), I found a couple of interesting pieces that might work to my advantage. The `jobs.strategy` and `jobs.needs` portions would allow me to parameterize and stage build steps in a job. Now, I will say that I am glossing over some exploration of these features so don't think I got this right on the first try. To see how I utilized these, let's take a look at the workflow file:

```yaml
name: Go
on: [push]
jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [darwin, linux, windows]
        arch: [amd64, 386]
    steps:
      - name: Set up Go 1.13
        uses: actions/setup-go@v1
        with:
          go-version: 1.13
        id: go

      - name: Check out code into the Go module directory
        uses: actions/checkout@v2

      - name: Get dependencies
        run: |
          go get -v -t -d ./...
          if [ -f Gopkg.toml ]; then
              curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
              dep ensure
          fi

      - name: Build
        run: go build -v -o ${{ matrix.platform }}.${{ matrix.arch }}.goCat .

      - name: Upload Artifacts
        uses: actions/upload-artifact@master
        with:
          name: binaries
          path: ${{ matrix.platform }}.${{ matrix.arch }}.goCat

  release:
    name: Release binaries
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download the results from builds
        uses: actions/download-artifact@v1
        with:
          name: binaries
      - name: Upload the release
        uses: marvinpinto/action-automatic-releases@latest
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          automatic_release_tag: latest
          files: |
            binaries/*
```

There is a lot going on in the workflow file above. However, I want to focus on the keys points that allowed me to automate the Go build. First, if you look at the `strategy` section, then you will see the following section:

```yaml
strategy:
    matrix:
        platform: [darwin, linux, windows]
        arch: [amd64, 386]
```
You can use this to set up templated tasks. For instance, one of the build steps that I use is

```
go build -v -o ${{ matrix.platform }}.${{ matrix.arch }}.goCat .
```
which repeats for every value of **platform** and **arch** for a total of six runs. This was way easier than writing a custom script to build the project. Furthermore, you can use GitHub Action to share artifacts between jobs. As a result, I would save the artifacts after they were compiled and move to a release job.

I separated these jobs so I could understand where a failure occurred. I am glad that I made that decisions, but that separation did lead to an issue where the release would run **before** the build portion. However, further consultation of the documentation led me to `needs`. Using the following:
```yaml
...
needs: build
...
```
can enforce run order of the jobs. After adding that single line, then VOILA! - automated rebuilds upon every push to master. 

Now if you look at this workflow a little closer, there are some shortcomings and/or things that require further consideration. 
1. I am not using versioning or a process for when to rebuild and release.
2. I could think about the event that triggers a rebuild as to not churn up GitHub resources.
3. I am using some marketplace custom actions (think of it as a store for build steps), that can have bugs (the small binary size was one such bug.).


Overall, GitHub Actions are a promising replacement for some CI setups. Your mileage may vary depending on the complexity of your current build process, but definitely check it out.










