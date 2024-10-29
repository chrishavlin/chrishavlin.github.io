---
file_format: mystnb
kernelspec:
  display_name: Python 3
  name: python3
---
# Quick Start

## Installation 

## Sample code 

```{code-cell} ipython3
import yt 
ds = yt.load_sample("IsolatedGalaxy")
yt.SlicePlot(ds, 'x', ('gas', 'density'))
```

For more, check out the full yt docs!

