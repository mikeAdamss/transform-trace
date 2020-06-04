
### Transform Tracing

The transform tracer allows you to document what you've done in regards to creating datacubes by capturing simple annotations in context.


### Initialising Transform Tracing
You initialise the tracer (once) via:
```python
trace = TransformTrace()
```


### Starting a Trace (of each tab/dataframe)

For each extracted tab, you need to start a trace that tracks changes and choices made in regards to **that particular tab** via:

```python
trace.start(<name of output>, <databaker tab or label of tab>, <list of columns>, <unique source identifier>)
```

example usage:
```python
columns=["Geography", "Time", "Info", "Interesting Stuff"]
trace.start("My Output 1", tab, columns, "path-to-my-source.xls")
```

Note:
- I'm using a databaker tab object in the example but as long as `<databaker tab or label of tab>` + `<unique source identifier>` forms a unique composite key you can pass in whatever you like.
- You can pass in as many columns as you like as `<list of columns>`, so if you're adding a column called `my_column` (even if that's your very last action) there's no reason not to declare it at the start.

that said, should you need to you can add column entries to the trace dynamically via `trace.add_column("my_column")`

### Adding comments

You can add a comment against a single column with the syntax`trace.<COLUMN_ID>(<comment as a string>)`

`COLUMN_ID` is always a column name you've passed in (with spaces replaces with underscores). So using our trace starting example (scroll up) the following would all be viable:

```python
trace.Geography("I am a comment")
trace.Time("I am a comment")
trace.Info("I am a comment")
trace.Interesting_Stuff("I am a comment")
```

If you need to add a comment against more than one column at the same time you can do it via:
```python
trace.multi(["Time", "Geography"], "Some sort of comment relevant to both Time and Geography")
```

If you want to add a comment against **all** the columns in a given dataframe you can do:
```python
trace.ALL("Something relevent to all columns of this dataframe.")
```

If you want to pass single comments that make use of variables you can do so with the `var` keyword argument.
```python
trace.Time("I am hardcoding this to {}", var="2020")
```

### Databaker: Dimensions and Previews

You can use the tracer to pass variables to databaker directly (to avoid redeclaring string variables - DRY etc), example:

```python
dimensions = [
        HDim(geography, trace.Geography.label, DIRECTLY, LEFT),
        HDimConst(trace.Time.label, trace.Time.var)
    ]
```

you generate a databaker preview html as follows:

```python
cs = ConversionSegment(observations, dimensions)
trace.with_preview(cs)
```


### Dataframes Derrived From Other Dataframes

To create the trace we need to keep track of where one dataframe is derrived from others. We do this with the `.store` method.

To store a dataframe we essentially "save it for later" against a chosen identifier, usage:

`trace.store(<identifier>, <the dataframe you're storing>)`

example

```python
trace.store("my_identifier", dataframe)
```

So when you want to combine everything you've stored under an identifier and start tracing events against that new dataframe you do:

`trace.combine_and_trace(<name of output>, <store identifier>)`

```python
df = trace.combine_and_trace("My Output 1", "my_identifier")
# then you can comment against "my_identifier" as if it were a tab, eg
trace.ALL("Something relevent to our combined dataframe")
```
