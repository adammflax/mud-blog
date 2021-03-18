> :Title color=black
>
> My Nim Environment

> :Author src=github

Setting up my nim environment
---------------------
Inspired by a [post](https://sourcery.ai/blog/python-best-practices/) I found on setting up a python environment I've decided I'll write one for my nim environment.
This is not a guide on setting up Nim itself for that use [choosenim](https://github.com/dom96/choosenim). I am also presuming that you're using [nimble](https://github.com/nim-lang/nimble) along with the standard nimble project layout


Code Editor and plugins
-------------------
I currently use Visual Studio Code (VSC) as my editor however I've been meaning to try Intellij's Nim support.

I use the following plugins in VSC:

- Nim by Konstantin Zaitsev - gives me syntax highlighting and code completion
- Native Debug by WebFreak - allows me to integrate GDB with Visual Studio Code
- Git Graph by mhutchie - to see a graphical view of my git history. I think this is prettier then git graph [A Dog](https://stackoverflow.com/questions/1057564/pretty-git-branch-graphs)
- Intellij IDEA Keybindings by Keisuke Kato - I've used Intellij for years and have grown use to its keybindings

### Setting up the launch template
I've setup one single launch template for VSC that allows for quick debugging with GDB


```json | launch.json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug current Test",
            "type": "gdb",
            "request": "launch",
            "target": "./bin/tests/${fileBasenameNoExtension}.exe",
            "cwd": "${workspaceRoot}",
            "valuesFormatting": "parseText",
            "preLaunchTask": "debug",
        }
    ]
}
```

we also need to setup a tasks.json

```json | tasks.json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
      {
        "label": "debug",
        "type": "shell",
        "command": "nim c --debugger:native tests/${fileBasename}",
        "group": "test"
      }
    ]
  }
```


Now when I want to debug a test I simply need to press f5 in that test file to open the debugger!

Nim Configuration
-----------
Nim allows you to pass command line flags to the compiler via nim.cfg files.  I have 2 nim.cfg's setup one in the root of the project and one in the tests folder

```bash | nim.cfg
--gc:arc
--experimental:strictFuncs
--outdir:bin
```
For my projects I like to default to using the arc garbage collector. If your project needs a more specialized GC check out the [garbage colletion documentation](https://nim-lang.org/docs/gc.html)

Likewise I am a big fan of immutability so I've enabled the strictFuncs parameter for stricter checks on mutations.

> [warning](:Icon) Enabling `--experimental:strictFuncs`via a compiler flag will also activate this for the standard library. Parts of the standard library that you would not expect to
be flagged as mutating will be flagged such as the [json module](https://github.com/nim-lang/Nim/issues/17387). If you know your code won't mutate you tell the compiler to ignore a block of code by using the `{.cast(noSideEffect).}:`pragma

Finally I've set the outdir flag so that our executable is created in the bin directory. Please note that `nimble build` does not currently respect the `--outdir` flag!


```bash | tests/nim.cfg
--path: "../src/"
--outdir: "bin/tests"
```
We overwrite the outdir for tests so that the binaries generated for tests are put in a test folder to not clutter our workspace. 

Likewise we update our path so that we can import project modules without them needing to be in your `~/.nimble` directory. For a more detailed explanation on the path param check out
the nimble [documentation](https://github.com/nim-lang/nimble)



> [warning](:Icon) If you use [testament](https://nim-lang.github.io/Nim/testament.html) as your test runner then don't set the ```--outdir```flag as testament
will not be able to find the megatest executable. For more information see [here](https://forum.nim-lang.org/t/7577#48099)


Git hooks
------------
I use the [pre-commit](https://pre-commit.com/) tool to run scripts before I commit changes. Currently I've set it up to auto format my nim files and run tests on commit.
My config can be found below

```yaml | .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: format
        name: format
        files: "^.*nim$"
        stages:
          - commit
        language: system
        entry: nimpretty -maxLineLen:120
        types:
          - file
      - id: test
        name: test
        stages:
          - commit
        language: system
        entry: nimble test
        types:
          - file
```
You may want to adjust the `maxLineLen` parameter if you have a small amount of screen real estate.