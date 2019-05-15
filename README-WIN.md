# plpgsql_check Documentation
Broken down into two sections
 1. Compiling the source code
 1. Updating extension
 1. Working with sqitch

## Compiling plpgsql_check on Win64

`plpgsql_check` is next generation of plpgsql_lint. It allows to check source code by explicit call `plpgsql_check` 
function. See <https://github.com/okbob/plpgsql_check>.

Like most extensions for postgres,`plpgsql_check` has a nice make file for Linux systems,
but nothing for Win64. Here are the steps to compile this extension on Win64.

**We use 64 bit version of postgres, so the build configuration has to be for 64-bit.**

### Before you start
* Don't try and run in Visual Studio 2012 - its too old.
* You need to have postgres files in certain directory.
* So read the steps below.

### Steps

1. Copy postgres binaries to the build machine. The postgres binaries must be the same version as the postgres instance where
the extension will be running. In the Visual C++ solution the postgres binaries have been copied to _D:\pg\dev_
1. We need to create a lib file from `plpgsql.dll`. Goto the directory (usually ../lib) where the postgres binaries were copied and find _plpgsql.dll_. From this we will create a _plpgsql.lib_ 
(for this step see `Create plpgsql.lib` section.)
1. Get Visual Studio C++ compiler (Visual Studio Community <https://go.microsoft.com/fwlink/?LinkId=691978&clcid=0x409>)
1. We are building for 64bit...x64.
1. Some tips on how to setup the visual studio C++ environment for postgres 
extension development can be found at <http://blog.2ndquadrant.com/compiling-postgresql-extensions-visual-studio-windows>.
1. I used this <https://github.com/jmguazzo/pg_empty> project as the base solution & project. After copying the solution and project:
    * Rename all references of _pg_empty_ to _plpgsql_check_
    * Open the _plpgsql_check.vcxproj_ (the old _pg_empty.vcxproj_) in a text editor and search 'n' replace _C:\Program Files\PostgreSQL\9.4_ or 
    _C:\Program Files\PostgreSQL\9.5_ with _C:\pg\dev_
    * Remove _restart_service_x64.cmd_ and _restart_service_x86.cmd_ from _plpgsql_check.vcxproj_ and _plpgsql_check.vcxproj.filters_.
    * Delete _restart_service_x64.cmd_ and _restart_service_x86.cmd_ from the actual solution.
    * Delete _pg_empty.c_ file from project.
1. Now the shell solution is ready. Need to copy the following `plpgsql_check` files into the solution (these need to live under the _Source Files_ directory:
    * plpgsql_check.c
    * plpgsql_check_builtins.h
1. Double-check the following settings:
    * Right-Click on the C++ project, select _properties_.
    * In _Configuration_ (top left), choose _All Configurations_.
    * Under _General -> Platform Toolset_ choose _Visual Studio 2013 (v120)_ (this corresponds to _Visual C++ Redistributable Packages for Visual Studio 2013
_)
    * Under _Linker -> Input -> Additional Dependencies_, add _postgres.lib_ to the library list.
    * Under _Linker -> Input -> Additional Dependencies_, add _plpgsql.lib_ to the library list 
    (this is the lib we created in the `Create plpgsql.lib` step below).
    * Include and library directories, C/C++ -> General. In Additional Include Directories you should have 
    (in the same order):
    ```text
    include\server\port\win32_msvc
    include\server\port\win32
    include\server
    include
    ```
    * For Postgres 11+ you need to also install unicode lib: https://github.com/unicode-org/icu/releases/tag/release-64-2
    * Extract the header files to where you have the postgres binaries then
    * Include and library directories, C/C++ -> General. In Additional Include Directories you should add 
    ```
    include\icu\include
    ```

    * Under _Linker -> General, set the library path: _C:\pg\dev\lib_
1. Now need to edit the actual `plpgsql_check.c` source file. Need to add PGDLLEXPORT line before every extension function. 
In current version these include:
    ```c
    plpgsql_check_function(PG_FUNCTION_ARGS)
    plpgsql_check_function_tb(PG_FUNCTION_ARGS)
    ```
    ```c
    PGDLLEXPORT
    Datum
    plpgsql_check_function_tb(PG_FUNCTION_ARGS)
    ```
1. How to use the extension is covered on the github page.
1. Compile a release version. Happy days...hopefully.
1. You may need to install _Visual C++ Redistributable Packages for Visual Studio 2013_ 
<https://www.microsoft.com/en-us/download/details.aspx?id=40784> on the postgres dev machine.

#### Create plpgsql.lib
* These instructions are based off <https://adrianhenke.wordpress.com/2008/12/05/create-lib-file-from-dll/>
* Goto the directory where visual studio command line tools live. Something like 
_C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin_
* Run `dumpbin`
```shell
cd C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin
dumpbin /exports C:\pg\dev\lib\plpgsql.dll | clip
```
* That copies the output to your clipboard.
* Dump the output into your favourite editor, and copy out the function names. Should end up with something like:
```text
Pg_magic_func
_PG_init
exec_get_datum_type
exec_get_datum_type_info
pg_finfo_plpgsql_call_handler
pg_finfo_plpgsql_inline_handler
```
* Add `EXPORTS` to the top of the file and save as _plpgsql.def_
* With the new definition file we can now create the .lib file. Use the lib tool for this:
```shell
cd C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin
lib /def:C:\Temp\plpgsql.def /OUT:C:\Temp\plpgsql.lib /MACHINE:x64
```
* Add the lib file to the same directory where you found the corresponding dll.

## Updating extension
1. Copy the updated dll to `PostgreSQL\9.3\dev\lib` (dll & lib)
1. Copy the extension files to `PostgreSQL\9.3\dev\share\extension` (CONTROL file, Sql Scripts)
1. Run the sql statement below:
```sql
ALTER EXTENSION plpgsql_check UPDATE;
```

## Working with sqitch

We need to be using this new `plpgsql_check` function in our verify scripts for functions and triggers.

The `plpgsql_check` function can only be run in _dev_ and _uat_ environments.


```sql
-- Sample verify using Xml
DO $$
    DECLARE vOutput xml;
BEGIN

    SELECT  plpgsql_check_function('sp_searchEvent(BIGINT,JSON)', format:='xml')::xml
    INTO    vOutput;

    IF (SELECT xmlexists('/Function/Issue' PASSING BY REF vOutput)) THEN
        RAISE EXCEPTION 'Error: %', vOutput;
    END IF;

END $$;
```

**Todo** - add the json version.



