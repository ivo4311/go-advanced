# Advanced Dependency Management

## About

In this session we will go over the problems that arise from code reuse and how to address them when developing application with Go.

## About the author/presenter

## Challenges

- Why `A little copying is better than a little dependency`? What do dependencies cost?
- Builds should be fast, repeatable and immutable. What does that mean and how can we get there?

## Solution

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

## Closing words

## References

- [Go Center](https://search.gocenter.io/)
- [Go Modules: Dependency Management the Right Way](https://www.youtube.com/watch?v=-BIDXOp6_LA)
- [Project athens](https://github.com/gomods/athens)
- [Aaron Schlesinger: Bring Sanity to your Dependencies](https://www.youtube.com/watch?v=z_ki4_1gxgQ)
