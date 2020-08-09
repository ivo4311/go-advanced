# Security Best Practices for Go Applications in the Cloud

## About

In this session we will consider the challenges in writing secure Go applications for the cloud

> Based on the 2019 talk [Going Secure with Go](https://www.youtube.com/watch?v=9e2gRtzemGo) by Natalie Pistunovich

## About the author/presenter

## Challenges

- How to secure the dependencies we use in our application
- How to secure our runtime (containers)

## Solution

- Dependencies checklist
    - Code Quality
    - Activity
    - Maintainers
    - Security
    - Licenses
- https://research.swtch.com/deps

- example 1
    - Picking a "safe" dependency is not enough - you need to apply patches regularly
    - Know your transitive dependencies
    - Continuously monitor for new vulnerabilities [dependabot](https://dependabot.com/go/)
    - Upgrade in a timely manner
- [example 2](https://www.theregister.co.uk/2018/11/26/npm_repo_bitcoin_stealer/)
    - Multiple maintainers is better than one maintainer
    - Automatic updates have risks -> lock your dependencies versions
- example 3 -> Someone pulls their code from the publicly available repo
    - Immutable dependencies are a must for a reproducible build
    - Central repositories are not reliable
- example 4 -> change of a license
    - Check the license before you upgrade
- Summary
    - Regularly check and update your go.mod
        - Unused packages are deleted from go.mod
        - Transitive dependencies are added to go.mod
    - Use vulnerability and licenses databases (e.g. NVD, JFrog Xray)
    - [Go Report Card](https://goreportcard.com/) as a minimum

## Containers

- How to delete something from a container
- Multi-stage builds
- Policies

> TODO: add examples

## Closing words

## References
