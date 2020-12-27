---
title: "Building My Blog on Windows"
date: 2019-02-03T15:38:44-05:00
---

## Part One: Creating a container

On Linux, I build my blog via a [container](https://quay.io/repository/adrianlucrececeleste/alpine-hugo),
I felt like doing the same on windows. This means I'd have to create a container
image for building my blog. Until now, I hadn't experimented with windows
containers, I chose the [nanoserver](https://hub.docker.com/r/microsoft/nanoserver)
image as the base for my container. After doing some experimentation, I wrote
[this](https://github.com/AdrianKoshka/windows-hugo/blob/master/Dockerfile) dockerfile:


```Dockerfile
FROM microsoft/nanoserver
LABEL maintainer="Adrian Lucrèce Céleste <adrianlucrececeleste@airmail.cc>"

RUN powershell Invoke-WebRequest -Uri https://github.com/gohugoio/hugo/releases/download/v0.54.0/hugo_0.54.0_Windows-64bit.zip -Outfile hugo.zip
RUN powershell Expand-Archive C:\\hugo.zip -DestinationPath C:\\hugo

WORKDIR C:\\workspace
CMD [ "--help" ]
ENTRYPOINT [ "C:\\hugo\\hugo.exe" ]
```

For those unfamiliar with docker, this file can be easily broken down
line-by-line.

`FROM microsoft/nanoserver`

This line tells docker what image we want to base our container off of, in this
case, I'm using microsoft's nanoserver image.

`LABEL maintainer="Adrian Lucrèce Céleste <adrianlucrececeleste@airmail.cc>"`

This line tells docker who the maintainer of this image is, by applying a label.

`RUN powershell Invoke-WebRequest -Uri https://verlonglink.com -Outfile hugo.zip`

The `RUN` statement tells docker to run powershell with the specified parameters
inside the container, `Invoke-WebRequest` is how I download the `.zip` that hugo
comes packaged in for windows.

`RUN powershell Expand-Archive C:\\hugo.zip -DestinationPath C:\\hugo`

Here, powershell is unzipping the archive to `C:\hugo` for us.

`WORKDIR C:\\workspace`

`WORKDIR` tells the container what directory to start out in at runtime.

`CMD [ "--help" ]`

`CMD` provides a default argument for the container while running, the container
passes `--help` to our `ENTRYPOINT` of `C:\hugo\hugo.exe`.

`ENTRYPOINT [ "C:\\hugo\\hugo.exe" ]`

The last and final line, `ENTRYPOINT` which tells the container what executable
(and optionally, parameters) we want to use during runtime.

## Preface

Part two is outdated, I now just use `wsl` to run `hugo` from instead of using
a windows docker container.

### Part Two: Setting up VScode to use the container on windows

Before in my [`tasks.json`](https://github.com/AdrianKoshka/blog/blob/master/.vscode/tasks.json),
I only had one `command` and one `args` section per task, now I seperate them
based on the OS that VScode is running on. So now if I wanted to preview my blog
on windows, all I have to do is run the `Preview` task, and the container will
be spun up, the ports mapped, and I can just open a web browser and look at my
progress. Building is also as easy as just running the build task.

![previewing the site](/imgs/preview-windows-container.jpg)

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Preview",
            "type": "process",
            "linux": {
                "command": "podman",
                "args": [
                    "run",
                    "--rm",
                    "--net=host",
                    "--name=hp",
                    "-v",
                    "${workspaceFolder}:/workspace:z",
                    "quay.io/adrianlucrececeleste/alpine-hugo:latest",
                    "serve",
                    "--theme=kiss",
                    "--source=/workspace/src",
                    "--buildDrafts"
                ]
            },
            "windows": {
                "command": "docker",
                "args": [
                    "run",
                    "--rm",
                    "-it",
                    "--isolation=hyperv",
                    "--name=hp",
                    "-p",
                    "1313:1313",
                    "-v",
                    "${workspaceFolder}:C:\\workspace",
                    "adrianlucrececeleste/windows-hugo:latest",
                    "serve",
                    "--theme=kiss",
                    "--bind",
                    "0.0.0.0",
                    "--source=src",
                    "--buildDrafts"
                ]
            },
            "problemMatcher": []
        },
        {
            "label": "Build Blog",
            "type": "process",
            "linux": {"command": "podman",
                "args": [
                    "run",
                    "--rm",
                    "--net=host",
                    "--name=hb-$(git rev-parse HEAD)",
                    "-v",
                    "${workspaceFolder}:/workspace:z",
                    "quay.io/adrianlucrececeleste/alpine-hugo:latest",
                    "--cleanDestinationDir",
                    "--source=/workspace/src",
                    "--destination=/workspace/docs",
                    "--theme=kiss",
                ]
            },
            "windows": {
                "command": "docker",
                "args": [
                    "run",
                    "--rm",
                    "--isolation=hyperv",
                    "-it",
                    "--name=hb",
                    "-p",
                    "1313:1313",
                    "-v",
                    "${workspaceFolder}:C:\\workspace",
                    "adrianlucrececeleste/windows-hugo:latest",
                    "--cleanDestinationDir",
                    "--source=C:\\workspace\\src",
                    "--destination=C:\\workspace\\docs",
                    "--theme=kiss"
                ]
            },
            "problemMatcher": [],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```