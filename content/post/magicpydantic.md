---
title: "Autogeneration of magicgui-pydantic widgets"
date: 2022-04-05T13:45:56-04:00
draft: false
---

This post describes a simple way of generating a magicgui widget from a pydantic model. 

## Background

For a little while now I've been working on a new [yt-napari plugin](https://github.com/data-exp-lab/yt-napari) which will create an interface for yt within napari ([background](https://ischool.illinois.edu/news-events/news/2021/11/ischool-researchers-receive-funding-napari-plugin-project)). The initial phases of the project (1) built out a reader plugin that lets you load a json file specifying the information needed to load data from a yt dataset like fields and selections and (2) added some helper functions for adding yt layers to an active napari viewer from the notebook, taking care of properly aligning multiple layers in napari's image coordinates. Lately, though I've been working on the final main piece for the initial release: a dockable widget for loading data. 

[Napari](https://napari.org) provides some great ways of building out widgets, including the related [magicgui](https://napari.org/magicgui/) package that helps to streamline widget creation. After playing around with building out simple widgets for data loading that allow the user to supply filenames, fields and other parameters, I turned to working out how to use the pydantic model that I already had for the reader plugin to generate a GUI plugin. In pseudo-code, my question was if I have some nested pydantic models:

```
import pydantic


class Field(pydantic.BaseModel):
    field_type: str
    field_name: str
    
class DataModel(pydantic.BaseModel):
    file: str
    field: Field
    ... etc ... 
    
```

and I already have methods for ingesting a `DataModel` instance and returning a yt dataset, how do I create a magicgui widget that lets me set all those fields and return the yt dataset? It turns out that there actually is some work in progress to allow direct generation of widgets from pydantic models ([link](https://github.com/napari/magicgui/pull/318)), but since my pydantic models are fairly straightforward it's actually not too bad to automate this myself until the magicgui devs formally implement it.

## requirements

The tutorial here relies on [magicgui](https://github.com/napari/magicgui) and [pydantic](https://pydantic-docs.helpmanual.io/) installed. So go do that! 

## magicgui intro

Ok, so I've not worked with widgets very much and there were a few things that I needed to work out that helped immensely. The magicgui intro ([link](https://napari.org/magicgui/usage/quickstart.html)) is great, but I knew that my case would be too complex to work with the function-decorating -- I needed to construct some gui classes with callbacks. But I was missing some basics... So first off, when you have a container widget:

``` python
from magicgui import widgets
container = widgets.Container()
```

and add another widget to that container, specifying the `name` parameter:

```python
container.append(widgets.SpinBox(name="x"))
```
then that new `SpinBox` widget is available as an attribute under `container`:

```python
print(container.x)

    SpinBox(value=0, annotation=None, name='x')
```

And to pop up a gui window, we do:

```python 
container.show(run=True)
```

and get our little widget:

![png](/images/magicguipydantic/widgets_01.png)

The second important bit is the mechanics of callbacks with magicgui. Most (all?) of the magicgui widgets have attributes that can be connected to callbacks. For example, the `SpinBox` that we added has a `.changed` attribute that will signal when the value is changed. So if we have a widget to display a result

```python
container.append(widgets.Label(name="result"))
```

and a function that looks at the value of x and updates the display:

```python
def calculate_result():
    result = c.x.value * 2
    c.result.value = f"{result}" 
```

we can trigger that function with 

```python 
container.x.changed.connect(calculate_result)
```

so now when we do 

```
container.show(run=True)
```
and change the value of `x`, we'll get:

![png](/images/magicguipydantic/widgets_02.png)


Putting all the above snippets together:

```python 
from magicgui import widgets

container = widgets.Container()
container.append(widgets.SpinBox(name="x"))
container.append(widgets.Label(name="result"))

def calculate_result():
    result = container.x.value * 2
    container.result.value = f"{result}" 
    
    
container.x.changed.connect(calculate_result)  
container.show(run=True)  
```

The final piece needed is to understand a little of how magicgui does its magic. When you use the fancy magicgui function decorators, it checks the type of the arguments and returns an appropriate widget. I still want to make use of that here, but in a little more explicit manner. For that, we can use `get_widget_class`, which can accept a type and returns the widget that magicgui thinks you'd want:

```python
from magicgui.type_map import get_widget_class

get_widget_class(annotation = str)
(magicgui.widgets.LineEdit, {})
```

## pydantic & magicgui

Given the above behavior of magicgui, the approach I arrived at is to start with a magicgui `Container`, and then walk through the pydantic model fields, adding a new `Container` when encountering a nested pydantic object or otherwise using `get_widget_class` to add widgets to the container when the field. For a simple initial pydantic model without nested models, we can just iterate over the `.__fields__` attribute of the pydantic model: 

```python
import pydantic
from magicgui import widgets
from magicgui.type_map import get_widget_class

class SimpleModel(pydantic.BaseModel):
    field_1: str
    field_2: float
    
container = widgets.Container()

for field, field_info in SimpleModel.__fields__.items():
    new_widget_class, _ = get_widget_class(annotation = field_info.type_)
    container.append(new_widget_class(name=field))
    
container.show(run=True)       
```
 
![png](/images/magicguipydantic/widgets_03.png)

For a more complex traversal, we'll want to use a recursive function.

```python
def add_pydantic_to_container(py_model, container):
    # recursively traverse a pydantic model adding widgets to a container. When a nested
    # pydantic model is encountered, add a new nested container
    for field, field_def in py_model.__fields__.items():
        ftype = field_def.type_
        if isinstance(ftype, pydantic.BaseModel) or isinstance(ftype, pydantic.main.ModelMetaclass):
            # the field is a pydantic class, add a container for it and fill it
            new_widget_cls = widgets.Container
            new_widget = new_widget_cls(name=field_def.name)
            add_pydantic_to_container(ftype, new_widget)
        else:
            # parse the field, add appropriate widget
            new_widget_cls, ops = get_widget_class(None, ftype, dict(name=field_def.name, value=field_def.default))
            new_widget = new_widget_cls(**ops)
            if isinstance(new_widget, widgets.EmptyWidget):
                warnings.warn(message=f"magicgui could not identify a widget for {py_model}.{field}, which has type {ftype}")
        container.append(new_widget)
```      

The above function takes in a pydantic model class and a magicgui container and recursively walks through the pydantic model, adding to the container as it goes. It's very similar to the simple example, but when it encounters a pydantic field that is itself a model, it adds a container and then calls itself with the new container and that pydantic field. It also respects the default values of the pydantic fields! And now we can use the above function to map out more complex pydantic models: 

```python 
class ComplexModel(pydantic.BaseModel):
    simple: SimpleModel
    complex_field: int
    field_with_default: float = 10.0
    
container = widgets.Container()   

add_pydantic_to_container(ComplexModel, container)

container.show(run=True) 
```    

![png](/images/magicguipydantic/widgets_04.png)

And because we were assigning the widget names with the pydantic field names, we can add callbacks easily! For example: 

```python 

container = widgets.Container()   
add_pydantic_to_container(ComplexModel, container)
container.append(widgets.Label(name="result"))

def calculate_result():
    result = container.simple.field_2.value * 2
    result = result * container.field_with_default.value
    container.result.value = f"{result}" 
    
    
container.simple.field_2.changed.connect(calculate_result)  
container.show(run=True)
```

![png](/images/magicguipydantic/widgets_05.png)

## completing the round-trip 

So what I **really** off to do was generate a widget that could instantiate a pydantic class, and at this point we're half-way there. To complete the trip, all we need to do is take our `container` and traverse it again, pulling out the values that are set so that we can instantiate the pydantic class that was used to generate the widget! 

Going back to the simple case, let's add a callback to generate the class 

```python

class SimpleModel(pydantic.BaseModel):
    field_1: str
    field_2: float
    
container = widgets.Container()

for field, field_info in SimpleModel.__fields__.items():
    new_widget_class, _ = get_widget_class(annotation = field_info.type_)
    container.append(new_widget_class(name=field))
    
# add a button and a place to display a json from the pydantic model
container.append(widgets.PushButton(name="build", label="Build json"))
container.append(widgets.Label(name="json_display", value=""))    

def display_json():
    
    # build a dict for args to instantiate
    pydantic_kwargs = {}
    for field in SimpleModel.__fields__.keys():
        field_widget = getattr(container, field)
        pydantic_kwargs[field] = field_widget.value
        
    # instantiate the model!
    pydantic_model = SimpleModel.parse_obj(pydantic_kwargs)
    
    # pull out a json from the model, update the magicgui display
    json_for_display = pydantic_model.json(indent=4)    
    container.json_display.value = json_for_display

container.build.clicked.connect(display_json)  # note buttons have `clicked` instead of `changed`
container.show(run=True)  
```

![png](/images/magicguipydantic/widgets_06.png)

And to generalize this to a more complex case, we again can traverse the pydantic object and look for fields that are also pydantic objects. But in this case, we simply add a new dictionary to the `pydantic_kwargs` dictionary for each new level :

```python 
def get_pydantic_kwargs(container, pydantic_model, pydantic_kwargs):
    # given a container that was instantiated from a pydantic model, get the arguments
    # needed to instantiate that pydantic model from the container.

    # traverse model fields, pull out values from container
    for field, field_def in pydantic_model.__fields__.items():
        ftype = field_def.type_
        if isinstance(ftype, pydantic.BaseModel) or isinstance(ftype, pydantic.main.ModelMetaclass):
            # go deeper
            pydantic_kwargs[field] = {} # new dictionary for the new nest level
            # any pydantic class will be a container, so pull that out to pass
            # to the recursive call
            sub_container = getattr(container, field_def.name)
            get_pydantic_kwargs(sub_container, ftype, pydantic_kwargs[field])
        else:
            # not a pydantic class, just pull the field value from the container
            if hasattr(container, field_def.name):
                value = getattr(container, field_def.name).value
                pydantic_kwargs[field] = value
```

which we can now use to reverse our complex container construction:


```python
container = widgets.Container()   
add_pydantic_to_container(ComplexModel, container)

# add a button and a place to display a json from the pydantic model
container.append(widgets.PushButton(name="build", label="Build json"))
container.append(widgets.Label(name="json_display", value=""))    

def display_json():
    
    # build a dict for args to instantiate
    pydantic_kwargs = {}
    get_pydantic_kwargs(container, ComplexModel, pydantic_kwargs)
    # instantiate the model!
    pydantic_model = ComplexModel.parse_obj(pydantic_kwargs)
    
    # pull out a json from the model, update the magicgui display
    json_for_display = pydantic_model.json(indent=4)    
    container.json_display.value = json_for_display

container.build.clicked.connect(display_json)  # note buttons have `clicked` instead of `changed`
container.show(run=True)  

```

![png](/images/magicguipydantic/widgets_07.png)

A full script capturing everything above is available [here](https://github.com/chrishavlin/miscellaneous_python/blob/master/src/pydantic_magicgui_roundtrip.py).

## final thoughts

The main limitation to the above examples are that I'm only handling simple pydantic attributes that are simple types. To handle other attributes (e.g., tuples), I've played around with [registering new types with magicgui](https://napari.org/magicgui/usage/types_widgets.html#register-type) and also modifying the recursive functions to have my own custom mapping from fields to widgets to varying degrees of success. Also note that there may be easier ways (e.g., [magicclass](https://github.com/hanjinliu/magic-class) used in combination with magicgui?), but for my application I have separate pydantic model creation and model ingestion methods, so I don't mind the somewhat manual approach here, at least until full pydantic support is in magicgui!
