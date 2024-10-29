---
myst:
  html_meta:
    "description lang=en": |
      Top-level documentation for pydata-sphinx theme, with links to the rest
      of the site..
html_theme.sidebar_secondary.remove: true
---

# The yt project

this landing page is markdown! using the 
[pydata sphinx theme](https://pydata-sphinx-theme.readthedocs.io/en/v0.15.4/index.html), 
which seems pretty nice.


Card layout?

````{grid} 2

```{grid-item-card}  Supported Data formats
:link: about/formats.html
yt supports amr, etc... learn more!
```

```{grid-item-card}  Title 2
B
```

````

`````{div} dropdown-group

````{dropdown} Dropdown A

content A

````

````{dropdown} Dropdown B
Dropdown content B
````

`````





```{toctree}
:maxdepth: 2
:hidden:

about/index
develop/index
extensions/index
```