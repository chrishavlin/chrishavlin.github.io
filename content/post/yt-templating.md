---
title: "yt templating"
date: 2021-10-20T16:58:04-05:00
draft: false
---

I recently started working on a [`CookieCutter`](https://github.com/cookiecutter/cookiecutter) template for yt called [`ytcookiecutter`](https://github.com/data-exp-lab/ytcookiecutter)! The aim is to provide a convenient method for spinning up a new yt-related package, particularly new frontend plugins. The usage follows any `CookieCutter` template. 

```
pip install cookiecutter 
cookiecutter https://github.com/data-exp-lab/ytcookiecutter
```

will install `CookieCutter` and bring up the `CookieCutter` recipe builder. The template is built off audreyfeldroy's excellent [`cookiecutter-pypackage`](https://github.com/audreyfeldroy/cookiecutter-pypackage/) with some extra yt-specific options. At present, the yt options include options for including some expanded yt-dependencies in a `requirements.txt` file and the initial code needed for a frontend:

```
include_yt_requirements [y]: 
Select frontend_type:
1 - None
2 - AMR Skeleton
3 - Stream: Uniform Grid
4 - Stream: AMR Grids
5 - Stream: Particles
6 - Stream: Octree
7 - Stream: Hexahedral Mesh
8 - Stream: Unstructured Mesh
Choose from 1, 2, 3, 4, 5, 6, 7, 8 [1]: 2
```

Once selections are made, a new directory with everything you need to start building your new yt project is available! This includes source code but also github actions and other configurations (like pre-commit).

### how it works

One of the neat things about `ytcookiecutter` is that it actually does some meta-templating to generate the CookieCutter template! If you check out the base `ytcookiecutter` [repository](https://github.com/data-exp-lab/ytcookiecutter), you'll see some things common to other `CookieCutter` templates. The directory `{{cookiecutter.project_slug}}` contains the actual template code. When you run 

```
cookiecutter https://github.com/data-exp-lab/ytcookiecutter
```

the options you choose will fill in or select certain files in the template code.

But writing the template code itself can be time consuming. One of the frontend options I added is for a wrapper around an in-memory `stream` dataset. Now it's true that if you're familiar with yt, you can simply use a function like `yt.load_uniform_grid` to load in-memory data as a yt dataset. But several times I've found myself writing wrappers to `yt.load_uniform_grid` to load from disk and then load into yt. I do this a lot working with gridded seismic data that is small enough to work with in memory and not worth the extra development time of a full yt frontend. So I wanted to add a frontend option for a stream wrapper.

But if you actually check out the `stream` template code, you'll see it's fairly long and repetitive ( [link](https://github.com/data-exp-lab/ytcookiecutter/blob/main/%7B%7Bcookiecutter.project_slug%7D%7D/%7B%7Bcookiecutter.project_slug%7D%7D/frontend_templates/stream/%7B%7Bcookiecutter.project_slug%7D%7D.py)). Each of the `Stream` types has different arguments and so each needed their own version in the template. Annoying to write by hand... 

BUT I don't have to write it by hand! 

While [`ytcookiecutter`](https://github.com/data-exp-lab/ytcookiecutter) **is** a template, it is **also** a package itself to maintain the template. Among other things, it uses [jinja](https://jinja.palletsprojects.com/en/3.0.x/) (which `CookieCutter` is actually built on) for some meta-templating. The base `stream` meta-template looks like:

```
{%- if cookiecutter.frontend_type|lower == "!{frontend_type_str}!" -%}
from yt.loaders import !{load_func}!

def load(filename: str):

    # write code to load data from filename into memory!
    <% for arg in argnames -%>
    !{arg}! = ????????
    <% endfor %>
    # set or delete optional kwargs
    <% for key, value in kwarg_dict.items() -%>
    !{key}! = !{value}!
    <% endfor %>
    # call the stream data loader
    ds = !{load_func}!(
        <% for arg in argnames -%>
        !{arg}!,
        <% endfor -%>
        <% for key in kwarg_dict.keys() -%>
        !{key}! = !{key}!,
        <% endfor %>
    )

    # return the in-memory ds
    return ds
<% if include_docstring -%>
# description of !{load_func}! for convenience:

"""
!{docstring}!
"""
<% endif %>
{%- endif -%}
```

The main thing to notice is that this contains **two** sets of jinja start/end strings to identify the templated lines. The standard sets, `{% %}`, `{{  }}` are those used by `CookieCutter` while `<% %>` and `!{ }!` are custom start/end strings used by `ytcookiecutter`. To generate the `stream` template, I used `inspect.getfullarspec` to pull out the arguments and default values for each stream loading function and then used the above meta-template to generate the cookiecutter template, resulting in a template file containing all the arguments for the stream loading functions. And when a `stream` frontend is selected when building the CookieCutter recipe, only that one will remain in the generated project. Neat-o!

The `Skeleton` frontend template gets generated a little more simply. Using the [PyGithub](https://github.com/PyGithub/PyGithub) package, I simply fetch yt's `frontend/_skeleton` code and replace the `Skeleton` occurrences with cookiecutter template parameters so that when a user selects a `Skeleton` frontend, the new frontend name can easily be swapped in. 

Both of these approaches have the benefit of easily updating templates if yt changes upstream. Simply go and re-run the generation and push the new templates!
