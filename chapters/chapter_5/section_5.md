# Simple Threaded C Destination

In order to implement a threaded C destination, you need to create a syslog-ng module and a plugin in it.
At the end of this section you're going to be ready to start syslog-ng with the following configuration:

```C
@version: 3.28
@include "scl.conf"

source s_local {
    system();
    internal();
};

destination d_dummy {
    dummy(filename("/tmp/test"));
};

log {
    source(s_local);
    destination(d_dummy);
};
```

### Creating a module skeleton

You can use a development script that prepares a skeleton code for the destination:
```
$ dev-utils/plugin_skeleton_creator/create_plugin.sh -n dummy -k dummy -t LL_CONTEXT_DESTINATION
```

The command above will create the following files.
```
modules/dummy
├── CMakeLists.txt
├── dummy-grammar.ym
├── dummy-parser.c
├── dummy-parser.h
├── dummy-plugin.c
└── Makefile.am
```

Then you need to modify `modules/Makefile.am` and `modules/CMakeLists.txt`.

Add `include modules/dummy/Makefile.am` for `Makefile.am`, and add `mod-dummy` to `SYSLOG_NG_MODULES`.
Add `add_subdirectory(dummy)` for CMakeLists.txt.

After that the build system will build the dummy module too.

- The `dummy-grammar` and `dummy-parser` files will contain the configuration grammar. Parser codes are generated from these.
- `dummy` will implement the destination logic through interfaces.

When you are writing a destination, you should extend from one of these abstract classes: `LogThreadedDestDriver`, `LogThreadedDestWorker`.

This example shows an empty and very simple `LogThreadedDestDriver` (threaded destination driver) implementation.

Threaded destination drivers use dedicated thread instead of the main thread. So it's allowed to use blocking operations.

### Dummy Destination

Create two new files; `dummy.h` and `dummy.c`. Don't forget to add these files to `modules/dummy/Makefile.am` and `modules/dummy/CMakeLists.txt` too.

You need to extend `modules_dummy_libdummy_la_SOURCES` and `dummy_SOURCES` respectively.

#### dummy.h
The destination header file only contains the initialization functions needed by the config parser.
- `dummy_dd_new`: constructs our destination
- `dummy_dd_set_*`: option setter functions for the parser

This example has only one dummy filename option.

```C
#ifndef DUMMY_H_INCLUDED
#define DUMMY_H_INCLUDED

#include "driver.h"

LogDriver *dummy_dd_new(GlobalConfig *cfg);

void dummy_dd_set_filename(LogDriver *d, const gchar *filename);

#endif
```

#### dummy.c

This is the implementation of the destination driver. It will be built as a shared library.

```C
#include "dummy.h"
#include "dummy-parser.h"

#include "plugin.h"
#include "messages.h"
#include "misc.h"
#include "stats/stats-registry.h"
#include "logqueue.h"
#include "driver.h"
#include "plugin-types.h"
#include "logthrdest/logthrdestdrv.h"


typedef struct
{
  LogThreadedDestDriver super;
  gchar *filename;
} DummyDriver;

/*
 * Configuration
 */

void
dummy_dd_set_filename(LogDriver *d, const gchar *filename)
{
  DummyDriver *self = (DummyDriver *)d;

  g_free(self->filename);
  self->filename = g_strdup(filename);
}

/*
 * Utilities
 */

static const gchar *
dummy_dd_format_stats_instance(LogThreadedDestDriver *d)
{
  DummyDriver *self = (DummyDriver *)d;
  static gchar persist_name[1024];

  g_snprintf(persist_name, sizeof(persist_name),
             "dummy,%s", self->filename);
  return persist_name;
}

static const gchar *
dummy_dd_format_persist_name(const LogPipe *d)
{
  DummyDriver *self = (DummyDriver *)d;
  static gchar persist_name[1024];

  if (d->persist_name)
    g_snprintf(persist_name, sizeof(persist_name), "dummy.%s", d->persist_name);
  else
    g_snprintf(persist_name, sizeof(persist_name), "dummy.%s", self->filename);

  return persist_name;
}

static gboolean
dummy_dd_connect(LogThreadedDestDriver *s)
{
  msg_debug("Dummy connection succeeded",
            evt_tag_str("driver", s->super.super.id), NULL);

  return TRUE;
}

static void
dummy_dd_disconnect(LogThreadedDestDriver *d)
{
  DummyDriver *self = (DummyDriver *)d;

  msg_debug("Dummy connection closed",
            evt_tag_str("driver", self->super.super.super.id), NULL);
}

/*
 * Worker thread
 */

static LogThreadedResult
dummy_worker_insert(LogThreadedDestDriver *d, LogMessage *msg)
{
  DummyDriver *self = (DummyDriver *)d;

  fprintf(stderr, "Message sent using file name: %s\n", self->filename);

  return LTR_SUCCESS;
  /*
   * LTR_DROP,
   * LTR_ERROR,
   * LTR_SUCCESS,
   * LTR_QUEUED,
   * LTR_NOT_CONNECTED,
   * LTR_RETRY,
  */
}

static void
dummy_worker_thread_init(LogThreadedDestDriver *d)
{
  DummyDriver *self = (DummyDriver *)d;

  msg_debug("Worker thread started",
            evt_tag_str("driver", self->super.super.super.id),
            NULL);
}

static void
dummy_worker_thread_deinit(LogThreadedDestDriver *d)
{
  DummyDriver *self = (DummyDriver *)d;

  msg_debug("Worker thread stopped",
            evt_tag_str("driver", self->super.super.super.id),
            NULL);
}

/*
 * Main thread
 */

static gboolean
dummy_dd_init(LogPipe *d)
{
  DummyDriver *self = (DummyDriver *)d;

  if (!log_threaded_dest_driver_init_method(d))
    return FALSE;

  msg_verbose("Initializing Dummy destination",
              evt_tag_str("driver", self->super.super.super.id),
              evt_tag_str("filename", self->filename),
              NULL);

  return log_threaded_dest_driver_start_workers(&self->super);
}


gboolean
dummy_dd_deinit(LogPipe *s)
{
  DummyDriver *self = (DummyDriver *)s;

  msg_verbose("Deinitializing Dummy destination",
              evt_tag_str("driver", self->super.super.super.id),
              evt_tag_str("filename", self->filename),
              NULL);

  return log_threaded_dest_driver_deinit_method(s);
}

static void
dummy_dd_free(LogPipe *d)
{
  DummyDriver *self = (DummyDriver *)d;

  g_free(self->filename);

  log_threaded_dest_driver_free(d);
}

/*
 * Plugin glue.
 */

LogDriver *
dummy_dd_new(GlobalConfig *cfg)
{
  DummyDriver *self = g_new0(DummyDriver, 1);

  log_threaded_dest_driver_init_instance(&self->super, cfg);
  self->super.super.super.super.init = dummy_dd_init;
  self->super.super.super.super.deinit = dummy_dd_deinit;
  self->super.super.super.super.free_fn = dummy_dd_free;

  self->super.worker.thread_init = dummy_worker_thread_init;
  self->super.worker.thread_deinit = dummy_worker_thread_deinit;
  self->super.worker.connect = dummy_dd_connect;
  self->super.worker.disconnect = dummy_dd_disconnect;
  self->super.worker.insert = dummy_worker_insert;

  self->super.format_stats_instance = dummy_dd_format_stats_instance;
  self->super.super.super.super.generate_persist_name = dummy_dd_format_persist_name;
  self->super.stats_source = stats_register_type("dummy");

  return (LogDriver *)self;
}
```

