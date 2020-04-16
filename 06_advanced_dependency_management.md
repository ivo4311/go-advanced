# Dependency Management

## Abstract

In this section we will tackle dependency management in Go. We will go over the basics of Go modules and SemVer and the related tools and best practices. We will learn what the difference between a package, a go module and a repository is. Lastly we will go over the problems that arise from code reuse and how to address them.

## Go Modules

- What is a package
- What is a repository
- What is a go module
- What is SemVer
- What are `go.mod` and `go.sum` and should you commit them to SVC?

> checking go.mod and go.sum into source control is another safety net. if two developers are trying to push features that have the same dependency but are using different versions of that same library you will get a merge conflict and be able to fix it rather then find out runtime.

- go mod tidy
- go mod why -m
- hacking on your dependencies with `replace` or [gohack](https://github.com/rogpeppe/gohack)
- migrating a module to v2 of a dependency

> You can import both versions v1 and v2 of a module ... most of the time. If a package has global variables or init functions it will cause issues.

TODO: sections needs more code/console snippets

### References

- [Intro to Go Modules and SemVer](https://www.youtube.com/watch?v=aeF3l-zmPsY)
- [Migrating Go Modules to v2+](https://www.youtube.com/watch?v=H_4eRD8aegk)

## Q&A

## Dependency Management Problems

- Why `A little copying is better than a little dependency`? What do dependencies cost?
- Builds should be fast, repeatable and immutable. What does that mean and how can we get there?
- vendor folder
    - [+] builds are repeatable and immutable (if you trust your team to not push -f)
    - [-] versioning is very difficult since you are depending on source code and not a release (*)
    - [-] updating a dependency is difficult (*)
    - [-] repository size increases which leads to slower builds (consider CI servers cloning your repo)
    - [-] difficult to replace with local dependencies during development (*)
    - (*) these negatives can be offset by using go modules and go mod vendor
- [go center](https://search.gocenter.io/)
    - [+] builds are repeatable and immutable
    - [-] great for open source ... not so much for IP
    - [-] [Terms of Service](https://search.gocenter.io/terms) say ... no SLAs
    - (*) or just buy [artifactory](https://jfrog.com/pricing/#onprem) and run it yourself to offset the negatives in exchange for a higher tco
- [project athens](https://github.com/gomods/athens)
    - [+] builds are repeatable and immutable
    - [+] builds can be really fast (by running this with fast network storage)
    - [+] works for all dependencies - open source and in-house
    - [-] tco

### References

- [Go Center](https://search.gocenter.io/)
- [Go Modules: Dependency Management the Right Way](https://www.youtube.com/watch?v=-BIDXOp6_LA)
- [Project athens](https://github.com/gomods/athens)
- [Aaron Schlesinger: Bring Sanity to your Dependencies](https://www.youtube.com/watch?v=z_ki4_1gxgQ)

## Q&A and Wrap-up