---
downloads:
  - url: /slides/1_01_setup.html
    static: false
    title: Slides
---

# Setting up for development

[Slides](/slides/1_01_setup.html)

## Before you begin

You'll want to have the following tools installed:

- `uv`: A fast package manager written in Rust that replaces `pip`, `venv`, `pipx`, and more.
- `pixi`: A fast package manager written in Rust for conda packages (high-level only replacement for conda/mamba)
- A system compiler for the later parts of the workshop. (xcode-clt for macOS, Visual Studio for Windows, GCC for Linux)

These tools will be handy too, but they can be all installed via `uv tool install` so you don't need to pre-install them:

- `prek`: A linter/formatter runner, written in Rust (similar to pre-commit)
- `nox`: A task runner

## Basic concepts

(Or how not to break your computer)

### Virtual environments

The first thing you should know when packaging is how to install stuff.
There are three options:

|                          | Location          | Pros                | Cons                   |
|--------------------------|-------------------|---------------------|------------------------|
| **System site-packages** | `/...`            | Works from anywhere | Can break your machine |
| **User site-packages**   | `~/.local/...`    | Works for user      | Only one, can break    |
| **Virtual environment**  | `.venv/` (common) | Many!               | More effort            |

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



> [!NOTE]
> Conda environments: the `base` environment has many of the same problems. One environment per project is best.
> Conda lets you make environments by name (stored in a directory) or by path.