`DummyDriver` is `LogThreadedDestDriver`. A `LogThreadedDestDriver` has at least one `LogThreadedDestWorker`. It is the worker's responsibility to send the messages.

You can have as many workers as you need, but for simplest cases one worker is enough. In that case, for the sake of simplicity, you can implement `LogThreadedDestDriver` to work in a so called `compatibility mode`. This is how `DummyDriver` is implemented in the example above. `LogThreadedDestDriver` embeds a `LogThreadedDestWorker` instance. In compatibility mode, `LogThreadedDestDriver` will use the embedded worker. If `compatibility mode` is not enough for you, then you need to override the `construct` method of `LogThreadedDestDriver`.

This unit can be separated into 5 parts:

1. configuration functions (setters)
2. utility functions
3. functions running in separate thread
4. functions running in the main thread

Functions from category 2-4 are implementations of the LogThreadedDestDriver's virtual methods.
In `dummy_dd_new`, we pass these function pointers to the base classes.

Our example overrides these 6 virtual methods:

- `new (dummy_dd_init)`: destination constructor.
- `free_fn (dummy_dd_free)`: destination destructor.
- `init (dummy_dd_init)`: It is called after startup, and after each reload. It is important to note that the init method may be called multiple times for the same driver. In case of a failed reload (for example syntax error in config), syslog-ng will resume using the same Driver instances instead of creating new ones, after calling their init method.
- `deinit (dummy_dd_deinit)`: It is called before shutdown, and before each reload.
- `thread_init (dummy_worker_thread_init)`: It is called after the init method.
- `thread_deinit (dummy_worker_thread_deinit)`: It is called before the deinit method.
- `connect (dummy_dd_connect)`: destination disconnects (on error or drop)
- `disconnect (dummy_dd_disconnect)`: destination disconnects (on error or drop)
- `insert (dummy_worker_insert)`, where you can format the received message and send to an actual destination.

### grammar

dummy-parser.c contains of a list of available keywords that can be referred in the grammar. A keyword is an integer with a string representation. The integer is defined in `dummy-grammar.ym`: see the example below. The string representation is defined in `dummy-grammar.c`.

The dummy module will support two keywords: dummy and filename. You need to replace `dummy-keywords` in `dummy-grammar.c` with:

```
static CfgLexerKeyword dummy_keywords[] =
{
  { "dummy", KW_DUMMY },
  { "filename", KW_FILENAME },
  { NULL }
};
```

### dummy-grammar

The dummy-grammar.ym file writes down the syntax of the configuration of our destination. It's written in yacc format. The bison parser generator creates parser code from this grammar.

#### dummy-grammar.ym

```C
%code requires {

#include "dummy-parser.h"

}

%code {

#include "dummy.h"

#include "cfg-grammar.h"
#include "plugin.h"
}

%name-prefix "dummy_"
%lex-param {CfgLexer *lexer}
%parse-param {CfgLexer *lexer}
%parse-param {LogDriver **instance}
%parse-param {gpointer arg}

/* INCLUDE_DECLS */

%token KW_DUMMY
%token KW_FILENAME

%%

start
        : LL_CONTEXT_DESTINATION KW_DUMMY
          {
            last_driver = *instance = dummy_dd_new(configuration);
          }
          '(' dummy_option ')' { YYACCEPT; }
;

dummy_options
        : dummy_option dummy_options
        |
        ;

dummy_option
        : KW_FILENAME '(' string ')'
          {
            dummy_dd_set_filename(last_driver, $3);
            free($3);
          }
        | threaded_dest_driver_option
;

/* INCLUDE_RULES */

%%
```
