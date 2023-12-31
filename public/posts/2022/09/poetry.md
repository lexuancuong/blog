# Poetry: An alternative tool of Pip

## Problem setup

At present, the most well-known tool for package management is Pip. Pip is straightforward and user-friendly, but it does have its potential drawbacks.

In my experience, I have encountered a difficult situation where a developer upgraded a dependency in a repository that was being reused, and then merged that branch with an existing tag unexpectedly.
As a result, the CI/CD automatically rebuilt the docker image and released it.
However, I was unable to detect the changes in the versions of the package's dependencies using requirements.txt and Pip.
This led to a great deal of time, effort, and cost being spent on tracing back, fixing, deploying, and retesting.

Moreover, Pip is unable to identify conflicts and only prioritizes the order of dependencies, installing the last one.

Therefore, it is necessary to have a better mechanism and a lock file that can capture and freeze all the dependencies' versions to avoid any unexpected issues related to versioning or overriding.

## Features Comparision

Compare Poetry with the most popular package manager tools


|Features|[pipenv ](https://github.com/pypa/pipenv)|[poetry](https://github.com/python-poetry/poetry)|[pdm](https://github.com/pdm-project/pdm)|[pip-tools](https://github.com/jazzband/pip-tools) and [conda](https://github.com/conda/conda)|
| :- | :- | :- | :- | :- |
|Locked file safely manage packages  with these functions (Proper dependency resolution instead of last-write-wins and Lock file for reproducibility) |**✓**(Pipenv support this feature with Pipefile.lock)|✓(Poetry has the poetry.lock)|✓(Pdm has the pdm.lock)|Pip-tool and conda cannot satisfy this crucial requirement. So, these tools are out of the game**[1]**|

After comparing the first crucial feature, we just have only 3 candiatates: pipenv, poetry and pdm.

|Features|[pipenv ](https://github.com/pypa/pipenv)|[poetry](https://github.com/python-poetry/poetry)|[pdm](https://github.com/pdm-project/pdm)|
| :- | :- | :- | :- |
|Github Star|23k|20.4k|2.7k|
|Resolver looks at package metadata instead of downloading whole packages just to look at the dependencies **[2]**|**✓**|✓|✓|
|Separation of direct dependencies (wants), transitive dependencies (needs), dependency overrides (have-to).|**✓**|✓|✓|
|Central config file with pyproject.toml(Main dependencies, dev dependencies (black, isort, autopep8), internal dependencies could be located in an unique file - pyproject.toml)|**X**(pipenv use Pipefile to store its dependencies specification. Pipefile cannot be used for dev dependencies. E.g black,...)|**✓**|✓|
|Checks conflict version issues **[3]**|**✓**(Less readable than the other)|✓|✓|
|Performance/speed|**X**|**✓**Poetry is the winner overall.(You could have a look at the detailed table below.)|**X**|
|Properly resolve the conflict version of dependencies **[4]**|**X**|**X**(Currently, Python doesn't support have installing the same package with 2 different versions.)|**X**|

Let's compare some of the nice-to-have features to determine which tool is the winner.

|Features|[pipenv ](https://github.com/pypa/pipenv)|[poetry](https://github.com/python-poetry/poetry)|[pdm](https://github.com/pdm-project/pdm)|
| :- | :- | :- | :- |
|Support python3.7 well|**✓**|✓|**X**(Pdm use the __pypackage_\_ folder. It was introduced in [PEP582](https://peps.python.org/pep-0582/) for python3.8+)|
|Support Python virtual environment natively|**✓**|✓|**X**|
|Visualize the current nested dependencies from installed packages |**✓**|✓|**X**|



## Performance Testing
Based on our previous analysis, we have identified pipenv, poetry, and pdm as the top tools. Now, we will evaluate their performance for common tasks.


Context: [python:3.7-slim-buster](https://github.com/docker-library/python/blob/957d539a9b50aba11e8af8687b5f7c2fbd25ee24/3.7/slim-buster/Dockerfile)

Environment preparation:

**Step 1**: Install Pipenv, Poetry, and PDM based on their official documentation.

**Step 2**: Set up the environment by following command lines:

```text
- Pipenv: pipenv –three
- Poetry: poetry init -n
- PDM: pdm init -n
```

### Scenario #1: Install a single library


||Pipenv|Poetry|PDM|
| :- | :- | :- | :- |
|Without cache, no lock file|29.987s|52.265s|43.923s|
|With cache, no lock file|9.544s|2.153s|19.842s|
|With cache, reuse lockfile|3.407s|2.007s|20.867s|

### Scenario #2: Install a complex project (more than 30 dependencies including some building wheels required packages)

Add package by package by:

```bash
cat requirements.txt | xargs {specific_installing_function*}
```

With specific_installing_function in [‘poetry add’, ‘pipenv install’, ‘pdm add’]


||Pipenv|Poetry|PDM|
| :- | :- | :- | :- |
|Without cache, no lock file|1m41.109s|1m36.072s|1m39.084s|
|With cache, no lock file|1m10.554s|4.884s|44.484s|
|With cache, reuse lockfile|51.055s|3.884s|46.253s|




## Appendix

### [1] Pip-tool and conda don’t support the lock file

By default, pip-tools need to coordinate with [virtualenv to create the folder](https://github.com/jazzband/pip-tools#installation) to store installed package metadata. So, it doesn’t generate the lock file automatically.

About the conda, although conda could manipulate environments natively, it doesn’t manage packages by an isolated lock file.

### [2] Look at package metadata instead of downloading the whole package

With this feature we expect the tool must specify needed dependencies from metadata (e.g setup.py, pyproject.toml,...) before the downloading process. After specifying all of the necessary dependencies, the process concurrently installs them.

Pipenv, poetry, and pdm also have this resolver mechanism. They resolve the dependencies from metadata and install them in a parallel way.

### [3] Checks the conflict version issues

About this feature, we expect the tool could raise the confliction error in the scenario:

Step 1: PP_scraper requires the python-dateutil==2.8.2.

Step 2: After installing python-dateutil==2.8.2, we install the pp_saturn package. But the pp_saturn requires python-dateutil==2.8.1.

=> Poetry and Pdm have the error message more readable than the Pipenv.

### [4] Properly resolve the conflict version of dependencies**

**Problem setup:**

In the application, we would like to use:

- packageA, which requires packageX==1.3
- packageB, which requires packageX==1.4

We need to install package X with 2 different versions: 1.3 and 1.4.

**Solution:**

Currently, Pipenv, [poetry](https://github.com/python-poetry/poetry/issues/697#issuecomment-1142711474), [pdm](https://github.com/pdm-project/pdm/issues/1195#issuecomment-1173368501) couldn’t handle this problem and they have no plan to resolve this problem. We have no absolute solution, but we could walk around some ways:

- If these packages are editable, we should align the version of the conflicting package, and then change the version of one or both of them to resolve this conflict.

=> In our case, we should change the version of package X==1.4 inside package A, and test Package A with this Package X version. After that, we could smoothly install package A and package B by Poetry.

- If we cannot change the version of the conflicting package inside them, we could grudgingly use Pip to fix this problem.

We install package A by Poetry. And then, we install package B with pip.

- [Not prefer] Install into different folders for different package versions (follow this [suggestion](https://stackoverflow.com/questions/6570635/installing-multiple-versions-of-a-package-with-pip/6572017#6572017)). But we need to work around a lot and I cannot ensure the stability of this solution.

## Reference links

<https://remastr.com/blog/pip-pipenv-poetry-comparison>

<https://stackoverflow.com/questions/58218592/feature-comparison-between-npm-pip-pipenv-and-poetry-package-managers>
