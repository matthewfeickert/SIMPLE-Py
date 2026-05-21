---
downloads:
  - file: index.md
    title: Source File
  - url: slides/1_01_setup.html
    static: false
    title: Slides
---

# Setting up for development

## Before you begin

You'll want to have the following tools installed:

- `uv`: A fast package manager written in Rust that replaces `pip`, `venv`, `pipx`, and more.
- `pixi`: A fast package manager written in Rust for conda packages (high-level only replacement for conda/mamba)
- A system compiler for the later parts of the workshop. (xcode-clt for macOS, Visual Studio for Windows, GCC for Linux)

These tools will be handy too, but they can be all installed via `uv tool install` so you don't need to pre-install them:

- `prek`: A linter/formatter runner, written in Rust (similar to pre-commit)
- `nox`: A task runner

## Low level concepts

(Or how not to break your computer)

The topics in this session are there just to give you an understanding of what's going on. We'll introduce high level abstractions in the next section!

### Virtual environments

The first thing you should know when packaging is how to install stuff.
There are three options:

|                          | Location          | Pros                | Cons                    |
|--------------------------|-------------------|---------------------|-------------------------|
| **System site-packages** | `/...`            | Works from anywhere | Can break your machine  |
| **User site-packages**   | `~/.local/...`    | User permission     | Can break other Pythons |
| **Virtual environment**  | `.venv/` (common) | Many!               | More effort             |

A system or user install sounds nice, but you can't install incompatible
packages, they can break your system, you can't control what each project
needs, you can end up not knowing what your requirements are, it's hard to
update, etc.

The standard solution is a virual environment. It places files inside a folder
with a name you choose (`.venv` in the project folder is the standard choice)
that looks like this:

```bash
.venv
├── .gitignore      # These make sure tools know not to include/archive
├── CACHEDIR.TAG    # this folder
│
├── bin
│   ├── activate    # "activation" scripts for different shells (bash default)
│   ...
│   ├── python      # Symlinks to your local Python install
│   ├── python3
│   └── python3.14
│
├── lib
│   └── python3.14
│       └── site-packages  # This is where installed libraries go
│           ...
│
└── pyvenv.cfg      # Special file telling Python this is a virtual environment
```

When Python runs, it checks to see if there's a `pyvenv.cfg` above it. If there is, it knows
it is in a virtual environment (venv) and reads site-packages from there. There are two ways to use it:

::::{tab-set}
:::{tab-item} Direct usage

```bash
.venv/bin/python ...
```

:::
:::{tab-item} Activation

```bash
. .venv/bin/activate  # Shell specific
python ...
deactivate
```

:::
::::

> [!WARNING]
> Some applications might require the enviroment be set to work. It's rare, but keep that in mind if you use the direct syntax. Things that use `sys.executable` work correclty.

The `.` at the start (most shells support `source` as well) allows the activation script to set environment variables, something a normal application could not do. It sets `PATH` (and `VIRTUALENV`, but that is informational) so things inside the virtual environment's bin are at the beginning of the PATH.

> [!NOTE]
> Conda environments: the `base` environment has many of the same problems. One environment per project is best.
> Conda lets you make environments by name (stored in a directory) or by path.

To create one of these, you have several options:

:::::{card} Create a virtual environment
::::{tab-set}
:::{tab-item} venv

```bash
python3 -m venv .venv
```

This is the slowest, but it's built in![^1]

:::
:::{tab-item} virtualenv

```bash
virtualenv .venv
```

This is faster than `venv`, has better default installs inside, but does require installation.

:::
:::{tab-item} uv

```bash
uv venv
```

This is really fast, though it's completly empty (no pip). And it defaults to `.venv`. Also requires installation (a single Rust binary or pip install).

:::
::::
:::::

We will be using `uv`, which can do a lot of this for us.

[^1]: Distibutions may strip it out and make it a seperate intallable package.


### Requirements

