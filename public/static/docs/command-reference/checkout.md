# checkout

Update data files and directories in the <abbr>workspace</abbr> based on current
DVC-files.

## Synopsis

```usage
usage: dvc checkout [-h] [-q | -v] [-d] [-f] [-R]
                    [targets [targets ...]]

positional arguments:
  targets          DVC-files to checkout. Optional. (Finds all
                   DVC-files in the workspace by default.)
```

## Description

[DVC-files](/doc/user-guide/dvc-file-format) in a <abbr>project</abbr> specify
which instance of each data file or directory should be used, with the checksums
saved in the `outs` field. The `dvc checkout` command updates the workspace data
to match with the <abbr>cached</abbr> files that correspond to those checksums.

Using an SCM like Git, the DVC-files are kept under version control. At a given
branch or tag of the SCM repository, the DVC-files will contain checksums for
the corresponding data files kept in the cache. After an SCM command like
`git checkout` is run, the DVC-files will change to the state at the specified
branch or commit or tag. Afterwards, the `dvc checkout` command is required in
order to synchronize the data files with the currently checked out DVC-files.

This command must be executed after `git checkout` since Git doesn't track files
that are under DVC control. For convenience a Git hook is available, simply by
running `dvc install`, that will automate running `dvc checkout` after
`git checkout`. See `dvc install` for more information.

The execution of `dvc checkout` does:

- Scan the `outs` entries in DVC-files to compare with the currently checked out
  data files. The scanned DVC-files is limited by the listed `targets` (if any)
  on the command line. And if the `--with-deps` option is specified, it scans
  backward from the given `targets` in the corresponding
  [pipeline](/doc/command-reference/pipeline).

- For any data files where the checksum doesn't match their DVC-file entry, the
  data file is restored from the cache. The link strategy used (`reflink`,
  `hardlink`, `symlink`, or `copy`) depends on the OS and the configured value
  for `cache.type` – See `dvc config cache`.

