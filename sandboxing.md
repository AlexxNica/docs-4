# Sandboxing

This document describes how sandboxing works in Fuchsia.

## An empty process has nothing

On Fuchsia, a newly created process has nothing. A newly created process cannot
access any kernel objects, cannot allocate memory, and cannot even execute code.
Of course, such a process isn't very useful, which is why we typically create
processes with some initial resources and capabilities.

Most commonly, a process starts executing some code with an initial stack, some
command line arguments, environment variables, a set of initial handles. One of
the most important initial handles is the `PA_VMAR_ROOT`, which the process can
use to map additional memory into its address space.

## Namespaces are the gateway to the world

Some of the initial handles given to a process are directories that the process
mounts into its _namespace_. These handles let the process discover and
communicate with other processes running on the system, including file systems
and other servers. See [Namespaces](namespaces.md) for more details.

The namespace given to a process strongly influences how much of the system the
process can influence. Therefore, configuring the sandbox in which a process
runs amounts to configuring the process's namespace.

## Archives and namespaces

In our current implementation, a process runs in a sandbox if its binary is
contained in an archive (i.e., a [FAR](glossary.md#FAR)). As the package manager
evolves, these details are likely to change.

An application run from an archive is given access to two namespaces by default:

 * `/svc`, which is a bundle of services from the environment in which the
   application runs.
 * `/pkg`, which is a read-only view of the archive containing the application.

A typical application will interact with a number of services from `/svc` in
order to play some useful role in the system.

The `far` command-line tool can be used to inspect packages installed on the
system. For example, `far list --archive=/system/pkgs/root_presenter` will list
the contents of the `root_presenter` archive:

```
$ far list --archive=/system/pkgs/root_presenter
bin/app
data/cursor32.png
meta/sandbox
```

To access these resources at runtime, a process can use the `/pkg` namespace.
For example, the `root_presenter` can access `cursor32.png` using the absolute
path `/pkg/data/cursor32.png`.

## Configuring additional namespaces

If a process requires access to additional resources (e.g., device drivers), the
package can request access to additional names by including a
[sandbox metadata file](package_metadata.md#sandbox) in its package. For
example, the following `meta/sandbox` file requests direct access to the
framebuffer driver:

```
{
    "dev": [ "class/framebuffer" ]
}
```

In the current implementation, the [AppMgr](glossary.md#AppMgr) grants all such
requests, but that is likely to change as the system evolves.

## Building an archive

To build a package, use the `package()` macro in `gn` defined in
[`//packages/packages.gni`](https://fuchsia.googlesource.com/packages/+/master/package.gni).
Specifically, to create a Fuchsia Archive (FAR) for your package (which will
trigger sandboxing), set the `archive` flag to `true`. See the documentation for
the `package()` macro for details about including resources

For examples, see [https://fuchsia.googlesource.com/packages/+/master/gn/mozart]
and [https://fuchsia.googlesource.com/mozart/+/master/BUILD.gn].
