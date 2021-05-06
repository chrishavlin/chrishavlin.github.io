To embeed jupyter notebook, use nbconvert to convert to markdown.

First run 
```
jupyter nbconvert --to markdown /path/to/notebook.ipynb --output-dir ./
```

The generated markdown can be copied into a post. Any image folder must be moved to `static/images`. and the image paths in the post need to be adjusted.
