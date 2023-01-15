---
title:  "Writing CLIs in Go with Cobra"
author: "Mario"
tags: [ "go", "golang", "cobra", "clis", "development" ]
url: "/writing-clis-go-cobra/"
draft: false
date: 2023-01-15
#lastmod: 2023-01-15
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Writing CLIs in Go using Cobra

A few months ago, I was working with my colleague [Alberto](https://github.com/alknopfler) in a CLI. The language of choice was Go and I manage to learn a thing or two. In today's post we will see a very basic skeleton for a CLI written in Go. This will certainly help me in the future as a `template` for the next CLI, and maybe it helps you too!

Before we start, Alberto has also a [blog](https://alknopfler.github.io/) where he shares his knowledge, don't forget to check it out.

## Cobra

If you searched for CLI libraries in Go, I'm 100% sure that you already know [Cobra](https://github.com/spf13/cobra). It's the most used library for writing CLI tools and projects such as Kubernetes, Hugo, and GitHub CLI use it under the hood.

Cobra is the library we will be using for building our CLI.

## Project structure

{{<tip>}}
You can find the example project at [https://github.com/mvazquezc/go-cli-template](https://github.com/mvazquezc/go-cli-template)
{{</tip>}}

We will structure our CLI project as follows:

~~~sh
.
├── cmd
│   ├── cli
│   │   ├── run.go
│   │   └── version.go
│   └── main.go
└── pkg
    ├── example
    │   └── run.go
    └── version
        └── version.go
~~~

* The `cmd` folder will have the root command implementation defined in the `main.go` file and a sub-folder named `cli`. This sub-folder will have the implementation of the different sub-commands for our CLI. In this example we have two sub-commands: `run` and `version`.
* The `pkg` folder will have the libraries required by our sub-commands. This is optional, but in our case we implement the `run` sub-command functionality in a package named `example` and the `version` sub-command functionality in a package named `version`.

## CLI Implementation

In this section we will go over the CLI implementation, we will start with the root command and will continue with the sub-commands and packages.

### Root Command

The root command is implemented in the `cmd/main.go` file, below the code used with comments:

~~~go
// This is our main package
package main

// Import required packages
import (
	color "github.com/TwiN/go-color"
	"github.com/mvazquezc/go-cli-template/cmd/cli"
	"github.com/spf13/cobra"
	"log"
	"os"
)

// Our CLI main function
func main() {
    // Create a cobra command that will be the CLI entry-point
	command := newRootCommand()
    // If the command execution fails return the error (this includes errors raised at sub-commands)
	if err := command.Execute(); err != nil {
		log.Fatalf(color.InRed("[ERROR] ")+"%s", err.Error())
	}
}

// newRootCommand implements the root command of example-ci
func newRootCommand() *cobra.Command {
    // Define a new cobra command with the binary name of our CLI and the 
    // This command run without sub-commands will return the help
	c := &cobra.Command{
		Use:   "example-cli",
		Short: "Example cli written in go",
		Run: func(cmd *cobra.Command, args []string) {
            // Return help if no sub-command received
			cmd.Help()
			os.Exit(1)
		},
	}
    // Add sub-commands to our main command
	c.AddCommand(cli.NewRunCommand())
	c.AddCommand(cli.NewVersionCommand())

	return c
}
~~~

### Run Command

The run command is implemented in the `cmd/cli/run.go` file, below the code used with comments:

~~~go
// This is our cli package
package cli

// Import required packages
import (
	"errors"
	"github.com/mvazquezc/go-cli-template/pkg/example"
	"github.com/spf13/cobra"
)

// Define vars used to store sub-command parameters
var (
	stringParameter      string
	intParameter         int
	stringArrayParameter []string
)

// NewRunCommand implements the run sub-command of example-ci
func NewRunCommand() *cobra.Command {
  // Define a new cobra command for the run sub-command 
  cmd := &cobra.Command{
		Use:   "run",
		Short: "Exec the run command",
		RunE: func(cmd *cobra.Command, args []string) error {
			// Validate command Args with the validateRunCommandArgs function we created
			err := validateRunCommandArgs()
            // If arguments are not valid, return error to the user
			if err != nil {
				return err
			}
			// run command logic is implemented in the example package, we call the function here
			err = example.RunCommandRun(stringParameter)
            // If the command fails, retun error to the user
			if err != nil {
				return err
			}
			return err
		},
	}
    // Add run sub-command flags using the function we created
	addRunCommandFlags(cmd)
	return cmd
}

// addRunCommandFlags receives a cobra command and adds flags to it
func addRunCommandFlags(cmd *cobra.Command) {

    // Define the flags we want to use for our run sub-command
	flags := cmd.Flags()
	flags.StringVarP(&stringParameter, "string-parameter", "s", "", "A string parameter")
	flags.IntVarP(&intParameter, "int-parameter", "i", 1, "An int parameter")
	flags.StringArrayVarP(&stringArrayParameter, "string-array", "a", []string{"example"}, "A string array parameter")
  // We can make flags required
	cmd.MarkFlagRequired("string-parameter")
}

// validateCommandArgs validates that arguments passed by the user are valid
func validateRunCommandArgs() error {
	if intParameter != 1 {
		return errors.New("Not valid int-parameter")
	}
	return nil
}
~~~

#### Run Command Logic

The run command logic is implemented in the `pkg/example/run.go` file, below the code used with comments:

~~~go
// This is our example package
package example

import "fmt"

// RunCommandRun has the logic for running the run sub-command
func RunCommandRun(stringParameter string) error {
	fmt.Printf("Run command executed with string parameter set to %s\n.", stringParameter)
	return nil
}
~~~

### Version Command

The version command is implemented in the `cmd/cli/version.go` file, below the code used with comments:

~~~go
// This is our cli package
package cli

// Import required packages
import (
	"fmt"
	"github.com/mvazquezc/go-cli-template/pkg/version"
	"github.com/spf13/cobra"
)

// Define vars used to store sub-command parameters
var (
	short bool
)

// NewRunCommand implements the version sub-command of example-ci
func NewVersionCommand() *cobra.Command {
    // Define a new cobra command for the version sub-command 
	cmd := &cobra.Command{
		Use:   "version",
		Short: "Display version information",
		RunE: func(cmd *cobra.Command, args []string) error {
			if !short {
				fmt.Printf("Build time: %s\n", version.GetBuildTime())
				fmt.Printf("Git commit: %s\n", version.GetGitCommit())
				fmt.Printf("Go version: %s\n", version.GetGoVersion())
				fmt.Printf("Go compiler: %s\n", version.GetGoCompiler())
				fmt.Printf("Go Platform: %s\n", version.GetGoPlatform())
			} else {
				fmt.Printf("%s\n", version.PrintVersion())
			}
			return nil
		},
	}
    // Add run sub-command flags using the function we created
	addVersionFlags(cmd)
	return cmd
}

// addVersionCommandFlags receives a cobra command and adds flags to it
func addVersionFlags(cmd *cobra.Command) {
	flags := cmd.Flags()
	flags.BoolVar(&short, "short", false, "show only the version number")
}
~~~

#### Version Command Logic

The run command logic is implemented in the `pkg/version/version.go` file, below the code used with comments:

~~~go
// This is our version package
package version

import (
	"fmt"
	"runtime"
)

// Define vars used by our package
var (
	version    = "0.0.1"
	buildTime  = "1970-01-01T00:00:00Z"
	gitCommit  = "notSet"
	binaryName = "example-cli"
)

// PrintVersion prints our root command version
func PrintVersion() string {
	version := fmt.Sprintf("%s v%s+%s", binaryName, version, gitCommit)
	return version
}

// GetGitCommit returns the gitCommit
func GetGitCommit() string {
	return gitCommit
}

// GetBuildTime returns the buildTime
func GetBuildTime() string {
	return buildTime
}

// GetGoVersion returns the Version
func GetGoVersion() string {
	return runtime.Version()
}

// GetGoPlatform returns the go platform
func GetGoPlatform() string {
	return runtime.GOOS + "/" + runtime.GOARCH
}
// GetGoCompiler returns the go compiler
func GetGoCompiler() string {
	return runtime.Compiler
}
~~~

## CLI in Action

Once we have the CLI implemented, this is what we will get.

1. Compile the CLI:

    ~~~sh
    go build -o example-cli cmd/main.go 
    ~~~

2. If we run the CLI without any sub-command, we will get the CLI help:

    ~~~sh
    ./example-cli 
    Example cli written in go

    Usage:
      example-cli [flags]
      example-cli [command]

    Available Commands:
      completion  Generate the autocompletion script for the specified shell
      help        Help about any command
      run         Exec the run command
      version     Display version information

    Flags:
      -h, --help   help for example-cli

    Use "example-cli [command] --help" for more information about a command.
    ~~~

3. We can go ahead and execute the version sub-command with the `--sort` flag:

    ~~~sh
    ./example-cli version --short
    example-cli v0.0.1+notSet
    ~~~

4. We can also execute the run sub-command:

    ~~~sh
    ./example-cli run -s hello
    Run command executed with string parameter set to hello
    ~~~

5. And if something goes wrong, the user will be notified:

    ~~~sh
     ./example-cli run -s hello -i 3
      Error: Not valid int-parameter
      Usage:
        example-cli run [flags]

      Flags:
        -h, --help                       help for run
        -i, --int-parameter int          An int parameter (default 1)
        -a, --string-array stringArray   A string array parameter (default [example])
        -s, --string-parameter string    A string parameter

      2023/01/15 23:46:17 [ERROR] Not valid int-parameter
    ~~~

## Closing Thoughts

As you have seen, writing CLIs in Go is pretty easy with the help of libraries like Cobra. If you want to see a more advanced implementation of a CLI you can check the CLI Alberto and I built [here](https://github.com/RHsyseng/ddosify-tooling/tree/main/tooling/cmd).
