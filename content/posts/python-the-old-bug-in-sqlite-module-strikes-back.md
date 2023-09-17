+++
date = "2020-05-15T15:06:19+01:00"
title = "Python: the old bug in SQLite module strikes back"
tags = ["python", "sqlite3", "gdb"]
description = "Python: the old bug in SQLite module strikes back"
categories = ["Linux"]
+++

#### Introduction:

**NOTE:** The problem is related to Python 2 but I'll be referring to Python 3 documentation unless it's not applicable.

I was recently writing some Python code that was using the [SQLite module](https://docs.python.org/3/library/sqlite3.html). Unfortunately it was still running on `Python 2.7.9` and as of January 1st, 2020, Python 2 has officially reached the end of life but let's focus on the actual problem.

The code was pulling data from different sources in chunks, saved the records into the local database and then data was sent to an API endpoint at each iteration. The logic was something like this:

```python
with DB() as db:
    db.init()

    it = iter(data)
    chunk = tuple(islice(it, 0, 1000))
    while chunk:
        db.cur.add(chunk)
        # Make the HTTP request or raise an exception on error
        db.conn.commit() # <- THIS COMMIT SEEMS TO BE THE CULPRIT
        chunk = tuple(islice(it, 0, 1000))
 ```

#### Variables:

```
CentOS 7.8.2003
Python 2.7.9
SQLite 3.7.17
pyenv 1.2.14
GDB 7.6.1
```

#### Problem:

This is a simplified test case to reproduce the issue:

```python
import sqlite3


def main():
    conn = sqlite3.connect(':memory:')
    cur = conn.cursor()
    
    sql = '''
        CREATE TABLE IF NOT EXISTS t1 (
          char TEXT NOT NULL
        )
    '''
    
    cur.executescript(sql)
    cur.execute('INSERT INTO t1 VALUES (?)', ('a',))
    cur.execute('SELECT char FROM t1 WHERE char = ?', ('a',))
    conn.commit()
    print(cur.fetchone())
    cur.execute('SELECT char FROM t1 WHERE char = ?', ('b',))


if __name__ == '__main__':
    main()
```

The code was working fine when I removed the _commit_ statement otherwise it was throwing the following exception:

```sh-session
$ python test.py
(u'a',)
Traceback (most recent call last):
  File "test.py", line 25, in <module>
    main()
  File "test.py", line 21, in main
    cur.execute('SELECT char FROM t1 WHERE char = ?', ('b',))
sqlite3.InterfaceError: Error binding parameter 0 - probably unsupported type.
[22472 refs]
```

#### Installing the required tools:

**NOTE:** I'm using [pyenv](https://github.com/pyenv/pyenv) to manage multiple versions of Python so all path locations are referring to my pyenv installation.

We'll install `Python 2.7.9` using `pyenv`:

```sh-session
$ pyenv install -f -k -v -g -p 2.7.9
```

The SQLite module is written in `C` so we'll use `gdb` (GNU Debugger) for debugging. We'll also need to install debugging symbols for Python and SQLite:

```sh-session
$ sudo yum install python-debuginfo http://debuginfo.centos.org/7/x86_64/glibc-debuginfo-2.17-307.el7.1.x86_64.rpm http://debuginfo.centos.org/7/x86_64/sqlite-debuginfo-3.7.17-8.el7_7.1.x86_64.rpm
```

I had to pull some pkgs directly from the `debuginfo` repo because my system was using `glibc-2.17-307.el7.1` and `sqlite-3.7.17-8.el7_7.1` but the equivalent `debuginfo` pkgs were not available:

```sh-session
$ sudo yum -q list --showduplicates --enablerepo=base-debuginfo glibc-debuginfo sqlite-debuginfo
Installed Packages
glibc-debuginfo.x86_64                                                              2.17-307.el7.1                                                                @/glibc-debuginfo-2.17-307.el7.1.x86_64   
sqlite-debuginfo.x86_64                                                             3.7.17-8.el7_7.1                                                              @/sqlite-debuginfo-3.7.17-8.el7_7.1.x86_64
Available Packages
...                            
glibc-debuginfo.x86_64                                                              2.17-292.el7                                                                  base-debuginfo                            
...           
sqlite-debuginfo.x86_64                                                             3.7.17-8.el7                                                                  base-debuginfo
```

(A quick look into the `repodata` directory suggests that metadata wasn't rebuilt for a while)

Add the following line into `~/.gdbinit` file to add some Python debugging tools for `gdb` (`py-bt`, `py-bt` etc.) :

```
add-auto-load-safe-path ~/.pyenv/versions/2.7.9-debug/bin/python2.7-gdb.py
```

It's time for some debugging, the `-k` flag instructs `pyenv` to keep the source tree, let's find the relevant error message from the exception:

```sh-session
$ grep -R "Error binding parameter" ~/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/
/home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/statement.c:                    PyErr_Format(pysqlite_InterfaceError, "Error binding parameter %d - probably unsupported type.", i);
/home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/statement.c:                    PyErr_Format(pysqlite_InterfaceError, "Error binding parameter :%s - probably unsupported type.", binding_name);
```

So after we checked `statement.c` file, it looks like both errors are located in `pysqlite_statement_bind_parameters` function that has the following signature:

```c
void pysqlite_statement_bind_parameters(pysqlite_Statement* self, PyObject* parameters, int allow_8bit_chars)
```

Let's start `gdb` session and create a `breakpoint` to pause the execution at this location. The exception happens when we try to do _select_ after the _commit_ statement (`SELECT char FROM t1 WHERE char = 'b'`) so we'll add a conditional breakpoint.

We also know that "everything is an [object](https://docs.python.org/3/c-api/structures.html) in Python" so we need to use some `C` functions from the [Python/C API](https://docs.python.org/3/c-api/index.html) :

```sh-session
break pysqlite_statement_bind_parameters if PyTuple_Size(parameters) == 1 && $_streq(PyString_AsString(PyTuple_GetItem(parameters, 0)), "b")
```

The condition says "pause" if `parameters` size is 1 and if that's the case, we use `$_streq` [convenience function](https://sourceware.org/gdb/current/onlinedocs/gdb/Convenience-Funs.html) to check if the first parameter is "b". 

This is equivalent to the following Python code:

```python
>>> parameters = ('b',)
>>> len(parameters) and parameters[0] == 'b'
True
```

#### GDB session:

```sh-session
$ gdb -q ~/.pyenv/versions/2.7.9-debug/bin/python
Reading symbols from /home/test/.pyenv/versions/2.7.9-debug/bin/python2.7...done.
(gdb) break pysqlite_statement_bind_parameters if PyTuple_Size(parameters) == 1 && $_streq(PyString_AsString(PyTuple_GetItem(parameters, 0)), "b")
Function "pysqlite_statement_bind_parameters" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y

Breakpoint 1 (pysqlite_statement_bind_parameters if PyTuple_Size(parameters) == 1 && $_streq(PyString_AsString(PyTuple_GetItem(parameters, 0)), "b")) pending.
```

We've set the breakpoint so it's time to start our script:

```sh-session
(gdb) run test.py
Starting program: /home/test/.pyenv/versions/2.7.9-debug/bin/python test.py
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
(u'a',)

Breakpoint 1, pysqlite_statement_bind_parameters (self=0x7fffef7be2b0, parameters=('b',), allow_8bit_chars=0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/statement.c:226
226	    Py_BEGIN_ALLOW_THREADS
```

We've hit our breakpoint so we can take a look around:

```sh-session
(gdb) whatis self
type = pysqlite_Statement *
(gdb) ptype self
type = struct {
    struct _object *_ob_next;
    struct _object *_ob_prev;
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
    sqlite3 *db;
    sqlite3_stmt *st;
    PyObject *sql;
    int in_use;
    PyObject *in_weakreflist;
} *
(gdb) p self->sql
$1 = 'SELECT char FROM t1 WHERE char = ?'
```

The `self` argument is a `pysqlite_Statement *` structure and the `sql` member is our _select_ statement.

Let's hit `next` (a few times) to proceed to the next line. We use a `tuple`  as a parameter so we go into this branch in the `if` statement and finally we see some interesting variable: 

```sh-session
(gdb) n
227	    num_params_needed = sqlite3_bind_parameter_count(self->st);
(gdb) 
228	    Py_END_ALLOW_THREADS
(gdb) 
230	    if (PyTuple_CheckExact(parameters) || PyList_CheckExact(parameters) || (!PyDict_Check(parameters) && PySequence_Check(parameters))) {
(gdb) 
232	        if (PyTuple_CheckExact(parameters)) {
(gdb) 
233	            num_params = PyTuple_GET_SIZE(parameters);
...
270	            rc = pysqlite_statement_bind_parameter(self, i + 1, adapted, allow_8bit_chars);
(gdb) 
271	            Py_DECREF(adapted);
(gdb) p rc
$2 = 21
```

The `pysqlite_statement_bind_parameter` return code was `21` and according to [SQLite docs](https://sqlite.org/rescode.html#misuse) this doesn't look good:


> (21) SQLITE_MISUSE
>
> The SQLITE_MISUSE return code might be returned if the application uses any SQLite interface in a way that is undefined or unsupported. For example, using a prepared statement after that prepared statement has been finalized might result in an SQLITE_MISUSE error.
> ...

We remember that the exception actually happens around the _commit_ so let's change our focus on that. The `pysqlite_connection_commit` subroutine is located in `connection.c`:

```c
PyObject* pysqlite_connection_commit(pysqlite_Connection* self, PyObject* args)
```

We disable our first breakpoint, set a new one and re-start our script:

```sh-session
(gdb) disable 1
(gdb) break pysqlite_connection_commit
Breakpoint 2 at 0x7fffefabc76a: file /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c, line 457.
(gdb) run test.py
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/test/.pyenv/versions/2.7.9-debug/bin/python test.py
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 2, pysqlite_connection_commit (self=0x7ffff7e5bd60, args=0x0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c:457
457	    if (!pysqlite_check_thread(self) || !pysqlite_check_connection(self)) {
(gdb) n
461	    if (self->inTransaction) {
(gdb) p self->inTransaction
$4 = 0
(gdb) p self->statements
$5 = []
```

We're not in transaction and there are no statements so that must be our call to [sqlite3.Cursor.executescript](https://docs.python.org/3/library/sqlite3.html#sqlite3.Cursor.executescript) as it will run `COMMIT` first:

> This is a nonstandard convenience method for executing multiple SQL statements at once. It issues a `COMMIT` statement first, then executes the SQL script it gets as a parameter.

Let's continue until we hit our breakpoint again. This time we're in transaction and we can see our statements. We also use again some Python `Python/C API` functions to get the data we want:

```sh-session
(gdb) cont
Continuing.

Breakpoint 4, pysqlite_connection_commit (self=0x7ffff7e5bd60, args=0x0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c:457
457	    if (!pysqlite_check_thread(self) || !pysqlite_check_connection(self)) {
(gdb) n
461	    if (self->inTransaction) {
(gdb) p self->inTransaction 
$6 = 1
(gdb) whatis self 
type = pysqlite_Connection *
(gdb) ptype self
type = struct {
    ...
    PyObject *statements;
    PyObject *cursors;
    int created_statements;
    int created_cursors;
    ...
} *
(gdb) p self->statements 
$7 = [<weakref at remote 0x7ffff095c528>, <weakref at remote 0x7ffff095c5b0>]
(gdb) p ((pysqlite_Statement *) PyWeakref_GetObject(PyList_GetItem(self->statements, 0)))->sql
$8 = 'INSERT INTO t1 VALUES (?)'
(gdb) p ((pysqlite_Statement *) PyWeakref_GetObject(PyList_GetItem(self->statements, 1)))->sql
$9 = 'SELECT char FROM t1 WHERE char = ?'
```

After that we see another function call `pysqlite_do_all_statements` that apparently is trying to "reset" our statements, let's step into that function to take a look:

```sh-session
(gdb) n
462	        pysqlite_do_all_statements(self, ACTION_RESET, 0);
(gdb) s
pysqlite_do_all_statements (self=0x7ffff7e5bd60, action=2, reset_cursors=0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c:245
245	    for (i = 0; i < PyList_Size(self->statements); i++) {
(gdb) p action
$24 = 2
(gdb) n
246	        weakref = PyList_GetItem(self->statements, i);
(gdb) 
247	        statement = PyWeakref_GetObject(weakref);
(gdb) 
248	        if (statement != Py_None) {
(gdb) 
249	            if (action == ACTION_RESET) {
(gdb) 
250	                (void)pysqlite_statement_reset((pysqlite_Statement*)statement);
(gdb) p ((pysqlite_Statement *) statement)->sql 
$25 = 'INSERT INTO t1 VALUES (?)'
```

So it looks like it loops over our statements and use the same `Python/C API` functions like we did to read them. The first one is our _insert_ so we can skip this iteration of the loop with `until`: 

```sh-session
(gdb) u
245	    for (i = 0; i < PyList_Size(self->statements); i++) {
(gdb) n
246	        weakref = PyList_GetItem(self->statements, i);
(gdb) 
247	        statement = PyWeakref_GetObject(weakref);
(gdb) 
248	        if (statement != Py_None) {
(gdb) 
249	            if (action == ACTION_RESET) {
(gdb) 
250	                (void)pysqlite_statement_reset((pysqlite_Statement*)statement);
(gdb) p ((pysqlite_Statement *) statement)->sql 
$26 = 'SELECT char FROM t1 WHERE char = ?'
```

Here is our _select_ query, this time we'll step into the `pysqlite_statement_reset` function call:

```sh-session
(gdb) s
pysqlite_statement_reset (self=0x7fffef7be2b0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/statement.c:392
392	    rc = SQLITE_OK;
(gdb) n
394	    if (self->in_use && self->st) {
(gdb) p self->in_use 
$27 = 1
(gdb) n
395	        Py_BEGIN_ALLOW_THREADS
(gdb) 
396	        rc = sqlite3_reset(self->st);
(gdb) 
397	        Py_END_ALLOW_THREADS
gdb) 
399	        if (rc == SQLITE_OK) {
(gdb) 
400	            self->in_use = 0;
(gdb) p self->in_use 
$28 = 1
(gdb) n
404	    return rc;
(gdb) p self->in_use 
$29 = 0
```

The code invokes [sqlite3_reset](https://sqlite.org/c3ref/reset.html) function to reset our statement and if that was successful it changes `self->in_use` from `1` to `0`.

We can see that the life-cycle of a [prepared statement](https://sqlite.org/c3ref/stmt.html) object usually goes like this:

> 1.  Create the prepared statement object using [sqlite3_prepare_v2()](https://sqlite.org/c3ref/prepare.html).
> 2.  Bind values to [parameters](https://sqlite.org/lang_expr.html#varparam) using the sqlite3_bind_*() interfaces.
> 3.  Run the SQL by calling [sqlite3_step()](https://sqlite.org/c3ref/step.html) one or more times.
> 4.  Reset the prepared statement using [sqlite3_reset()](https://sqlite.org/c3ref/reset.html) then go back to step 2. Do this zero or more times.
> 5.  Destroy the object using [sqlite3_finalize()](https://sqlite.org/c3ref/finalize.html).

Let's test quickly what will happen if we skip this step:

```sh-session
(gdb) del
Delete all breakpoints? (y or n) y
(gdb) break pysqlite_connection_commit
Breakpoint 5 at 0x7fffefabc76a: file /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c, line 457.
(gdb) run test.py
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/test/.pyenv/versions/2.7.9-debug/bin/python test.py
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 5, pysqlite_connection_commit (self=0x7ffff7e5bd60, args=0x0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c:457
457	    if (!pysqlite_check_thread(self) || !pysqlite_check_connection(self)) {
(gdb) cont
Continuing.

Breakpoint 5, pysqlite_connection_commit (self=0x7ffff7e5bd60, args=0x0) at /home/test/.pyenv/sources/2.7.9-debug/Python-2.7.9/Modules/_sqlite/connection.c:457
457	    if (!pysqlite_check_thread(self) || !pysqlite_check_connection(self)) {
(gdb) n
461	    if (self->inTransaction) {
(gdb) p self->inTransaction
$47 = 1
(gdb) set self->inTransaction = 0
(gdb) p self->inTransaction
$48 = 0
(gdb) cont
Continuing.
(u'a',)
[22472 refs]
[Inferior 1 (process 29338) exited normally]
```

We've set the `self->inTransaction` flag to `0` so the `pysqlite_do_all_statements` function wasn't invoked and the program completed successfully.

This sounds like we shouldn't reset the statements in commits, especially change their states.

At this point I've decided to check for any changes around that part of the code and bingo! The issue was reported in [#10513](https://bugs.python.org/issue10513) and the fix was committed in [#dc60c75](https://github.com/python/cpython/commit/dc60c75aee34d12d93e95bf0f941f8afd56bca98#diff-affe43c743133796bb0a7eec464483b9) (it seems to be fixed in `Python 2.7.13`).

The patch below can be easily applied with `pyenv`:

```sh-session
$ cat ~/issue10513.patch 
--- Modules/_sqlite/connection.c	2014-12-10 15:59:53.000000000 +0000
+++ connection.c	2020-05-15 14:23:36.232892608 +0100
@@ -459,7 +459,6 @@
     }
 
     if (self->inTransaction) {
-        pysqlite_do_all_statements(self, ACTION_RESET, 0);
 
         Py_BEGIN_ALLOW_THREADS
         rc = sqlite3_prepare(self->db, "COMMIT", -1, &statement, &tail);
```

```
$ pyenv install -f -k -v -g -p 2.7.9 < ~/issue10513.patch
~/.pyenv/sources/2.7.9-debug ~
~/.pyenv/sources/2.7.9-debug/Python-2.7.9 ~/.pyenv/sources/2.7.9-debug ~
Installing Python-2.7.9...
patching file Modules/_sqlite/connection.c
Hunk #1 succeeded at 458 (offset -1 lines).
...
```

```sh-session
$ pyenv local 2.7.9-debug
$ python --version
Python 2.7.9
$ python test.py
(u'a',)
[25767 refs]
```

#### Resources:

- https://wiki.python.org/moin/DebuggingWithGdb
- http://www.brendangregg.com/blog/2016-08-09/gdb-example-ncurses.html
