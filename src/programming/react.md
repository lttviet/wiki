# React

## `npx create-react-app some-app`

Ref: [github bug](https://github.com/facebook/create-react-app/issues/9091)

I'm using Win 10 and the username has a space (V L).

```cmd
C:\Users> dir /x    # got VL~1
```

`npm config edit` and add `cache=C:\Users\VL~1\AppData\Roaming\npm-cache` to config file.