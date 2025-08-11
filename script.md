# Python's pathlib module - 25 minute talk script

## Intro (1 minute)

- Quick introduction
  - I'm Trey, I teach Python, and I've been using pathlib for years
  - Today I want to convince you that pathlib should be your go-to for file path handling
  - This isn't just about replacing `os.path` - it's about writing better, more readable code

## Stringly typed path code (3 minutes)

### The tradition

- It is *very* common to use strings to represent file paths, in Python and in other programming languages
- Show example of traditional approach:

```python
import os.path

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
TEMPLATES_DIR = os.path.join(BASE_DIR, "templates")
```

This code has worked for decades... and it will probably *always* work.

But, there are some downsides to this code:

- Reading nested function calls from inside-out is painful
- When using strings that represent paths, it's easy to make mistakes with path separators (especially on Windows)


```python
import os.path

BASE_DIR = os.path.abspath(__file__).rsplit("/")[0]
TEMPLATES_DIR = BASE_DIR  + "/templates"
```

Also, there's no easy way to make a clear distinction "a string that represents a path" and "just a string". That means:

- When using type annotations, everything is just `str`
- There's no semantic difference between a filename and any other string
- Which could make it harder to catch path-related bugs with static analysis or even when debugging

```python
license: str  # Is this a filename or the contents of a file?

def find_editorconfig_file() -> str:
    ...  # Does this return a file path or file contents?
```

## Python supports a better way (3 minutes)

### Introducing `pathlib.Path`

- Python has a `pathlib` module that includes a `Path` class which is specifically meant for representing file paths
- TODO Here's code that uses `pathlib.Path` to ...

Okay, but what's the selling point?

Well, the main reason *I* reach for `pathlib` is that it consolidates *lots* of different path-related functionality into one class.
So if I have a `pathlib.Path` object and I want to ask questions about it, I can look up help on that object to find out which attribute or method that I need. There's no extra imports, no digging through different standard library modules looking for the answer... it's all either an attribute or a method directly on the `Path` object.

TODO show an example

### Why use `pathlib.Path` objects if we'll just need to convert them back to strings?

But doesn't using `Path` objects require a bit of commitment?

Once we need to *use* the path for something, won't we need to convert it back into a string?

Well, probably not.

Python's `open` function accepts `Path` objects:

```python
from pathlib import Path

path = Path("example.txt")
with open(path) as file:
    contents = file.read()
```

This has worked since Python 3.6 and Python 3.5 was end-of-life'd in September of 2020

And it's not just the `open` function that accepts `Path` objects.

### Pretty much every filepath-accepting standard library utility accepts `pathlib.Path` objects

Every standard library module I use that accepts a string representing a file also accepts a `Path` object.

- `shutil.copy()` accepts `pathlib.Path` objects
- `shutil.move()` does too
- All the `os` module's many path-related functions:
    - `os.makedirs()`
    - `os.rename()`
- The various `os.path` functions for manipulating file path strings also accept `Path` objects
- Even `subprocess.run()` will accept a `pathlib.Path` object

```python
import subprocess
subprocess.run([sys.executable, Path("my_script.py")])
```

### Pretty much every filepath-accepting third-party library utility accepts `pathlib.Path` objects

Most third-party libraries that work with file paths accept either strings or `Path` objects.

- Any library that *properly* handles file paths will work with Path objects
- If a library does not accept `Path` objects, I would argue that it doesn't properly handle file paths
- In other words, if `pathlib.Path` objects don't work, there's probably a bug with that library

I *very* rarely convert Path objects to strings.

## But why? (7 minutes)

### `pathlib` does not replace file objects, only file path strings

- Important distinction: we're not replacing the `open()` function or file objects
- We're replacing various utilities that:
    - ask questions of file paths
    - grab pieces of file parts and that construct file paths
    - locate files
    - copy, move, and remove files
- Path objects *represent* file paths, but they don't *contain* file contents

### Common `os.path` and other pre-`pathlib` idioms shown using `pathlib` instead

#### Path joining - the old way vs the new way

```python
# Old way - nested function calls
import os.path

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
TEMPLATES_DIR = os.path.join(BASE_DIR, "templates")

# New way - chaining attributes and method calls
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
TEMPLATES_DIR = BASE_DIR / "templates"
```

#### Creating directories and files

```python
# Old way
import os
import os.path

base = "src"
os.makedirs(os.path.join(base, "__pypackages__"), exist_ok=True)
os.rename(".editorconfig", os.path.join(base, ".editorconfig"))

# New way
from pathlib import Path

base = Path("src")
(base / "__pypackages__").mkdir(parents=True, exist_ok=True)
Path(".editorconfig").rename(base / ".editorconfig")
```

#### Reading and writing files

```python
# Old way
with open("config.txt", mode="rt") as file:
    content = file.read()

with open("output.txt", mode="wt") as file:
    file.write("Hello world")

# New way (for simple cases)
content = Path("config.txt").read_text()
Path("output.txt").write_text("Hello world")

# Of course, this works too
config_path = Path("config.txt")
with open(config_path, mode="rt") as file:
    content = file.read()

output_path = Path("output.txt")
with open(output_path, mode="wt") as file:
    file.write("Hello world")
```

