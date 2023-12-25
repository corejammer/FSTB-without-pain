# FSTB – File Handling in Node.js Without Pain

When working with files in Node.js, I often find myself writing a lot of repetitive code. Creating, reading and writing, moving, deleting, traversing files and subdirectories – all of this accumulates an incredible amount of boilerplate, further exacerbated by the peculiar names of functions in the fs module. While one can live with all this, the thought of making things more convenient has always lingered in my mind. I wished for simple tasks, like reading or writing text (or JSON) to a file, to be achievable in just one line.

As a result of these musings, the FSTB library was born, where I attempted to improve interactions with the file system. Whether I succeeded or not, you can determine by reading this article and trying out the library in action.

## Background
Working with files in Node.js goes through several stages: denial, anger, bargaining... first, we obtain a path to a file system object in some way, then we check its existence (if necessary), and finally, we work with it. Working with paths in Node.js is generally encapsulated in a separate module. The coolest function for working with paths is `path.join`. It's a really cool thing that, when I started using it, saved me a bunch of nerve cells.

However, there is a problem with paths. A path is a string, even though it essentially describes the location of an object in a hierarchical structure. Since we're dealing with an object, why not use the same mechanisms as when working with regular JavaScript objects?

The main problem is that a file system object can have any name with allowed characters. If I create methods for working with it, for example, this code: `root.home.mydir.unlink` will be ambiguous – what if there is a directory called `unlink` inside `mydir`? What then? Do I want to delete `mydir` or refer to `unlink`?

Once I experimented with JavaScript Proxy and came up with an interesting construction:

```
const FSPath = function(path: string): FSPathType {
  return new Proxy(() => path, {
    get: (_, key: string) => FSPath(join(path, key)),
  }) as FSPathType;
};
```
Here, `FSPath` is a function that takes a string representing a file path as input and returns a new function, closing over this path and wrapped in a Proxy. The Proxy intercepts all property accesses to the resulting function and returns a new `FSPath` function with the appended property name as a segment. At first glance, it may seem strange, but in practice, this construction allows for interesting ways of constructing paths:
```
FSPath(__dirname).node_modules //работает аналогично path.join(__dirname, "node_modules")
FSPath(__dirname)["package.json"] //работает аналогично path.join(__dirname, "package.json")
FSPath(__dirname)["node_modules"]["fstb"]["package.json"] //работает аналогично path.join(__dirname, "node_modules", "fstb", "package.json")
```
As a result, we obtain a function that, when invoked, returns the constructed path. For example:
```
const package_json = FSPath(__dirname).node_modules.fstb["package.json"]
console.log(package_json()) // <путь к скрипту>/node_modules/fstb/package.json
```
Here again, what's so special, just ordinary JavaScript tricks. But then I thought – instead of returning just a path, why not return an object that has all the necessary methods for working with files and directories.

So, the FSTB library was born – it stands for **F**ile**S**ystem **T**ool**B**ox.
