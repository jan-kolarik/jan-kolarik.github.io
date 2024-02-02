---
title: "Maintaining C++ ABI compatibility"
date: "2024-02-22"
summary: "Update the codebase safely to not break your consumers"
tags: ["cpp", "dnf", "fedora"]
ShowCodeCopyButtons: true
ShowToc: true
ShowReadingTime: true
ShowPostNavLinks: true
---

## Intro

In this post, I'd like to share some insights on the crucial topic of
maintaining ABI compatibility with programs written in C++, especially
when your component serves as a dependency in a software ecosystem.
I'll try to provide some tips and insights into our approach within
the DNF5 project and explore specific tools for analyzing issues
related to ABI changes.

---

## Understanding ABI compatibility

In simple terms, the Application Binary Interface (ABI) is a contract
between different parts of a software system which ensures they can
work together seamlessly, regardless of the programming languages or
compilers used to create them.

When we mention maintaining ABI compatibility, our goal is to ensure
that clients of our library do not need to recompile their applications
when downloading and deploying a new version. They should be able to
continue running their application builds without making any changes
themselves.

### ABI vs API

It's also important to distinguish between ABI compatibility and
API compatibility.

API compatibility focuses on preserving the functionality and syntax
of the exposed interfaces. In other words, ABI compatibility ensures
that the compiled code can work interchangeably, while API compatibility
guarantees that the expected methods, parameters, and behaviors remain
consistent across different versions of the library.

Both aspects are crucial for a smooth and easy integration of new library
versions into existing applications.

---

## How to not break anything

To ensure ABI compatibility, developers should adhere to best practices,
which include avoiding changes to the layout of classes, the size of
data types, and the order of virtual functions in interfaces.

The KDE community has compiled [a comprehensive guide][1] on binary compatibility
issues with C++ that you might find helpful. The guide provides a list of
do’s and don’ts when writing cross-platform C++ code meant to be compiled
with several different compilers.

While breaking ABI compatibility is generally undesirable, there are situations
where it becomes necessary. This may be due to the need to address critical
security vulnerabilities, eliminate long-deprecated features that impede
development progress, or undertake significant refactoring to introduce major
enhancements that require incompatible changes in the library.

---

## Our approach to ABI compatibility in DNF5

### Pimpl

In DNF5, we commonly face a scenario during development where adding new items
to existing structures or classes, part of the external interface, can
potentially break ABI compatibility. To mitigate this, we proactively identify
candidates that might be affected in the future and convert them to use
the [Pimpl idiom][2].

The Pimpl idiom involves hiding the implementation details of a class
behind a pointer, using a forward declaration in the header file, and keeping
the implementation details in a separate source file. The public interface
exposed in the header file remains stable since it only contains a forward
declaration of the implementation class, ensuring that changes to internal
implementation details won't affect the ABI as seen by external code.

### Branching

Additionally, when anticipating an ABI-breaking change, we create a separate
development branch for introducing the next major version release. Here, we
consolidate all planned breaking changes, conduct thorough testing,
and introduce the changes in a single release. This approach not only minimizes
disruption for users but also allows more efficient testing of the entire set
of changes and enables clear communication with users about planned modifications.

### Bumping soname

Ensuring seamless transitions during changes that break the ABI, we practice
soname library bumps. This involves incrementing the version number associated
with our shared libraries on Linux systems. By doing so, we signal to users 
and applications that a backward-incompatible change has occurred. This soname
bump not only aligns with semantic versioning principles but also helps with
symbol versioning and dependency management.

---

## What about some tools

Of course, there are a lot of tools for static analysis that can compare the new
and existing binaries and check for backward API/ABI compatibility.

Keep in mind that any tool may produce false positives, so a manual review
of the outputs is still required.

### ABI Compliance Checker

One well-known tool for this purpose is the [ABI Compliance Checker][3], capable
of generating clear XML or HTML outputs:

![abi-compliance-checker-example](/posts/images/abi-compliance-checker-example.jpg "ABI Compliance Checker - Example output")

For a quick guide on using the commands, refer to [this resource][4].

### abidiff

Another commonly used tool is [abidiff][5], a command-line utility that compares
the ABI of two ELF shared libraries. It generates textual reports detailing changes
affecting exported functions, variables, and their types.

It can be easily deployed as a GitHub action to check ABI compatibility with new
changes in submitted pull requests. For example, you can review the [mlibc][6]
project, which has the configuration for such a workflow [here][7].

---

## Exploring Fedora Linux

Here are some specific insights related to the Fedora Linux and its RPM packages
ecosystem.

### rpminspect

When a package maintainer submits a new build or update through the [Bodhi][9] system,
automated test suites evaluate the build candidate. Some tests are mandatory,
preventing the package submission if they fail, while others are optional, requiring
user verification for potential issues:

![bodhi-tests-results](/posts/images/bodhi-tests-results.png "Bodhi Automated Tests")

One such tool in these automated tests is [rpminspect][8], contributing to general
analysis of RPM packages. It produces a comprehensive report on policy compliance,
changes between the previous and current build, and overall correctness and best
practices. Abidiff is utilized there as one of the available analysis modes:

![rpm-inspect-report](/posts/images/tmt-rpm-inspect-output.png "rpminspect report")

### Packit and Testing Farm

The [Packit][10] project, designed to automate the package release process, has recently
gained significant popularity. Maintainers can effortlessly create configurations
based on a wide range of available examples. Packit automation streamlines the entire
release process, from triggering builds and manipulating package spec files to generating
changelogs and running tests.

In the context of checking ABI changes, there's a simple [way][11] to configure a GitHub
action in your upstream project, enabling the triggering of rpminspect analysis on pull
requests. This functionality is managed by the [Testing Farm][12], Packit's testing system,
which is an integral part of the Fedora CI infrastructure.

---

## References

- [KDE Guide to Binary Compatibility][1]
- [ABI Compliance Checker][3]
- [abidiff][5]
- [rpminspect][8]

[1]: https://community.kde.org/Policies/Binary_Compatibility_Issues_With_C%2B%2B
[2]: https://en.cppreference.com/w/cpp/language/pimpl
[3]: https://lvc.github.io/abi-compliance-checker
[4]: https://fedoraproject.org/wiki/How_to_check_for_ABI_changes_with_abi_compliance_checker
[5]: https://sourceware.org/libabigail/manual/abidiff.html
[6]: https://github.com/managarm/mlibc
[7]: https://github.com/managarm/mlibc/blob/master/.github/workflows/abidiff.yml
[8]: https://github.com/rpminspect/rpminspect
[9]: https://bodhi.fedoraproject.org/
[10]: https://packit.dev/
[11]: https://packit.dev/docs/configuration/upstream/tests#rpminspect
[12]: https://docs.testing-farm.io/Testing%20Farm/0.1/index.html
