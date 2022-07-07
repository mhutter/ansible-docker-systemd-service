# How to contribute

Hi and thanks for your interest in this project! Below are some guidelines on how to contribute to it.

**Table of contents**

* [Submitting Issues](#submitting-issues)
* [Contributing A Patch](#contributing-a-patch)
* [Running the tests](#running-the-tests)


## Submitting Issues

When reporting an issue, make sure to include enough information so that others can reproduce the bug. This usually includes describing the steps you did to reproduce the issue, the expected outcome, and the actual outcome.


## Contributing A Patch

Before you start working on a feature or fix, please submit an issue describing your proposed changes.

Once your proposal is accepted, fork the repo and start developing and testing (see below) your changes.

Ensure that your changes remain backwards compatible if possible.

When opening the pull request, title it following [Conventional Commits] styling.



## Running the tests

1. Ensure you have [yamllint] and Ansible installed.
1. Run yamllint:

        yamllint .

1. Run an Ansible syntax check:

        ansible-playbook tests/ci.yml -i tests/inventory --syntax-check


[Conventional commits]: https://www.conventionalcommits.org/
[yamllint]: https://yamllint.readthedocs.io/