#### Finding files with glob

```python
# Old way
from glob import glob
csv_files = glob("**/*.csv", recursive=True)

# New way
from pathlib impoch Path
csv_files = list(Path.cwd().rglob("*.csv"))
```

### Do we even need `shutil` anymore?!

- Well, yes and no
- `pathlib.Path` objects include the methods `mkdir()`, `rmdir()`, `unlink()`, and `rename()`
- Python 3.14 adds new `copy()` and `move()` methods to `pathlib.Path` along with `copy_into()` `move_into()`
- As of Python 3.13, `shutil.copy()` and `shutil.move()` are still needed, but once you upgrade to Python 3.14, you may never need to import `shutil` again

## Okay, but how? (6 minutes)

### `pathlib`-ifying code that may or may not be un-`Path`'ed

Let's say you *don't* yet use `pathlib`... how should you go about adopting `pathlib`? Do you need to change everything all at once?

- You don't! You can *gradually* adopt pathlib.
- `Path` objects work everywhere strings work
- You can convert any path-like object (PEP 519 terminology we'll get to later) to a `Path` object:

```python
def process_path(path_or_string):
    path = Path(path_or_string)  # Works with both strings and Path objects
    return path.resolve()
```

The `Path` class accept strings *and* other `Path` objects, so if you don't know whether a given object is a `Path` object yet, you can always pass it to the `Path` class to turn it into one.

### The many flavors of `pathlib` usage: `Path`, `joinpath`, `/`, and other alternatives

There are many different ways to join paths with `pathlib`.

For example, let's say we have a `pathlib.Path` object representing our home directory:

```python
from pathlib import Path

home = Path.home()
```

To create a `Path` representing a `.config.toml` file in our home directory, we could use the `joinpath` method:

```python
config1 = home.joinpath(".config.toml")
```

Or the `/` operator:

```python
config2 = home / ".config.toml"
```

`Path` objects overload Python's division operator so that it does a join operation when used between a multiple `Path` objects or between a `Path` object and a string.

We could also pass the `Path` object to the `Path` class along with other path segments to join with it:

```python
config3 = Path(home, ".config.toml")
```

This approach is especially helpful if the `home` variable might be passed into our code from code that we don't control and we don't know whether it's a string or a `Path` object.

```python
# All equivalent!
assert config1 == config2 == config3
```

### Using `pathlib` when working with a single file path

What if you just need to work with a single file path?

Is it worth it to use `pathlib` for just one file path?

I think it often *is*.

When I plan to read the entire contents of a file at once, I would rather do this:

```python
from pathlib import Path

contents = Path("some_file.txt").read_text()
```

Than this:

```python
with open("some_file.txt") as file:
    contents = file.read()
```

Also, even with just one file, it's convenient that with a `Path` object it's easy to get the absolute path for the sake of printing the full path for an end user:

```python
from pathlib import Path

path = Path("some_file.txt").resolve()
print(f"Reading {path!r}...")

contents = path.read_text()
```

And if we wanted to get *just* the name of the file or the directory that the file lives within, that's easy also:

```python
from pathlib import Path

config_path = Path(".config").resolve()

print(f"Reading {config_path.name}!r from {config.path.parent!r}...")
```

And if we need to do something special based on the file extension or some other feature of the filename, `pathlib` makes that easier:

```python
if user_path.suffix == ".py":
    print("Handling the given Python file...")
```

Also using `pathlib` instead of a string that represents a filename is just *one* extra import:

```python
from pathlib import Path
```


### Using `pathlib.Path` with `argparse` and friends

But... how *should* file paths be accepted from users?

Well, it depends on how we're accepting inputs from a user.

Let's say we're using the `argparse` module to accept command-line arguments.

If we wanted to open a file immediately, without ever displaying or processing the file path ourselves, we could use `argparse.FileType`:

TODO show

But what if we want to accept either a directory name or a filename?

Instead of using `argparse.FileType`, we could specify an argument type of `pathlib.Path`:

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("path", type=Path)
args = parser.parse_args()

if args.path.is_dir():
    ...  # Directory given
else args.path.is_file():
    ...  # Existing file given
else:
    ...  # Path doesn't represent a file or a directory!
```

Since `argparse` will call whatever type is given to it, we'll end up with a new `pathlib.Path` object that represents the user's given filename.


### Embracing the iterators and generators of `pathlib`

TODO explanation of the various methods that return iterators and how that's often a nicer default than returning a list... though you can always do a list conversion. Compare each of the approaches to the equivalent non-pathlib list and iterator approaches (glob.glob, glob.iglob, etc.)

```python
# Find all Python files in current directory and subdirectories
for py_file in Path.cwd().rglob('*.py'):
    print(f"Processing {py_file}")
    
# List all directories
for item in Path.cwd().iterdir():
    if item.is_dir():
        print(f"Directory: {item.name}")

# Walk the entire tree (like os.walk)
for path, subdirs, files in Path.cwd().walk():
    print(f"In {path}: {len(files)} files")
```

## This isn't just about `pathlib` (3 minutes)

### Not good enough for you? Well, `pathlib` usage isn't really just about `pathlib`

If the `pathlib` module seems like it's not quite *good enough* for your needs, keep in mind that:

- You can actually extend the functionality of `pathlib`
- Or... you can completely replace `pathlib` with your own library while getting all the benefits of `pathlib`

#### The ability to extend `pathlib.Path` functionality through inheritance

What if wished that `Path` objects had a method to easily change directories?

We could make our own class that inherits from `pathlib.Path` and add a `chdir` method to it:

```python
import os
import pathlib


class BetterPath(pathlib.Path):
    def chdir(self):
        os.chdir(self)
        return self.cwd()
```

Now, instead of using `pathlib` directly, our code could use this new enhanced `BetterPath` class with its `chdir` method:

```pycon
>>> BetterPath.home().chdir()
BetterPath('/home/trey')
```

If we got really accustomed to using `os.path.commonprefix` and we wish that that *exact* some functionality existed in the `pathlib.Path` class, we could add a method for that:

```python
import os.path
import pathlib


class BetterPath(pathlib.Path):
    def common_prefix(self, other):
        return self.with_segments(os.path.commonprefix((self, other)))
```

If we wanted to enhance the `rmdir` method allow `recursive` directory removal, we could do that:

```python
import pathlib
import shutil


class BetterPath(pathlib.Path):
    def rmdir(self, *, recursive=False, ignore_errors=False):
        if recursive:
            shutil.rmtree(self, ignore_errors=ignore_errors)
        else:
            try:
                super().rmdir()
            except Exception:
                if not ignore_errors:
                    raise
```

This all works as Python 3.12.
Before that, extending `pathlib` wasn't easy.

But now extending `pathlib` *is* officially supported.

There's even a `with_segments` method that subclasses can use to pass extra metadata to *any* derived paths:

TODO show


#### Python's built-in support for third-party `pathlib` alternatives thanks to its support for Path-like objects (`__fspath__`)

So `pathlib` can be extended, but what if you would actually prefer a very different path-based system?

Well PEP 519 defines the `os.PathLike` protocol.

This protocol is the reason that lots of functions accept either `pathlib.Path` objects *or* strings that represent file paths.

But this protocol isn't really about `pathlib`... it's about a `__fspath__` method.

- Any object that implements a `__fspath__()` method is a path-like object... which means such an object will work with the built-in `open` function and any other function that currently accepts either a `pathlib.Path` object or a string
- `__fspath__` just needs to return a string representing a file path
- Libraries like `plumbum`, `path.py` benefit from this standardization

TODO show good example of a popular third-party non-pathlib-based library and using its path as a path-like object with `open` or something


## pathlib anti-patterns

### Using the `open` method

Back in the early days of `pathlib`, before Python 3.6, if you wanted to open a file path, you could use the `open` method on the `pathlib.Path` class:

```python
from pathlib import Path

path = Path("example.txt")
with path.open() as file:
    contents = file.read()
```

I recommend *against* using the `open` method.
Instead, use the built-in `open` function:

```python
from pathlib import Path

path = Path("example.txt")
with open(path) as file:
    contents = file.read()
```

I see the `open` method as a relic that only exists because when `pathlib` was first added in Python 3.4, the `open` function did not yet accept `pathlib.Path` objects. You can think of the `open` method as just a wrapper around the `open` function.

### Using `joinpath` instead of passing arguments to `Path`

TODO explain `config = Path(directory).joinpath(".editorconfig")` or `config = Path(directory) / ".editorconfig"` versus `config = Path(directory, ".editorconfig")`

### TODO ??? another anti-pattern

TODO


## Outro (3 minutes)

### "But `pathlib` is slow"... but is it really? How slow? Does it matter?
- Yes, pathlib can be slower than os.path for some operations
- In my testing: 2x slower for finding .py files, 4x slower for all files
- But context matters: I searched 600,000 files and lost 6 seconds
- Don't optimize parts of your code that aren't bottlenecks
- Most path operations aren't in tight loops
- Readability usually trumps micro-optimizations

```python
# If you really need the performance in a tight loop:
from os import walk
for root, dirs, files in walk('/some/path'):
    # Use the faster approach where it matters
    pass

# But for most code:
for file_path in Path('/some/path').rglob('*'):
    # Use the readable approach everywhere else
    pass
```

### Use `pathlib`. You won't regret it.
- More readable code
- Cross-platform compatibility built-in
- Type safety with modern Python
- Less prone to path separator bugs
- Rich API for common operations
- Future-proof - new Python versions keep adding features
- You can start using it today without changing existing code

**Key takeaway**: Don't think of pathlib as "just another way to do paths." Think of it as "the right way to represent file paths as objects instead of strings."
