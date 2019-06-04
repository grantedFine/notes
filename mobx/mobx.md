### async change in autoRun, reaction
```
  autoRun(() => {
    console.log('aba)
  }, {
    scheduler: f => {
      setTimeOut(f, 1000)
    }
  })
```

### observable.array use a key of an enum in computed, like this:

```
  arr = observable.array([...])

  @computed
  value(() => {
    return arr[0][keyName]
  })
```

If remove or replace the enum, you still get the old enum in the computed function.
