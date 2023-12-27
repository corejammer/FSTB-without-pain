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

## Trying it out
Let's install FSTB:
```
npm i fstb
```
And we'll plug it into the project:
```
const fstb = require('fstb');
```
To form a path to a file, you can use the FSPath function or one of the shortcuts: `cwd`, `dirname`, `home`, or `tmp` (see the documentation for more details). You can also pull paths from environment variables using the `envPath` method.

Reading text from a file:
```
fstb.cwd["README.md"]().asFile().read.txt().then(txt=>console.log(txt));
```
FSTB operates on promises, so you can use it with async/await in your code:
```
(async function() {
  const package_json = await fstb.cwd["package.json"]().asFile().read.json();
  console.log(package_json);
})();
```
Here we deserialize JSON from a file. In my opinion, it's concise; with one line, we explain where it is, what's inside, and what to do with it.

If I were to write this using standard functions, it would look something like this:
```
const fs = require("fs/promises");
const path = require("path");

(async function() {
  const package_json_path = path.join(process.cwd(), "package.json");
  const file_content = await fs.readFile(package_json_path, "utf8");
  const result = JSON.parse(file_content);
  console.log(result);
})();
```
This is certainly not the code to be proud of, but in this example, you can see how verbose working with files can be using the standard library.

Another example. Let's say you need to read a text file line by line. I don't even need to come up with an example; here's one from the Node.js documentation:
```
const fs = require('fs');
const readline = require('readline');

async function processLineByLine() {
  const fileStream = fs.createReadStream('input.txt');

  const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity
  });
  // Note: we use the crlfDelay option to recognize all instances of CR LF
  // ('\r\n') in input.txt as a single line break.

  for await (const line of rl) {
    // Each line in input.txt will be successively available here as `line`.
    console.log(`Line from file: ${line}`);
  }
}
processLineByLine();
```
Now let's try to do this using FSTB:
```
(async function() {
  await fstb.cwd['package.json']()
    .asFile()
    .read.lineByLine()
    .forEach(line => console.log(`Line from file: ${line}`));
})();
```

Yes, yes, I'm a cheater. The library has this function, and under the hood, it uses the same code from the documentation. But what's interesting here is that its output implements an iterator that can do `filter`, `map`, `reduce`, etc. So, if you need to, for example, read CSV, just add `.map(line => line.split(','))`.
