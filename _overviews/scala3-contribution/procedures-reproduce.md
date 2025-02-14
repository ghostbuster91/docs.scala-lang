---
title: Reproducing an Issue
type: section
description: This page describes reproducing an issue in the Scala 3 compiler.
num: 5
previous-page: procedures-cheatsheet
next-page: procedures-navigation
---

An issue found in the [GitHub repo][lampepfl/dotty] usually describes some code that
manifests undesired behaviour.

To try fixing it, we first need to reproduce the issue, so that
- we can understand its cause
- we can verify that any changes made to the codebase have a positive impact on the issue.

Say you want to reproduce locally issue [#7710], we would copy the code from the *"Minimised Code"*
section of the issue to a file named e.g. `local/i7710.scala`,
and then try to compile it from the sbt console opened in the dotty root directory:
```bash
$ sbt
sbt:scala3> scala3/scalac -d local/out local/i7710.scala
```
> Here, the `-d` flag specifies a directory `local/out` where generated code will be output.

You can then verify that the local reproduction has the same behaviour as originally reported in the issue.
If so, then we can get to trying to fix it, else, perhaps the issue is out of date, or
is missing information about how to accurately reproduce the issue.

## Dotty Issue Workspace

Sometimes we need more complex commands to reproduce an issue, and it is useful to script these, which
can be done with [dotty-issue-workspace]. It allows to bundle sbt commands for issue reproduction in one
file and then run them from the Dotty project's sbt console.

### Try an Example Issue

Let's use [dotty-issue-workspace] to reproduce issue [#7710]:
1.  Follow [steps in README][workspace-readme] to install the plugin.
2.  In your Issue Workspace directory (as defined in the plugin's README file,
    "Getting Started" section, step 2), create a subdirectory for the
    issue: `mkdir i7710`.
3.  Create a file for the reproduction: `cd i7710; touch Test.scala`. In that file,
    insert the code from the issue.
4.  In the same directory, create a file `launch.iss` with the following content:
    ```bash
    $ (rm -rv out || true) && mkdir out # clean up compiler output, create `out` dir.

    scala3/scalac -d $here/out $here/Test.scala
    ```

    - The first line, `$ (rm -rv out || true) && mkdir out` specifies a shell command
      (it starts with `$`), in this case to ensure that there is a fresh `out`
      directory to hold compiler output.
    - The next line, `scala3/scalac -d $here/out $here/Test.scala` specifies an sbt
      command, which will compile `Test.scala` and place any output into `out`.
      `$here` is a special variable that will be replaced by the path of the parent
      directory of `launch.iss` when executing the commands.
5.  Now, from a terminal we will run the issue from sbt in the dotty directory
    ([See here][clone] for a reminder if you have not cloned the repo.):
    ```bash
    $ sbt
    sbt:scala3> issue i7710
    ```
    This will execute all the commands in the `i7710/launch.iss` file one by one.
    If you've set up `dotty-issue-workspace` as described in its README,
    the `issue` task will know where to find the folder by its name.

### Using Script Arguments

You can use script arguments inside `launch.iss` to reduce steps when
working with issues.

Say you have an issue `foo`, with two alternative files that are very similar
`original.scala`, which reproduces the issue and `alt.scala`, which does not,
and you want to compile them selectively?

You can achieve this via the following `launch.iss`:

```bash
$ (rm -rv out || true) && mkdir out # clean up compiler output, create `out` dir.

scala3/scalac -d $here/out $here/$1.scala # compile the first argument following `issue foo <arg>`
```

It is similar to the previous example, except we now compile a file `$1.scala`, referring
to the first argument passed after the issue name. The command invoked would look like
`issue foo original` to compile `original.scala`, and `issue foo alt` for `alt.scala`.

In general, you can refer to arguments passed to the `issue <issue_name>` command using
the dollar notation: `$1` for the first argument, `$2` for the second and so on.

### Multiline Commands

`launch.iss` files support putting commands accross multiple lines, which is useful for
toggling lines by using a comment.

The following `launch.iss` file is a useful template for issues that run code after
compilation, it also includes some debug compiler flags, commented out.
The advantage of having them is, if you need one them, you can enable it quickly by
uncommenting it – as opposed to looking it up and typing it in your existing command.
Put your favourite flags there for quick usage.

```bash
$ (rm -rv out || true) && mkdir out # clean up compiler output, create `out` dir.

scala3/scalac  # Invoke the compiler task defined by the Dotty sbt project
  -d $here/out  # All the artefacts go to the `out` folder created earlier
  # -Xprint:typer  # Useful debug flags, commented out and ready for quick usage. Should you need one, you can quickly access it by uncommenting it.
  # -Ydebug-error
  # -Yprint-debug
  # -Yprint-debug-owners
  # -Yshow-tree-ids
  # -Ydebug-tree-with-id 340
  # -Ycheck:all
  $here/$1.scala  # Invoke the compiler on the file passed as the second argument to the `issue` command. E.g. `issue foo Hello` will compile `Hello.scala` assuming the issue folder name is `foo`.

scala3/scala -classpath $here/out Test  # Run the class `Test` generated by the compiler run (assuming the compiled issue contains such an entry point, otherwise comment this line)
```

## Conclusion

In this section, we have seen how to reproduce an issue locally, next we will see
how to try and detect its root cause.

[lampepfl/dotty]: https://github.com/lampepfl/dotty/issues
[#7710]: https://github.com/lampepfl/dotty/issues/7710
[dotty-issue-workspace]: https://github.com/anatoliykmetyuk/dotty-issue-workspace
[workspace-readme]: https://github.com/anatoliykmetyuk/dotty-issue-workspace#getting-started
[clone]: {% link _overviews/scala3-contribution/start-intro.md %}#clone-the-code