Note that this command by default tries NOT to copy files between the cache and
the workspace, using reflinks instead when supported by the file system. (Refer
to
[File link types](/doc/user-guide/large-dataset-optimization#file-link-types-for-the-dvc-cache).)
The next linking strategy default value is `copy` though, so unless other file
link types are manually configured in `cache.type` (using `dvc config`), files
will be copied. Keep in mind that having file copies doesn't present much of a
negative impact unless the project uses very large data (several GBs or more).
But leveraging file links is crucial for large files where checking out a 50Gb
by copying file might take a few minutes for example, whereas with links,
restoring any file size will be almost instantaneous.

> When linking files takes longer than expected (10 seconds for any one file)
> and `cache.type` is not set, a warning will be displayed reminding users about
> the faster link types available. These warnings can be turned off setting the
> `cache.slow_link_warning` config option to `false` with `dvc config cache`.

The output of `dvc checkout` does not list which data files were restored. It
does report removed files and files that DVC was unable to restore because
they're missing from the <abbr>cache</abbr>.

This command will fail to checkout files that are missing from the cache. In
such a case, `dvc checkout` prints a warning message. Any files that can be
checked out without error will be restored.

There are two methods to restore a file missing from the cache, depending on the
situation. In some cases a pipeline must be reproduced (using `dvc repro`) to
regenerate its outputs. (See also `dvc pipeline`.) In other cases the cache can
be pulled from remote storage using `dvc pull`.

## Options

- `-d`, `--with-deps` - determine files to update by tracking dependencies to
  the target DVC-files (stages). This option only has effect when one or more
  `targets` are specified. By traversing all stage dependencies, DVC searches
  backward from the target stages in the corresponding pipelines. This means DVC
  will not checkout files referenced in later stages than the `targets`.

- `-R`, `--recursive` - `targets` is expected to contain at least one directory
  path for this option to have effect. Determines the files to checkout by
  searching each target directory and its subdirectories for DVC-files to
  inspect.

- `-f`, `--force` - does not prompt when removing workspace files. Changing the
  current set of DVC-files with `git checkout` can result in the need for DVC to
  remove files that don't match those DVC-file references or are missing from
  cache. (They are not "committed", in DVC terms.)

- `-h`, `--help` - shows the help message and exit.

- `-q`, `--quiet` - do not write anything to standard output. Exit with 0 if no
  problems arise, otherwise 1.

- `-v`, `--verbose` - displays detailed tracing information from executing the
  `dvc pull` command.

## Examples

Let's employ a simple <abbr>workspace</abbr> with some data, code, ML models,
pipeline stages, as well as a few Git tags, such as our
[get started example repo](https://github.com/iterative/example-get-started).
Then we can see what happens with `git checkout` and `dvc checkout` as we switch
from tag to tag.

<details>

### Click and expand to setup the project

Start by cloning our example repo if you don't already have it:

```dvc
$ git clone https://github.com/iterative/example-get-started
$ cd example-get-started
```

</details>

The workspace looks almost like in this
[pipeline setup](/doc/tutorials/pipelines):

```dvc
.
├── data
│   └── data.xml.dvc
├── evaluate.dvc
├── featurize.dvc
├── prepare.dvc
├── train.dvc
└── src
    └── <code files here>
```

We have these tags in the repository that represent different iterations of
solving the problem:

```dvc
$ git tag
baseline-experiment     <- first simple version of the model
bigrams-experiment      <- use bigrams to improve the model
```

This project comes with a predefined HTTP
[remote storage](/doc/command-reference/remote). We can now just run `dvc pull`
that will fetch and checkout the most recent `model.pkl`, `data.xml`, and other
files that are under DVC control. The model file checksum
`3863d0e317dee0a55c4e59d2ec0eef33` will be used in the `train.dvc`
[stage file](/doc/command-reference/run):

```dvc
$ dvc pull
...
Checking out model.pkl with cache '3863d0e317dee0a55c4e59d2ec0eef33'
...

$ md5 model.pkl
MD5 (model.pkl) = 3863d0e317dee0a55c4e59d2ec0eef33
```

What if we want to rewind history, so to speak? The `git checkout` command lets
us checkout at any point in the commit history, or even checkout other tags. It
automatically adjusts the files, by replacing file content and adding or
deleting files as necessary.

```dvc
$ git checkout baseline
Note: checking out 'baseline'.
...
HEAD is now at 40cc182...
```

Let's check the `model.pkl` entry in `train.dvc` now:

```yaml
outs:
  md5: a66489653d1b6a8ba989799367b32c43
  path: model.pkl
```

But if you check `model.pkl`, the file hash is still the same:

```dvc
$ md5 model.pkl
MD5 (model.pkl) = 3863d0e317dee0a55c4e59d2ec0eef33
```

This is because `git checkout` changed `featurize.dvc`, `train.dvc`, and other
DVC-files. But it did nothing with the `model.pkl` and `matrix.pkl` files. Git
doesn't track those files, DVC does, so we must do this:

```dvc
$ dvc fetch
$ dvc checkout
$ md5 model.pkl
MD5 (model.pkl) = a66489653d1b6a8ba989799367b32c43
```

What happened is that DVC went through the sole existing DVC-file and adjusted
the current set of files to match the `outs` of that stage. `dvc fetch` is run
once to download missing data from the remote storage to the <abbr>cache</abbr>.
Alternatively, we could have just run `dvc pull` in this case to automatically
do `dvc fetch` + `dvc checkout`.

## Automating `dvc checkout`

We have the data files (managed by DVC) lined up with the other files (managed
by Git). This required us to remember to run `dvc checkout`, and of course we
won't always remember to do so. Wouldn't it be nice to automate this?

Let's run this command:

```dvc
$ dvc install
```

This installs Git hooks to automate running `dvc checkout` (or `dvc status`)
when needed. Then we can checkout the master branch again:

```dvc
$ git checkout bigrams
Previous HEAD position was d171a12 add evaluation stage
HEAD is now at d092b42 try using bigrams
Checking out model.pkl with cache '3863d0e317dee0a55c4e59d2ec0eef33'.

$ md5 model.pkl
MD5 (model.pkl) = 3863d0e317dee0a55c4e59d2ec0eef33
```

Previously this took two steps, `git checkout` followed by `dvc checkout`. We
can now skip the second one, which is automatically executed for us. The
workspace is automatically synchronized accordingly.
