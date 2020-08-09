# Go Modules

## About

In this session we will tackle dependency management in Go. We will go over the basics of Go modules and SemVer and the related tools and best practices. We will learn what the difference between a package, a go module and a repository is.

## About the author/presenter

## Challenges

- What is a package
- What is a repository
- What is a go module
- What is SemVer
- What are `go.mod` and `go.sum` and should you commit them to SVC?

## Solution

TODO: refactor the content of this section to better fit the webinar format.

> checking go.mod and go.sum into source control is another safety net. If two developers are trying to push features that have the same dependency but are using different versions of that same library you will get a merge conflict and be able to fix it rather then find out runtime.

- go mod tidy
- go mod why -m
- hacking on your dependencies with `replace` or [gohack](https://github.com/rogpeppe/gohack)
- migrating a module to v2 of a dependency

> You can import both versions v1 and v2 of a module ... most of the time. If a package has global variables or init functions it will cause issues.

TODO: sections needs more code/console snippets

## Closing words

## References

- [Intro to Go Modules and SemVer](https://www.youtube.com/watch?v=aeF3l-zmPsY)
- [Migrating Go Modules to v2+](https://www.youtube.com/watch?v=H_4eRD8aegk)
