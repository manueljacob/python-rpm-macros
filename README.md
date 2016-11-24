# Multi-Python, Single-Spec macro system

This repository contains a work-in-progress macro system generator for the singlespec python initiative.

### Dictionary

__flavor__ is a kind of python interpreter. At this point, we recognize the following flavors:
`python2`, `python3` and `pypy3`.

In place of flavor, you can encounter `python`. This is for compatibility reasons. In most places,
using `python` is either a redefinition of `python2` for compatibility reasons, or a stand-in for
"flavor-agnostic". Conditionals are in place to switch `python` to mean `python3` in the future.

The name of the flavor should be the name of the binary in `/usr/bin`. It is also used as a prefix
for all flavor-specific macros. Some macros are redefined with "short" flavor, such as `py2` for
`python2`, for compatibility reasons. All of them have a "long" form too.

__modname__ is the PyPI name, or, if the package in question is not on PyPI, the moniker that we
chose to stand in for it.

Packages adhering to the SUSE Python module naming policy are usually called `%{flavor}-%{modname}`.
In some cases, it is only `%{modname}` though.

__pkgname__, or __subpackage name__, is internal to a spec file, and is that thing you put after the
`%package` macro. Pkgname of the package itself is an empty string. Pkgname of a `%package -n
something` is at this point `-n something`, which is not very meaningful. This is a TODO item.

The purpose of the singlespec system is to take a package called `%{flavor}-%{modname}` for a
particular flavor, and autogenerate subpackages `%{flavor}-%{modname}` for all the other flavors.
Alternately, it is to take package `python-%{modname}` and generate subpackages for all flavors,
leaving the top-level package empty.

### Macros

The following macros are considered public API:

__`%pythons`__ - list of flavors we are building for.

__`%have_python2`, `%have_python3`, `%have_pypy3`__. Defined as 1 if we are building for that
flavor, 0 otherwise. (TODO: %have_python should be %have_python2)

__`%{python_module modname [= version]}`__ expands to `$flavor-modname [= version]` for every
flavor. Intended as: `BuildRequires: %{python_module foo}`.

__`%ifpython2`, `%ifpython3`, `%ifpypy3`__: applies the following section only to subpackages of
that particular flavor.

__`%py2_only`, `%py3_only`, `%pypy_only`__: applies the contents of the line only to subpackages of
that particular flavor.

__`%python_build`__ expands to build instructions for  all flavors.

__`%python_install`__ expands to install instructions for all flavors.

__`%python_exec something.py`__ expands to `$flavor something.py` for all flavors, and moves around
the distutils-generated `build` directory so that you are never running python2 script with a
python3-generated `build`. This is only useful for distutils/setuptools. `%python_exec` may gain
more abilities in the future.

__`%python2_build`, `%python3_build`, `%pypy3_build`__ expands to build instructions for the
particular flavor.  
__`%python2_install`, `%python3_install`, `%pypy3_install`__ expands to install
instructions for the particular flavor.

__`%python_subpackages`__ expands to the autogenerated subpackages. This should go after your
`%description` section.

Alternative-related, general:

__`%prepare_alternative [-t <targetfile> ] <name>`__  replaces `<targetfile>` with a symlink to
`/etc/alternatives/<name>`, plus related housekeeping. If no `<targetfile>` is given, it is
`%{_bindir}/<name>`.

__`%install_alternative [-n ]<name> [-s <sourcefile>] [-t ]<targetfile> [-p ]<priority>`__  runs the
`update-alternative` command, configuring `<sourcefile>` alternative to be `<targetfile>`, with a
priority `<priority>`. If no `<sourcefile>` is given, it is `%{_bindir}/<name>`.  Can be followed by
additional arguments to `update-alternatives`, such as `--slave`.

__`%uninstall_alternative [-n ]<name> [-t ]<targetfile>`__  if uninstalling (not upgrading) the
package, remove `<targetfile>` from a list of alternatives under `<name>`

Alternative-related, for Python:

__`%python_alternative <name>`__: expands to a filelist entries for `%{_bindir}/<name>`, the
`/etc/alternatives/<name>` symlink, and the target called `%{_bindir}/<name>-%{py_ver}` (or
$flavor_ver respectively).

__`%python_install_alternative <name>`__: runs `update-alternatives` for  `<name>-%{py_ver}`.

__`%python_uninstall_alternative <name>`__: reverse of the preceding.

Each of these has a flavor-specific spelling: `%python2_alternative` etc.

__`%python2_bin_suffix`, `%python3_bin_suffix`, `%pypy3_bin_suffix`__: what to put after
a binary name. Binaries for CPython are called `binary-%{python_version}`, for PyPy the name
is `binary-pp%{pypy3_version}`

In addition, the following flavor-specific macros are known and supported by the configuration:

__`%__python2`__: path to the $flavor executable.

__`%python2_sitelib`, `%python2_sitearch`__: path to noarch and arch-dependent `site-packages`
directory.

__`%python2_version`__: dotted major.minor version. `2.7` for CPython 2.7.  
__`%python2_version_nodots`__: concatenated major.minor version. `27` for CPython 2.7.

For reasons of preferred-flavor-agnosticity, aliases `python_*` are available for all of these.

We recognize `%py_ver`, `%py2_ver` and `%py3_ver` as deprecated spellings of `%flavor_version`. No
such shortcut is in place for `pypy3`. Furthermore, `%py2_build`, `_install` and `_shbang_opts`, as
well as `py3` variants, are recognized for Fedora compatibility.


### Files in repository

__`macros` directory__: contains a list of files that are concatenated to produce the resulting
`macros.python_all` file. This directory is incomplete, files `020-flavor-$flavor` and
`040-automagic` are generated by running the compile script.

__`macros/001-alternatives`__: macro definitions for alternatives handling. These are not
python-specific and might find their way into `update-alternatives`.

__`macros/010-common-defs`__: setup for macro spelling templates, common handling, preferred flavor
configuration etc.

__`macros/030-fallbacks`__: compatibility and deprecated spellings for some macros.

__`compile-macros.sh`__: the compile script. Builds flavor-specific macros, Lua script definition,
and concatenates all of it into `macros.python_all`

__`apply-macros.sh`__: compile macros and run `rpmspec` against first argument. Useful for examining
what is going on with your spec file.

__`flavor.in`__: template for flavor-specific macros. Generates `macros/020-flavor-$flavor` for
every flavor listed in `compile-macros.sh`.

__`macros.in`__: pure-RPM-macro definitions for the single-spec generator. References and uses the
private Lua macros.  The line `### LUA-MACROS ###` is replaced with inlined Lua macro definitions.

__`macros.lua`__: Lua macro definitions for the single-spec generator.  
This is actually pseudo-Lua: the top-level functions are not functions, and are instead converted to
Lua macro snippets. (That means that you can't call the top-level functions from Lua, and that you
need to place actual Lua functions inside a snippet, preferably the `_python_scan_spec()` snippet.)

__`embed-macros.pl`__: takes care of slash-escaping and wrapping the Lua functions and inserting
them into the `macros.in` file in order to generate the resulting macros.

__`python-rpm-macros.spec`__: spec file for the `python-rpm-macros` package generated from this
github repository.

__`process-spec.pl`__: Simple regexp-based converter into the singlespec format.

__`README.md`__: This file. As if you didn't know.
