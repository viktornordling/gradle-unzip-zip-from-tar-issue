# Repo for showing issue when trying to use a file from another task

# The background
In the beginning, we had a zip file which we wanted to filter. Let's say it containts two
files, a.txt and b.txt, but we want to exclude b.txt from the zip file.

## The goal
Now, the "business case" has changed, and we have the .zip file inside a .tgz file. We want
to unpack the tar file, unzip the zip file and then process the files inside the zip.

## example-0
This example shows that things worked as expected before the .tgz file.
```
0. cd example-0
1. gw clean unzipAndRezipZip
2. tree
.
├── build
│   └── zipOutput
│       └── rezipped.zip
```

## example-1
The first idea was to have a task `extractTar` which precedes the `unzipAndRezipZip` task, which
extracts the zip file from the tar file.

This example is using Gradle 4.2.1, because this is what the real project was on when this
adventure started (don't ask).

```
0. cd example-1
1. gw clean
2. gw unzipAndRezipZip (calls extractTar because it depends on it)
```

This fails with `path may not be null or empty string. path='null'`, but notice that the
"copyTask" did run:

```
> Task :extractTar
<snip>
Path may not be null or empty string. path='null'
```
And, `a.tgz` is exactly where it should be:
```
$ tree
.
├── build
│   └── tarOutput
│       └── a.zip
```

And if we simply run `gw unzipAndRezipZip` again, it succeeds.

Also notice that this, weirdly, succeeds:

```
$ gw clean unzipAndRezipZip
```

## example-2
This is the same as example-1, but with a modern version of Gradle. In this case the build
doesn't fail, but the result is the same: after one run we only have the untarred zip file
and after two runs we have the rezipped file.

```
0. cd example-2
1. gw clean
2. gw unzipAndRezipZip
3. tree
.
├── build
│   └── tarOutput
│       └── a.zip

4. gw unzipAndRezipZip
5. tree
.
├── build
│   ├── distributions
│   │   └── rezipped.zip
│   └── tarOutput
│       └── a.zip
```

## example-3
In googling around, I read somewhere (maybe [here](https://stackoverflow.com/a/20726722/214429))
that with curly braces you can defer searching for the file until execution (rather than during
the "parsing" / "building" of the project). This results in the following interesting
behavior:

```
0. cd example-3
1. gw unzipAndRezipZip

FAILURE
path may not be null or empty string. path='null'

2. gw extractTar 
3. gw unzipAndRezipZip

Circular dependency between the following tasks:
:unzipAndRezipZip
\--- :unzipAndRezipZip (*)
```

## example-4
In example-4, we present THE SOLUTION (well almost), which is to simply untar the file in the same task:

```
0. cd example-4
1. gw clean unzipAndRezipZip
2. tree

.
├── build
│   ├── tmp
│   │   └── expandedArchives
│   │       └── a.tgz_80f151c6b63f43b01621e096c7cdfd7c
│   │           └── a.zip
│   └── zipOutput
│       └── rezipped.zip
```

However, there's still one kink: if we run `gw clean unzipAndRezipZip` in one go, we get:
```
> Cannot expand ZIP '/Volumes/git/projects/pi/gradle-tests/basic/example-4/build/tmp/expandedArchives/a.tgz_80f151c6b63f43b01621e096c7cdfd7c/a.zip' as it does not exist.
```

The real question is: shouldn't it be easier to have tasks which produce files as outputs and other tasks that take these files as inputs?

## example-5
In example-5, we have had some [very helpful feedback](https://stackoverflow.com/questions/66485931/gradle-process-files-inside-a-zip-inside-a-tar/66538778?noredirect=1#comment117638004_66538778)
from the user mcernak on Stackoverflow, who pointed out that we can do:

```
from{fileTree("${buildDir}/tarOutput").find { it.name.endsWith("zip") }} {
```

instead of 

```
from(fileTree("${buildDir}/tarOutput").find { it.name.endsWith("zip") }) {
```

and thus defer the execution of the `fileTree` call such that it doesn't complain about the the path being null. However, unfortunately, in my quest to make
the example as minimal as possible, I had removed too much. We also need to `zipTree` the `fileTree`, so that the zip gets unzipped, so that we can filter out
`b.txt`.

So in this example, we try what mcernak suggested, with the only addition of adding a `zipTree` around the `fileTree` and we get the same old error:

```
0. cd example-5
1. gw unzipAndRezipZip

FAILURE
path may not be null or empty string. path='null'

2. gw extractTar 
3. gw unzipAndRezipZip => works
```

