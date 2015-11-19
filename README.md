# perf-map-agent

A java agent to generate `/tmp/perf-<pid>.map` files for just-in-time(JIT)-compiled methods for use with the [Linux `perf` tools](https://perf.wiki.kernel.org/index.php/Main_Page).

## Build

Make sure `JAVA_HOME` is configured to point to a JDK. Then run the following on the command line:

    cmake .
    make

    # will create links to run scripts in <somedir>
    bin/create-links-in <somedir>

## Architecture

Linux `perf` tools will expect symbols for code executed from unknown memory regions at `/tmp/perf-<pid>.map`. This allows runtimes that
generate code on the fly to supply dynamic symbol mappings to be used with the `perf` suite of tools.

perf-map-agent is an agent that will generate such a mapping file for Java applications. It consists of a Java agent written C and a small
Java bootstrap application which attaches the agent to a running Java process.

When the agent is attached it instructs the JVM to report code blobs generated by the JVM at runtime for various purposes. Most importantly,
this includes JIT-compiled methods but also various dynamically-generated infrastructure parts like the dynamically created interpreter, 
adaptors, and jump tables for virtual dispatch (see `vtable` and `itable` entries). The agent creates a `/tmp/perf-<pid>.map` file which
it fills with one line per code blob that maps a memory location to a code blob name.

The Java application takes the PID of a Java process as an argument and an arbitrary number of additional arguments which it passes to the agent.
It then attaches to the target process and instructs it to load the agent library.

## Command line scripts

The `bin` directory contains a set of shell scripts to combine common `perf` operations with creating the map file. The scripts will
use `sudo` to call `perf` scripts.

 - `create-java-perf-map.sh <pid> <options*>` takes a PID and options. It knows where to find libraries relative to the `bin` directory.
 - `perf-java-top <pid> <perf-top-options>` takes a PID and additional options to pass to `perf top`. Uses the agent to create a new
    `/tmp/perf-<pid>.map` and then calls `perf top` with the given options.
 - `perf-java-record-stack <pid> <perf-record-options>` takes a PID and additional options to pass to `perf record`. Runs
   `perf record -g -p <pid> <perf-record-options>` to collect performance data including stack traces. Afterwards it uses the agent to create a
   new `/tmp/perf-<pid>.map` file.
 - `perf-java-report-stack <pid> <perf-record-options>` calls first `perf-java-record-stack <pid> <perf-record-options>` and then runs
   `perf report` to directly analyze the captured data. You can call `perf report -i /tmp/perf-<pid>.data` again with any options after the
   script has exited to further analyze the data from the previous run.
 - `perf-java-flames <pid> <perf-record-options>` collects data with `perf-java-record-stack` and then creates a visualization
   using [@brendangregg's FlameGraph](https://github.com/brendangregg/FlameGraph) tools. To get meaningful stacktraces spanning several JIT-compiled methods,
   you need to run your JVM with `-XX:+PreserveFramePointer` (which is available starting from JDK8 update 60 build 19) as detailed
   in [ag netflix blog entry](http://techblog.netflix.com/2015/07/java-in-flames.html).
 - `create-links-in <targetdir>` will install symbolic links to the above scripts into `<targetdir>`.

Environment variables:

 - `PERF_MAP_OPTIONS`: a string of additional options to pass to the agent as described below.
 - `PERF_RECORD_SECONDS`: the number of seconds, `perf-java-report-stack` and similar tools will record performance data
 - `PERF_RECORD_FREQ`: the sampling frequence as passed to `perf record -F`
 - `FLAMEGRAPH_DIR`: the directory into which [@brendangregg's FlameGraph](https://github.com/brendangregg/FlameGraph) has been checked out
 - `PERF_JAVA_TMP`: the directory to put temporary files in, the default is `/tmp`
 - `PERF_DATA_FILE`: the file name where `perf-java-record-stack` will output performance data into, the default is `$PERF_JAVA_TMP/perf-<pid>.data`
 - `PERF_FLAME_OUTPUT`: the file name to which the flamegraph SVG will be written, the default is `flamegraph-<pid>.svg`
 - `PERF_FLAME_OPTS`: options to pass to flamegraph.pl (found in FLAMEGRAPH_DIR), the default is  `--color java`

## Options

You can add a comma separated list of options to `perf-java` (or the `AttachOnce` runner). These options are currently supported:

 - `unfold`: Create extra entries for every codeblock inside a method that was inlined from elsewhere
    (named &lt;inlined_method&gt; in &lt;root_method&gt;). Be aware of the effects of 'skid' in relation with unfolding.
    See the section below.
 - `unfoldsimple`: similar to `unfold`, however, the extra entries do not include the " in &lt;root_method&gt;" part
 - `msig`: include full method signature in the name string
 - `dottedclass`: convert class signature (`Ljava/lang/Class;`) to the usual class names with segments separated by dots
   (`java.lang.Class`). NOTE: this currently breaks coloring when used in combination with [flamegraphs](https://github.com/brendangregg/FlameGraph).
 - `sourcepos`: Adds the name of the source file and the line number on which it is declared for each method. Useful
   when profiling Scala applications that crate a lot of synthetic classes and methods. Does not work with native methods.

## Known Issues

### Skid

You should be aware that instruction level profiling is not absolutely accurate but suffers from
'[skid](http://www.spinics.net/lists/linux-perf-users/msg02157.html)'. 'skid' means that the actual instruction
pointer may already have moved a bit further when a sample is recorded. In that case, (possibly hot) code is reported at
an address shortly after the actual hot instruction.

If using `unfold`, perf-map-agent will report sections that contain code inlined from other methods as separate entries.
Unfolded entries can be quite short, e.g. an inlined getter may only consist of a few instructions that now lives inside of another
method's JITed code. The next few instructions may then already belong to another entry. In such a case, it is more likely that skid
will not only affect the instruction pointer inside of a method entry but may affect which entry is chosen in the first place.

Skid that occurs inside a method is only visible when analyzing the actual assembler code (as with `perf annotate`). Skid that
affects the actual symbol resolution to choose a wrong entry will be much more visible as wrong entries will be reported with
tools that operate on the symbol level like the standard views of `perf report`, `perf top`, or in flame graphs.

So, while it is tempting to enable unfolded entries for the perceived extra resolution, this extra information is sometimes just noise
which will not only clutter the overall view but may also be misleading or wrong.

### Agent Library Unloading

Unloading or reloading of a changed agent library is not supported by the JVM (but re-attaching is). Therefore, if you make changes to the
agent and recompile it you need to restart a target process that has an older version loaded to use the newer version.

## Disclaimer

I'm not a professional C code writer. The code is very "experimental", and it is e.g. missing checks for error conditions etc.. Use it at your own risk. You have been warned!

## License

This library is licensed under GPLv2. See the LICENSE file.