Now that you know how to make virtual environments, how should you install stuff into them? If the environment is active, then installers will target it. If an environment is not active, `uv pip install` will check for a `.venv` folder, and will target that by default (which is why it doesn't need to install `pip` in the venv).

But a virtual environment is meant to be expendable. You should be able to delete it and recreate it any time. So instead of manually installing, you want to list packages in some format:

:::::{grid} 1 1 2 2
::::{grid-item}
:::{card} Project (app)
These are for making a virtual env. They don't affect libraries.

* **requirement.txt**: Classic, very old format
  * Basically a list of args to pass to `pip install`
* **requirements.in**: Manual locking
  * You make a locked requirements.txt from this file
* **Lock file**: Versions are pinned exactly
* **dependency-groups**: Multiple collections of packages
:::
::::
::::{grid-item}
:::{card} Package (library)
These are for libraries.

* **dependencies**: The minimum requirements to install your package
* **optional-dependencies**: Sets of extra requirements that can be requested on install

Most libraries also have developer environements, which follows the "Project
(app)" patterns for things like tests and documentation.

> [!WARNING]
> For historical reasons, `optional-dependencies` is sometimes used for
> for these too; this is due to it pre-dating `dependency-groups`.

:::
::::
:::::

Locked dependencies means that every dependency is fully specified, ideally
with a SHA to make sure users get exactly the versions you used. Using a
lockfile is a great way to recover a virtual environment exactly on another
machine. However, dependencies cannot be locked for libraries; it would be a
problem if your two favorite libraries could not be installed together in one
environment because they conflicted on pins!


### Summary

So far, we have discussed:

:::{glossary}
venv
: Virtual environment that isolates dependencies

requirements
: A list of packages to install

lock files
: Fully pinned set of packages
:::

This might seem simple, but creating, activating, installing, and locking are all seperate steps with different incatations and different tools.

:::{exercise} Basic virtual environment
:label: basic-venv

Use uv to create a `.venv` virtual environment. Install the `cowsay`
package and then run `cowsay`. Remove the venv folder when done.

:::

:::{solution} basic-venv
:class: dropdown

Assuming a unix-like system and bash:

```bash
uv venv                 # defaults to .venv
uv pip install cowsay   # defaults to .venv
.venv/bin/cowsay        # run without activation is fine
rm -r .venv             # venvs can be removed
```

:::

## High level packaging

### Apps

#### Persistent apps

Let's break up applications and libraries. An application is something you install and run, but
(assuming you use virtual environments) never needs to be installed with other _unrelated_ things (apps that support plugins are okay). Generally, you won't `import` an app inside Python, you'll run it from the command line (or a graphical interace, etc).

This special propety allows us to do something interesting. Imagine we:

1. Made a venv somewhere
2. Installed our application in it
3. Put _just_ that application somwehere on our PATH

Since we never need to import it, we can get away without activation. This is exactly what `pipx` (pip for executables) and `uv tool` do!

Use `uv tool install` to install apps. Use `uv tool list` to see what you've
installed. `uv tool upgrade` to upgrade, and `uv tool uninstall` to remove
them. (See `uv tool --help`).


:::{exercise} Tool install
:label: tool-install

Use `uv tool` to install `cowsay`. Run it. Remove the tool when done.

If you have trouble, uv's tool directory might not be on your PATH. It should
warn you and tell you how to fix it.

:::

:::{solution} tool-install
:class: dropdown

Assuming a unix-like system and bash:

```bash
uv tool install cowsay
cowsay
uv tool uninstall cowsay
```

:::


#### Single use apps

We can do one better if we have something we don't want to run all the time.
The install and run steps can be combined! This is such a common need that
`uv` comes with a seperate CLI `uvx` that does this. Running this:

```bash
uvx cowsay
```

Will make a venv, download the app, then run a command with the same name in
the venv.  If you run it again, it will recreate the venv if it's over a week
old.

With this, you basically have all of PyPI at your fingertips, and you don't
have to remember to update things too!

