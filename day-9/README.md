# Day 9 — Fixing a Broken Cookiecutter Template for ML Projects

## Overview

Part of the *100 Days of MLOps* challenge. This exercise repairs a broken
[Cookiecutter](https://cookiecutter.readthedocs.io/) template so that it renders
cleanly, then uses it to scaffold a new ML project. The write-up documents what
Cookiecutter is, the failing starting point, the fix, the commands used to verify the
end state, and pointers for deeper study.

The scenario: the xFusionCorp Industries ML platform team keeps a Cookiecutter template
at `/root/code/mlops-template/` from which every new ML project is generated. The draft
template does not render — running `cookiecutter /root/code/mlops-template/` aborts with
a Jinja error. The template must be corrected until it renders, and a project must be
generated from it at `/root/code/churn-model/`.

## Task

The end state must satisfy all of the following:

- `cookiecutter.json` declares **four** variables:
  - `project_name` (default `my-ml-project`);
  - `author` (default `xFusionCorp`);
  - `python_version` (default `3.11`);
  - `ml_framework` with the choices `sklearn`, `pytorch`, `tensorflow`.
- The generated `requirements.txt` depends on `ml_framework`:
  - `scikit-learn` when `ml_framework == "sklearn"`;
  - `torch` when `ml_framework == "pytorch"`;
  - `tensorflow` when `ml_framework == "tensorflow"`.
- The generated `README.md` references both `project_name` and `author`.
- The template directory `{{cookiecutter.project_name}}/` contains the files `README.md`
  and `requirements.txt`, and the directories `data/`, `models/`, `src/`, `tests/`.
- A project generated at `/root/code/churn-model/` (`project_name=churn-model`,
  `ml_framework=sklearn`) has a `requirements.txt` listing `scikit-learn` and a
  `README.md` mentioning `xFusionCorp`.

## Background: what Cookiecutter is

Cookiecutter is a **command-line tool that generates projects from project templates**
("cookiecutters"). A template is just a directory tree: a `cookiecutter.json` file
describing the input variables, plus a `{{cookiecutter.<var>}}/` directory whose file and
directory names are themselves rendered. Cookiecutter asks the user for each variable,
then renders the tree through the [Jinja2](https://jinja.palletsprojects.com/) templating
engine.

Why teams use it:

- one command scaffolds a project with a **consistent structure** and boilerplate;
- variables and Jinja conditionals make the scaffold **adaptive** (different dependencies,
  files, or content per choice);
- the template is versioned and shared, so "how a project starts" becomes a **reviewed
  artefact** rather than tribal knowledge.

Anatomy of a template:

```
mlops-template/
├── cookiecutter.json                 # variables + defaults (a list = multiple choice)
└── {{cookiecutter.project_name}}/    # rendered directory name
    ├── README.md                     # rendered file content
    ├── requirements.txt
    ├── data/  models/  src/  tests/
```

Key files and concepts:

| Item | Meaning |
| --- | --- |
| `cookiecutter.json` | declares the input variables and their defaults |
| a list value, e.g. `["sklearn","pytorch","tensorflow"]` | becomes a multiple-choice prompt; the **first** entry is the default |
| `{{cookiecutter.<var>}}` | a variable reference, rendered by Jinja2 (also works in file/dir names) |
| `{% if %}` / `{% elif %}` / `{% endif %}` | Jinja2 control flow, e.g. conditional dependencies |
| `hooks/` | optional `pre_gen_project.py` / `post_gen_project.py` scripts |

Common commands:

| Command | Effect |
| --- | --- |
| `cookiecutter <template>` | render the template, prompting for each variable |
| `cookiecutter <template> --no-input` | render using defaults, no prompts |
| `cookiecutter <template> --no-input key=value` | render, overriding some variables |
| `cookiecutter <template> -o <dir>` | write the generated project into `<dir>` |
| `cookiecutter <template> --overwrite-if-exists` | overwrite an existing generated project |
| `cookiecutter --help` | full CLI reference |

## Initial state: the broken template

`cookiecutter.json` — only three variables, `ml_framework` missing:

```json
{
    "project_name": "my-ml-project",
    "author": "xFusionCorp",
    "python_version": "3.11"
}
```

`{{cookiecutter.project_name}}/README.md` — references a non-existent `Author` key
(capital `A`):

```markdown
# {{cookiecutter.project_name}}

Created by {{ cookiecutter.Author }}.
```

Running the template aborts while rendering the README:

```console
$ cookiecutter .
  [1/3] project_name (my-ml-project):
  [2/3] author (xFusionCorp):
  [3/3] python_version (3.11):
Unable to create file 'README.md'
Error message: 'collections.OrderedDict object' has no attribute 'Author'
```

| Requirement | Initial state |
| --- | --- |
| four variables incl. `ml_framework` | `cookiecutter.json` has only three |
| `ml_framework` choices | not declared |
| `README.md` references `author` | uses `{{ cookiecutter.Author }}` → render error |
| `requirements.txt` conditional on the framework | no Jinja conditional |
| `data/ models/ src/ tests/` | missing from the template |

## Solution

### 1. Declare the four variables

```json
{
    "project_name": "my-ml-project",
    "author": "xFusionCorp",
    "python_version": "3.11",
    "ml_framework": ["sklearn", "pytorch", "tensorflow"]
}
```

The list turns `ml_framework` into a 1/2/3 menu; the first element (`sklearn`) is the
default.

### 2. Fix the README variable name

```markdown
# {{cookiecutter.project_name}}

Created by {{ cookiecutter.author }}.

- Python version: {{ cookiecutter.python_version }}
- ML framework: {{ cookiecutter.ml_framework }}
```

`author`, not `Author` — Jinja is case-sensitive and the key in `cookiecutter.json` is
lower-case.

### 3. Make `requirements.txt` framework-dependent

```jinja
{% if cookiecutter.ml_framework == 'sklearn' %}
scikit-learn
{% elif cookiecutter.ml_framework == 'pytorch' %}
torch
{% elif cookiecutter.ml_framework == 'tensorflow' %}
tensorflow
{% endif %}
```

### 4. Add the standard directories

```bash
mkdir -p '{{cookiecutter.project_name}}/{data,models,src,tests}'
```

### 5. Generate the project

```bash
cookiecutter /root/code/mlops-template/ -o /root/code/ --no-input \
  project_name=churn-model ml_framework=sklearn
```

## Verification

From the sandbox:

```bash
# template structure
ls -R '/root/code/mlops-template/{{cookiecutter.project_name}}'
# → README.md  requirements.txt  data  models  src  tests

# the literal Jinja block must be present in the template file
grep -n '{% if' '/root/code/mlops-template/{{cookiecutter.project_name}}/requirements.txt'

# generated project
cat /root/code/churn-model/requirements.txt   # → scikit-learn
cat /root/code/churn-model/README.md          # → churn-model + xFusionCorp

# other branches
cookiecutter /root/code/mlops-template/ -o /tmp/ --no-input project_name=t ml_framework=pytorch
cat /tmp/t/requirements.txt                   # → torch
```

The end state is reached when all three branches render the right dependency, the README
mentions both the project name and the author, the template holds the required
files/directories, and `/root/code/churn-model/` exists with `scikit-learn` and
`xFusionCorp`.

## Gotchas

### Jinja is case-sensitive: `author` ≠ `Author`

`{{ cookiecutter.Author }}` resolves against the `cookiecutter` context dict, so a wrong
case raises `'collections.OrderedDict object' has no attribute 'Author'` and the file is
never written. The key must match `cookiecutter.json` exactly.

### `=` vs `==` in Jinja conditions

`{% if cookiecutter.ml_framework = 'sklearn' %}` is a syntax error
(`expected token 'end of statement block', got '='`). Jinja comparisons use `==`, and
every `{% if %}` needs a matching `{% endif %}`.

### Whitespace-trimming tags can break literal checks

`{%- if ... -%}` (with the trim dashes) is valid Jinja and renders fine, **but** a grader
that looks for the literal substring `{% if` will not match it. The KodeKloud checker does
exactly that, so the corrected template must use `{% if` / `{% elif` / `{% endif`
**without** dashes. Prefer the plain form when a grader (or `grep`) may inspect the
template source.

### Cookiecutter never overwrites silently

Rendering into a directory that already exists aborts with
`Error: "<name>" directory already exists`. Either remove the old project first or pass
`--overwrite-if-exists`.

### Generate from outside the template directory

Running `cookiecutter .` from inside the template writes the generated project *into* the
template tree, polluting it. Run from the parent directory (or use `-o`).

### `--no-input` still needs `key=value` to override defaults

With `--no-input`, unanswered variables fall back to their defaults — pass
`project_name=... ml_framework=...` to override them without prompts.

### Empty directories and git

Cookiecutter does create the declared directories, but empty directories are invisible to
git; if the template is versioned, add a `.gitkeep` in each empty directory so the
structure is preserved.

## Further reading

### Books

- **Publishing Python Packages: Test, share, and automate your projects** — Dane Hillard
  (Manning, 2022, ISBN 9781617299919). Chapter 10, *"Scaling and solidifying your
  practices"*, includes **§10.1 "Creating a project template for future packages"** — a
  step-by-step extraction of a cookiecutter template from an existing project. This is the
  closest book-length treatment of the exact skill exercised here.
  <https://www.manning.com/books/publishing-python-packages>
- **Python Packages** — Tomas Beuzen & Tiffany Timbers (Chapman & Hall/CRC, 2022), an
  **open-access** book at <https://py-pkgs.org/>. It ships a companion template,
  `py-pkgs-cookiecutter` (<https://github.com/py-pkgs/py-pkgs-cookiecutter>), used to
  scaffold the package built throughout the book.
  - *Note:* as with pre-commit, there is no book devoted **exclusively** to Cookiecutter;
    these are the two best chapters/templates inside broader Python-packaging texts.

### Official documentation

- Cookiecutter docs — <https://cookiecutter.readthedocs.io/en/stable/> — the reference for
  every CLI flag and `cookiecutter.json` key.
- Tutorial: *Getting to Know Cookiecutter* —
  <https://cookiecutter.readthedocs.io/en/stable/tutorials/tutorial1.html> — consuming an
  existing template end to end.
- Tutorial: *Create a Cookiecutter From Scratch* —
  <https://cookiecutter.readthedocs.io/en/stable/tutorials/tutorial2.html> — the closest
  match to this lab: authoring your own template, variable defaults, and a
  `{{ cookiecutter.project_slug }}` directory.
- Cookiecutter source — <https://github.com/cookiecutter/cookiecutter>.
- Jinja2 template syntax — <https://jinja.palletsprojects.com/en/stable/templates/> — for
  `{% if %}` / `{% elif %}` and whitespace control.
- Cookiecutter Data Science (CCDS) — <https://cookiecutter-data-science.drivendata.org/> —
  the community's canonical ML/data-science template; the natural next step from the
  mini-template built here.

### Blog articles

- Marc Semjan, *"Create Project Templates With Cookiecutter"* —
  <https://msemjan.github.io/en/posts/create-project-templates-with-cookiecutter/> — a
  concise, practical walkthrough of consuming a template and writing your first one.
- Thu Kha Kyawe, *"Day 9: Create a Custom ML Project Template with Cookiecutter"* —
  <https://dev.to/thukhakyawe_cloud/day-9-create-a-custom-ml-project-template-with-cookiecutter-4n7p>
  — the same KodeKloud lab solved step by step, useful as a cross-check.
- CodeRivers, *"Mastering Cookiecutter Python: A Comprehensive Guide"* —
  <https://coderivers.org/blog/cookiecutter-python/> — broader coverage of concepts,
  hooks, and best practices.

### Video tutorials

- **Michael Breitung — "Create Powerful Templates for Your Software Projects Using
  Cookiecutter"** — <https://www.youtube.com/watch?v=bh6S2981luk> — a short (≈8 min)
  hands-on intro: template example, setup, and usage.
- **Talk Python — "Using and Mastering Cookiecutter"** (paid course) —
  <https://training.talkpython.fm/courses/using-and-mastering-cookiecutter-templates-for-project-creation>
  — the most thorough video resource, from consuming templates to authoring them and using
  the Cookiecutter API.
- **PyData Berlin 2017 — "Kickstarting projects with Cookiecutter"** —
  <https://pyvideo.org/pydata-berlin-2017/kickstarting-projects-with-cookiecutter.html> —
  a conference talk covering CLI usage and authoring your own template, with a data
  science angle (`widget-cookiecutter`).
- **"Cookiecutter Django 2: Project Init / Project Tour"** —
  <https://www.youtube.com/watch?v=WPbgD2hFsFw> — a real-world tour of a large, popular
  cookiecutter (the Django one), showing what a mature template looks like.

## Repository notes

- This directory documents the exercise; the template and the generated project live on
  the lab sandbox at `/root/code/mlops-template/` and `/root/code/churn-model/` and are
  not committed here.
- `task.txt` is the exercise brief and is kept local only (see `.gitignore`).
