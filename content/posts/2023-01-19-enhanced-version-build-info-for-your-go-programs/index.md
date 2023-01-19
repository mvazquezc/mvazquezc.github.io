---
title:  "Enhanced Version and Build Information for your Go programs with ldflags"
author: "Mario"
tags: [ "go", "golang", "cobra", "clis", "development" ]
url: "/enhanced-version-and-build-information-for-your-go-programs/"
draft: false
date: 2023-01-19
#lastmod: 2023-01-19
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Enhanced Version and Build Information for your Go programs with ldflags

In the [previous post](https://linuxera.org/writing-clis-go-cobra/) we show how to create a simple CLI in Go with Cobra. I received a suggestion from one of my colleagues, [Andrew Block](https://twitter.com/sabre1041). He suggested complementing that post with the use of ldflags at build time in order to define specific information to a specific build like build time, git commit, etc.

Andy is also running a [blog](https://blog.andyserver.com/), go check it out!

## How does ldflags improve the build and version information?

With the use of `ldflags` we can change values for variables defined in our go programs at build time. Taking the [CLI program](https://github.com/mvazquezc/go-cli-template) we created in the previous post as an example, in the `version.go` file we defined the following variables:

~~~go
var (
	version    = "0.0.1"
	buildTime  = "1970-01-01T00:00:00Z"
	gitCommit  = "notSet"
	binaryName = "example-cli"
)
~~~

You can see these variables have pre-defined values that are not being set to anything meaningful, some of these variables will benefit from having dynamic values set at build time, like for example `buildTime` and `gitCommit`.

In this post we will see how we can modify the value of these vars at build time. This information will be helpful in the future if we need to know what code is included in a specific binary or what time it was built at.

Remember, we're using the [CLI program](https://github.com/mvazquezc/go-cli-template) we created in the previous post for the next steps. 

## Building our Go programs with ldflags

The Go ldflags supports passing many link flags, you can find the ldflags documentation [here](https://pkg.go.dev/cmd/link). For now, we're going to focus on how variable values can be changed at build time.

We will be using the -X flag which is described in the docs as _Set the value of the string variable in importpath named name to value_.

The syntax is something like this:

~~~sh
go build -ldflags="-X 'package_path.variable_name=variable_value'"
~~~

For example, if we had a variable named `version` in the `main` package we could change it like this:

~~~sh
go build -ldflags="-X 'main.version=v1.0'"
~~~

In our example application, the variables we need to modify are present in a sub-package though, so the `package_path` we need to use is a bit different, and not always easy to deduce. For these cases we can use the `go tool nm` command to help us find the variables package_path easily.

If for example, we would like to change the value for the `gitCommit` var we would do something like this:

1. Get the `package_path` for the `gitCommit` variable:

    ~~~sh
    $ go tool nm ./example-cli | grep gitCommit
    
    6add80 D github.com/mvazquezc/go-cli-template/pkg/version.gitCommit
    ~~~

2. Run the go build with ldflags:

    ~~~sh
    go build -ldflags="-X 'github.com/mvazquezc/go-cli-template/pkg/version.gitCommit=changed_at_build_time'" -o example-cli cmd/main.go
    ~~~

3. If we run our application we will see the new value for the `gitCommit` variable:

    ~~~sh
    $ ./example-cli version | grep commit
    
    Git commit: changed_at_build_time
    ~~~

{{<warning>}}
ldflags only support changing variables of type string. On top of that, these variables cannot be constants or get their value set from a function call.
{{</warning>}}

Now that we explained how `ldflags` can be used, let's finish our example with relevant information by using commands to gather the data at build time.

~~~sh
go build -ldflags="-X 'github.com/mvazquezc/go-cli-template/pkg/version.gitCommit=$(git rev-parse HEAD)' -X 'github.com/mvazquezc/go-cli-template/pkg/version.buildTime=$(date +%Y-%m-%dT%H:%M:%SZ)'" -o example-cli cmd/main.go
~~~

The result will be something like this:

~~~sh
$ ./example-cli version

Build time: 2023-01-19T19:11:37Z
Git commit: 23456bce1170173af228f47162fa7c70ae884d02
Go version: go1.18.8
Go compiler: gc
Go Platform: linux/amd64
~~~

## Closing Thoughts

Adding information like this to your Go programs can help you in different ways, for example when someone reports a regression bug having information about the git commit can help you identify when the regression was introduced. There are multiple use cases, now it's your turn to investigate how you can leverage `ldflags` in your builds!
