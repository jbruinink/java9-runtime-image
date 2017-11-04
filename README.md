# Creating a slim modular Java 9 runtime Docker image with Alpine Linux

With the release of Java 9, and the introduction of Project Jigsaw (the Java Platform Module System),
we no longer have the need for a full-blown JRE to run our Java applications.
It is now possible to construct a stripped-down Java Runtime, containing the minimum set of required modules.
This allows us to create slim Docker containers without excess baggage.

## Building and running the application
Our example application consists of two Java 9 modules (basically two JARS, with a `module-info.java`).
We assume the concept of modules is familiar to the reader. If not, you can learn more about it e.g. here: <http://openjdk.java.net/projects/jigsaw/>

Firstly, we have a `backend` module consisting of a class which provides us with a `String` (to keep it simple).
The backend module has no explicit dependencies on other modules (only an implicit dependency on `java.base`).

Secondly, we have `frontend` module consisting of an executable main class, which gets the String from the backend
and prints it to `System.out` (again, very straightforward). This module has an explicit dependency on `backend`,
and an implicit dependency on `java.base`.

To see the application in action, build it with Maven (`mvn clean package`), and run it from the command line:

```sh
> java -p frontend-module/target/frontend-module-1.0-SNAPSHOT.jar:backend-module/target/backend-module-1.0-SNAPSHOT.jar \
    -m com.jdriven.java9runtime.frontend/com.jdriven.java9runtime.frontend.FrontendApplication
```
(or use the provided [run-app.sh](run-app.sh))

The `-p` option sets the module path (similar to the 'old' classpath), and the `-m` option specifies the module and class to run.

## Creating a custom Java runtime image with JLink

The `jlink` tool, provided with the Java 9 JDK, allows us to combine our application modules with the required modules
from the JDK (in our case, only `java.base`) into a custom-tailored JRE. Please note that the generated JRE, like any JRE, is *NOT* platform independent!
The generated JRE is not portable to other platforms.

Because we would like to run our application inside Docker on top of [Alpine Linux](https://alpinelinux.org/) (a very small Linux distro, approximately 5 MB),
we need to run `jlink` with a JDK that is compatible with the Alpine OS. Unfortunately, at the time of writing,
neither the Oracle JDK nor the OpenJDK release of Java 9 support Alpine Linux yet (see: <https://github.com/anapsix/docker-alpine-java/issues/38>).
However, there is an OpenJDK Early-Access Build available, which we can use, at: <http://jdk.java.net/9/ea>

We could install this EA JDK and use it to run `jlink`, or we could do it all from inside a Docker container, which is cooler.
The [Dockerfile.jlink](Dockerfile.jlink) file is used to create a Docker container for our `jlink` command.

Build this Docker image using the command: `docker build -f Dockerfile.jlink -t jlink-java9-runtime-image .` (<-- note the period at the end of the command)

Basically, this Docker image (based on Alpine), downloads and installs the EA JDK for Alpine, and runs the `jlink` command
on our sources. To be able to access our compiled jars, we mount our project directory when running the container,
using the command: `docker run -it --rm -v "$PWD":/app/ jlink-java9-runtime-image` (see [docker-jlink.sh](docker-jlink.sh)).

Our `jlink` command takes the following parameters:

- `--module-path` once again sets the module path to include our modules, and the default JDK modules (located at `$JAVA_HOME/jmods`)
- `--add-modules` defines the set of root modules to include. We only need to include the frontend module. The backend module is included transitively because it is required by frontend.
- `--launcher` specifies a command name to launch our application, and defines which class in which module is the main class (in the form of `--launcher commandName=module/mainClass`)
- `--output` specifies a destination directory (which should not exist yet!) in which the runtime is generated

The other parameters are included to decrease the image in size, by using compression and stripping some irrelevant data.
For more information regarding these parameters, see: <https://docs.oracle.com/javase/9/tools/jlink.htm>  

## Executing the custom Java runtime

After running JLink, we now have our own custom JRE, which is only approximately 30 MB in size, in a newly created `./dist` directory.
This of course is still quite large for a hello world app, but compared to the default JRE, which is larger than 200 MB,
it's quite an improvement.

We can execute our runtime by using the launcher command defined earlier (`run`), which is located at `./dist/bin/run`.

Examining this script, generated by JLink, we see that it basically just starts the stripped down JRE,
which contains only our modules and `java.base`, supplying the main module and class we provided earlier:

```sh
> cat dist/bin/run
#!/bin/sh
JLINK_VM_OPTIONS=
DIR=`dirname $0`
$DIR/java $JLINK_VM_OPTIONS -m com.jdriven.java9runtime.frontend/com.jdriven.java9runtime.frontend.FrontendApplication $@
```

## Building a Docker image for our application

Because the created JRE is fully self-contained, all we need is a simple Alpine-based Docker base image to execute
our `run` command. See: [Dockerfile.run](Dockerfile.run)

The Dockerfile is quite simple. Starting with the Alpine base image, copy the contents of `dist` into our image, and set `bin/run` as an entrypoint.

Build the Docker image:

```sh
> docker build -f Dockerfile.run -t java9-runtime-image .
```

To test the image, run an instance by executing:

```sh
> docker run --rm java9-runtime-image
```
(or use the provided [docker-run.sh](docker-run.sh) to build and run it all)

That's it!

## Conclusion

The combination of a small Alpine Linux distro (5 MB) and our stripped down JRE (30 MB),
results in a total Docker image size of approximately 35 MB. By comparison, the `openjdk:8-jre-alpine` Docker image is 80 MB.
A reduction of more than 50 percent!

Of course, any _real-life_ software project also includes a number of third-party dependencies,
and will almost certainly need more modules from the JRE than this Hello World example.
We have yet to see if this approach will give us a significant benefit in real-life applications.
