# picocli-jansi-graalvm
Helper library for using Jansi in GraalVM native images.

Create native Windows executable command line applications with colors in Java.

## Introduction

GraalVM now offers experimental support for Windows native images,
so it is now possible to **build applications in Java and compile them to a native Windows executable** that does not require a JVM to be installed and has extremely fast startup time and lower memory requirements.

<a href="https://www.graalvm.org/"><img src="https://www.graalvm.org/resources/img/logo-colored.svg" alt="GraalVM logo"></a> &nbsp;&nbsp;&nbsp;  <a href="https://github.com/remkop/picocli"><img src="https://picocli.info/images/logo/horizontal.png" height="40" alt="picocli logo"></a> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <a href="https://github.com/fusesource/jansi"><img src="https://camo.githubusercontent.com/f1eebfa71af81086762ab38337c9d563bd7ac6a1/687474703a2f2f66757365736f757263652e6769746875622e696f2f6a616e73692f696d616765732f70726f6a6563742d6c6f676f2e706e67" alt="jansi logo" height="30"></a>

By building your command line application with the [picocli](https://github.com/remkop/picocli) library you get ANSI colors and styles for free, and you naturally want this functionality when building a native Windows executable.

The [Jansi](https://github.com/fusesource/jansi) library makes it easy to enable ANSI escape codes in the `cmd.exe` console or PowerShell console. Unfortunately, the Jansi library (as of version 1.18) by itself is not sufficient to show show colors in the console when running as a GraalVM native image in Windows.

`picocli-jansi-graalvm` is a helper library that enables the use of ANSI escape codes in GraalVM native image applications running on Windows.

## Usage

The `picocli.jansi.graalvm.AnsiConsole` class can be used as a drop-in replacement of the standard Jansi `org.fusesource.jansi.AnsiConsole` class to enable the Jansi ANSI support, either when running on the JVM or as a native image application.


```java
import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.jansi.graalvm.AnsiConsole; // not org.fusesource.jansi.AnsiConsole

@Command(name = "myapp", mixinStandardHelpOptions = true, version = "1.0",
         description = "Example native CLI app with colors")
public class MyApp implements Runnable {

    @Option(names = "--user", description = "User name")
    String user;
    
    public void run() {
        System.out.printf("Hello, %s%n", user);
    }

    public static void main(String[] args) {
        try (AnsiConsole ansi = AnsiConsole.windowsInstall()) { // enable colors on Windows
            new CommandLine(new MyApp()).execute(args);
        }                                                       // Closeable does cleanup when done
    }
}
```

If you require other Jansi functionality than `AnsiConsole`,
call the `Workaround.enableLibraryLoad()` method before invoking the Jansi code. For example:

```java
import org.fusesource.jansi.internal.WindowsSupport;
import picocli.jansi.graalvm.Workaround;

public class OtherApp {

    static {
        Workaround.enableLibraryLoad();
    }

    public static void main(String[] args) {
        int width = WindowsSupport.getWindowsTerminalWidth();
        doCoolStuff(width);
    }
    // ...
}
```

See the [Jansi README](https://github.com/fusesource/jansi) for more details on what Jansi can do.

## Dependencies

The `picocli-jansi-graalvm` library depends on `org.fusesource.jansi` and nothing else.

In your project, we recommend you use the following dependencies:

```
// Gradle example
dependencies {
    compile "info.picocli:picocli:4.3.2"
    compile "info.picocli:picocli-jansi-graalvm:1.2.0"
    compile "org.fusesource.jansi:jansi:1.18"

    annotationProcessor "info.picocli:picocli-codegen:4.3.2"
}
```

See the [example project](https://github.com/remkop/picocli-jansi-graalvm/tree/master/examples).

## Generating a native image for Windows

There are three steps to set up the toolchain for building native images on Windows. First, install the [latest version of GraalVM](https://www.graalvm.org/docs/getting-started/), 19.2.1 as of this writing.  Next, install the [Microsoft Windows SDK for Windows 7 and .NET Framework 4](https://www.microsoft.com/en-us/download/details.aspx?id=8442) as well as the [C compilers from KB2519277](https://stackoverflow.com/a/45784634/873282). The easiest way to install these is using [chocolatey](https://chocolatey.org/docs/installation):

```
choco install windows-sdk-7.1 kb2519277
```

Finally (from the cmd prompt), activate the sdk-7.1 environment:

```
call "C:\Program Files\Microsoft SDKs\Windows\v7.1\Bin\SetEnv.cmd"
```

This starts a new Command Prompt, with the sdk-7.1 environment enabled. Run all subsequent commands in this Command Prompt window. This completes the toolchain setup.

You can now generate a [native image](https://www.graalvm.org/docs/reference-manual/native-image/) for your application by calling the `native-image` generator tool in the `%GRAAL_HOME%\bin` directory. For example:

```
set GRAAL_HOME=C:\apps\graalvm-ce-19.2.1

:: compile our my.pkg.MyApp class (assuming the source is in the .\src directory)
mkdir classes
javac -cp ^
  .;picocli-4.3.2.jar;picocli-codegen-4.3.2.jar;jansi-1.18.jar;picocli-jansi-graalvm-1.2.0.jar ^
  -sourcepath src ^
  -d classes src\my\pkg\MyApp.java

:: create a jar
cd classes && jar -cvef my.pkg.MyApp ../myapp.jar * && cd ..

:: generate native image
%GRAAL_HOME%\bin\native-image ^
  -cp picocli-4.3.2.jar;jansi-1.18.jar;picocli-jansi-graalvm-1.2.0.jar;myapp.jar ^
  my.pkg.MyApp myapp
```

This creates a `myapp.exe` Windows executable in the current directory for the `my.pkg.MyApp` class.

![MyApp usage](docs/images/myapp-usage.png)

Note that there is a [known issue](https://github.com/oracle/graal/issues/1762) with Windows native images generated with Graal 19.2.1:  the `msvcr100.dll` library is required as an external dependency. This file is not always present on a Windows 10 system, so we recommend that you distribute the `msvcr100.dll` file (you can find it in the `C:\Windows\System32` directory) together with your Windows native image.


## When do we need picocli-jansi-graalvm?

When you want to generate a Windows GraalVM native image for your application.
Other platforms support ANSI escape codes out of the box so Jansi is not needed.

## Why do we need picocli-jansi-graalvm?

When generating a native image, we need two configuration files for Jansi:

* [JNI](https://github.com/oracle/graal/blob/master/substratevm/JNI.md) - Jansi uses JNI, and all classes, methods, and fields that should be accessible via JNI must be specified during native image generation in a configuration file. This library adds a `/META-INF/native-image/picocli-jansi-graalvm/jni-config.json` configuration file for the Jansi `org.fusesource.jansi.internal.CLibrary` and `org.fusesource.jansi.internal.Kernel32` classes.
* [resources](https://github.com/oracle/graal/blob/master/substratevm/RESOURCES.md) - to get a single executable we need to bundle the jansi.dll in the native image. We need some configuration to ensure the jansi.dll is included as a resource.  This library adds a `/META-INF/native-image/picocli-jansi-graalvm/resource-config.json` configuration file that ensures the `/META-INF/native/windows64/jansi.dll` file is included as a resource in the native image.

By including these configuration files in our JAR file, developers can simply put this JAR in the classpath when creating a native image; no command line options are necessary.

Also, when using Jansi without this library, there is a problem extracting the `jansi.dll` from the native image.
The `org.fusesource.hawtjni.runtime.Library` (in jansi 1.18) uses non-standard
system properties to determine the bitMode of the platform,
and these system properties are not available in SubstrateVM (the Graal native image JVM).
As a result, the native library embedded in the Jansi JAR under `/META-INF/native/windows64/jansi.dll`
cannot be extracted from the native image even when it is included as a resource.

The `picocli.jansi.graalvm.Workaround` class provides a workaround for this.

*I filed [a ticket for the above](https://github.com/fusesource/jansi/issues/162) on the Jansi issue tracker, and this project will become obsolete if the Jansi library maintainers accept the [pull](https://github.com/fusesource/jansi-native/pull/21) [requests](https://github.com/fusesource/hawtjni/pull/61) I proposed, or fix these problems in some other way.*

## JNI Configuration Generator

The `jni-config.json` file contains JNI configuration for all classes, methods and fields in `org.fusesource.jansi.internal.CLibrary` and `org.fusesource.jansi.internal.Kernel32`.

The following command can be used to regenerate it:

```
java -cp picocli-4.3.2.jar;jansi-1.18.jar;picocli-codegen-4.3.2.jar ^
  picocli.codegen.aot.graalvm.JniConfigGenerator ^
  org.fusesource.jansi.internal.CLibrary ^
  org.fusesource.jansi.internal.Kernel32 ^
  -o=.\jni-config.json
```
