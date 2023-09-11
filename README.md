# poetry-reasoning

A project which explains the reasoning for using [poetry](https://python-poetry.org/docs/) in python projects. The
project itself follows a mono-repository pattern, where services `a` and `b` use `poetry` (as does `root`) and service
`c` shows use cases of `pip freeze` and `pip-compile`.

## Table of Contents

- [Problem Statement](#problem-statement)
- [Pip freeze and pip-tools](#pip-freeze-and-pip-tools)
- [Anaconda/Conda](#anacondaconda)
- [Poetry](#poetry)

## Problem Statement

Ensuring reproducibility is a paramount requirement for production-level software. However, in Python projects,
achieving this can be challenging. The native package manager, `pip` (Preferred Installer Program), doesn't inherently
offer convenient tools to ensure reproducibility (as well as dependency resolution). As a result, several tools have
emerged to bridge this gap.

_NOTE: Reproducibility is most of the time ensured by a `.lock` file. A `.lock` file in dependency management systems is
used to "lock" the versions of the dependencies that a project uses. It's mostly used to lock transitive dependencies:
Projects often depend on packages that, in turn, have their own dependencies. These are known as transitive or indirect
dependencies. The lock file ensures that even these transitive dependencies are consistent across all environments._

Furthermore, many Python projects, particularly extensive mono-repositories, consist of production-level services,
quality assurance tooling and built packages that are consumed by services. This interdependence leads to pertinent
structural questions:

- What is the source of truth for dependency specification and installation?
- What is the source of truth for package build?
- What is the source of truth for tooling configuration?

In many instances, the answers are scattered across multiple files (`pyproject.toml`, `setup.cfg`, `requirements.txt`,
`.lock`) and tools (`pip`, `setuptools`, `twine`, `.venv`/`conda` etc) — leading to potential confusion and
inconsistencies.

This project provides an overview of available solutions and delves into why, at this juncture, `poetry`
stands out as the optimal choice.

## Pip freeze and pip-tools

Natively, pip offers only `freeze` command, which outputs the exact versions of all installed packages in the current
environment:

```bash
pip3 install --upgrade pip
pip3 freeze > requirements_freeze.txt
# see service-c/requirements_freeze.txt
```

This is not ideal, for two reasons:

1) Instead of having specified dependencies and having corresponding `.lock` file, it outputs **all** installed
   packages. This is prone to errors.
2) There are no convenient tools provided to update packages - poor developer experience (DX).

Pip-compile offers a bit more compared to `pip freeze`. Given "source of truth", it generates a requirements file with
transitive dependencies and adds comments from where did it come:

```bash
pip3 install --upgrade pip
pip-compile --extra dev -o requirements_pc.txt pyproject.toml
# see service-c/requirements_pc.txt
```

Then, the virtual environment can by synchronized with `pip-sync`:

```bash
pip-sync requirements_pc.txt
```

`pip-tools` also lack proper tooling to update dependencies conveniently. One will have to update the source of truth
manually, as `pip-compile --upgrade` only acts on the `requirements` derivative.

In the end, both `pip freeze` and `pip-tools` try to address only dependency-reproducibility issue. They do not tackle
package building or runtime version specification/verification.

## Anaconda/Conda

In the context of data science, `anaconda` and `conda` have emerged as primary tools to manage virtual environments and
dependencies. They offer centralized virtual environment management with support for multiple Python runtime versions,
come with pre-installed Python packages, and handle other necessary binaries. However, there are notable challenges:

- **Size of the Distribution**:
    - `Anaconda` is a heavyweight distribution.
    - Even the stripped-down `miniconda` is heftier than the vanilla Python virtual environment (`.venv`)
    - 3GB/400MB/24MB

- **Lack of Explicit Lock File Support**:
    - Using `conda env export > environment.yml` encounters issues similar to those seen with `pip freeze`.
    - To achieve locking functionality, one might need to resort to external tools
      like [conda-lock](https://github.com/conda/conda-lock).

- **Interoperability with pip**:
    - There are known package management challenges when mixing `pip`-installed packages within `conda` environments.
      Although the interoperability has improved over the years, it's not flawless.

It's worth noting that CI and production builds often sidestep `conda` due to its size, support for specific tools, and
some packages being exclusive to PyPI. This divergence can pose challenges, especially when developers locally rely on
different dependency, packaging, and environment management tools than what's used in production.

## Poetry

In the evolving landscape of Python dependency and environment management, `poetry` emerges as a modern and
comprehensive tool designed to simplify the process. Here are some of its distinct advantages:

- **Unified Dependency Specification**: With `poetry`, you declare dependencies and manage your project using a
  single `pyproject.toml` file, making dependency specification more organized and straightforward.

- **Packaging and Publishing**: Beyond dependency management, poetry can build packages directly from the pyproject.toml
  file. It also provides integrated tools for publishing packages to PyPI, streamlining the entire package lifecycle.

```bash
poetry build
poetry config repositories.my-repo <REPOSITORY_URL>
poetry publish my-repo
```

- **Integrated Locking Mechanism**: Unlike the challenges seen with `conda` and `pip freeze`, `poetry` generates
  a `poetry.lock` file out-of-the-box. This lock file ensures that all installations of the project will use the same
  versions of dependencies, enhancing reproducibility. With every `poetry add`, `poetry update` and `poetry remove`,
  the `.lock` file is managed automatically.

- **Built-in Environment Management**: `poetry` manages virtual environments seamlessly, ensuring that projects are
  isolated and don't interfere with one another, without needing third-party tools. Within `poetry` dependencies,
  one can specify python runtime versions that will be respected.

- **Robust Dependency Resolution**: `poetry` has a dependable resolver that reduces version conflicts, ensuring that the
  installed packages are compatible with each other.

- **Interoperability**: While `poetry` primarily interacts with PyPI, it's designed to be aware of potential challenges
  when using mixed package sources, thereby reducing integration issues seen with tools like `conda`.

- **Lightweight**: Without the bulk associated with distributions like `Anaconda`, `poetry` offers a leaner approach,
  making it more suitable for CI/CD pipelines and production builds.

In essence, `poetry` presents a unified, efficient, and user-friendly approach to Python project management, bridging
the gaps seen with other tools in the ecosystem. `service-a` and `service-b` are examples of how `poetry` could be used
in a mono-repository approach, which on a `root` level uses `poetry` too.

## Not Ideal

At the moment, `poetry` itself doesn't manage python runtime versions themselves. To successfully make initial install
with poetry (and to select the desired python version), additional tools might be needed (such like `pyenv`):

```bash
pyenv install 3.9
pyenv local 3.9
poetry install
```
