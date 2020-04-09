# lowRISC SystemVerilog Coding Style Guide for Design Verification

This document is a supplemental Style Guide to the [lowRISC Verilog style
guide](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md),
with emphasis on writing code for Design Verification (DV) in SystemVerilog,
following the [UVM methodology](https://www.accellera.org/images//downloads/standards/uvm/uvm_users_guide_1.2.pdf).

## Table of Contents

* [Introduction](#introduction)
* [Naming and Style](#naming-and-style)
* [Directory Structure and Packages](#directory-structure-and-packages)
  * [UVM Agent Directory Structure](#uvm_agent-directory-structure)
  * [UVM Env Directory Structure](#uvm_env-directory-structure)
* [UVM Guidelines](#uvm-guidelines)
  * [Class Definitions](#class-definitions)
  * [UVM new functions](#uvm-new-functions)
  * [uvm_scoreboard usage](#uvm_scoreboard-usage)
  * [uvm_agent usage](#uvm_agent-usage)
  * [uvm_driver usage](#uvm_driver-usage)
  * [Macro Usage](#macro-usage)
    * [Guidelines for UVM library macros](#guidelines-for-uvm-library-macros)
    * [Guidelines for macros within dv_macros.svh](#guidelines-for-macros-within-dv_macrossvh)
    * [General macro use guidelines](#general-macro-use-guidelines)
    * [Typical gotchas with macro usage](#typical-gotchas-with-macro-usage)
  * [Factory](#factory)
  * [Configuration Mechanism](#configuration-mechanism)
  * [Sequences](#sequences)
  * [Sequence Items](#sequence-items)
  * [RAL Usage](#ral-usage)
  * [Objections and Coordinating End of Test](#objections-and-coordinating-end-of-test)
  * [Logging and Print Messages](#logging-and-print-messages)
  * [DPI and C Connections](#dpi-and-c-connections)
* [SystemVerilog Language Features](#systemverilog-language-features)
  * [Function Declarations](#function-declarations)
  * [Randomization](#randomization)
  * [Enums](#enums)
  * [Interfaces, Clocking Blocks, Modports](#interfaces,-clocking-blocks,-modports)
  * [Loop Operators](#loop-operators)
  * [Code Within Asserts](#code-within-asserts)
  * [Wait and Fork](#wait-and-fork)
  * [Void Casts](#void-casts)
  * [Associative Arrays](#associative-arrays)
  * [Bind Statements](#bind-statements)
  * [Simulator Specific Code](#simulator-specific-code)
  * [Forbidden System Tasks and Functions](#forbidden-system-tasks-and-functions)



## Introduction

This guide defines the Comportable style for DV-specific SystemVerilog code. The
goal is to use this common set of guidelines when implementing testbenches using
the UVM methodology to allow all developed Comportable DV code to be uniform,
reusable, and portable.
This document assumes previous experience and working knowledge of UVM, and
as such all included code snippets will be fairly concise.


## Naming and Style

Refer to [lowRISC Verilog naming
conventions](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md#naming)
for any general naming style guidelines. In general, common UVM testbench
components must be declared and assigned handles as follows:

| Testbench component               | Style                                        |
| --------------------------------- | -------------------------------------------- |
| Virtual interfaces                | `virtual <if_name>_if <if_name>_vif;`        |
| Environment                       | `<dut>_env env;`                             |
| Env level config object           | `rand <dut>_env_cfg cfg;`                    |
| Env coverage collection object    | `<dut>_env_cov cov;`                         |
| Agent                             | `<dut>_agent m_<dut>_agent;`                 |
| Agent level config object         | `rand <dut>_agent_cfg m_<dut>_agent_cfg;`    |
| Agent coverage collection object  | `<dut>_agent_cov cov;`                       |
| Driver                            | `<dut>_driver driver;`                       |
| Monitor                           | `<dut>_monitor monitor;`                     |
| Scoreboard                        | `<dut>_scoreboard scoreboard;`               |
| Virtual sequencer                 | `<dut>_virtual_sequencer virtual_sequencer;` |
| Sequencer                         | `<dut>_sequencer <dut>_sequencer;`           |

Additional naming guidelines:

1.  The top level testbench file that instantiates the DUT must be named `tb.sv`.
    The testbench module must be named `tb`, and the instance name of the DUT in
    `tb.sv` must be `dut`. Using this standardized naming allows universal usage
    of any supporting DV sources (coverage cfg files, and so on).
2.  When instantiating any objects, the `name` argument to `type_id::create()`
    must match the name of the variable being assigned to.
    This ensures that any reporting messages refer to the right objects.
3.  To avoid name collisions, it is recommended that global types and types defined
    within a package have unique prefixes. This includes typedefs, enumerated
    types/values, package parameters, package localparams, any functions/tasks, and
    macros. This prefixing is not necessary for any types defined within a class,
    module, or interface, as they will be scoped by the parent entity.
4.  Note that all package variables (parameters, etc.) must have a specified type.
5.  When instantiating a user-defined class member variable that is a class object,
    it is recommended (but not necessary) to follow this prefixing scheme:
    *   Prefix the instance name of the class object with `m_`, as such:

        ```systemverilog
        my_animal m_my_animal;
        ```

        This scheme does not apply to any class member variables that are
        not themselves class objects.

    *   If multiple instantiations of the class are desired without an array
        to store them, use `<class_name> m_<instance_name>` to declare a variable,
        and use an informative instance name.

        ```systemverilog
        my_animal m_my_cat;
        my_animal m_my_dog;
        my_animal m_my_fish;
        ```

    *   If an array of class handles or `logic`  is desired, choose a name such
        that it follows one of the following schemes:

        ```systemverilog
        int NUM_ANIMALS = 10;
        // This plural convention is recommended, but both are equally acceptable
        my_animal m_my_animals[NUM_ANIMALS];
        my_animal m_my_animal_array[NUM_ANIMALS];
        ```


## Directory Structure and Packages

Define one class per file, and the filename should match the name of the class.
The class files must be `` `include``-ed into the package file.
Directory structure of the UVM agent and UVM environment testbench components
along with their related files are shown below, with some additional guidance on
declaring functions from a package context.




### `uvm_agent` directory structure

* All DV code for a particular agent will be placed under a single directory
`<block>_agent/` separate from the DV code for the corresponding environment.
* It is highly recommended that all agent directories be placed in a common area
  to create a repository of UVM agents that allows for more vertical reuse.

```systemverilog
<block>_agent/
  <block>_if.sv
  <block>_item.sv
  <block>_agent.sv
  <block>_agent_pkg.sv
  <block>_agent_cfg.sv
  <block>_agent_cov.sv
  <block>_driver.sv
  <block>_host_driver.sv (Only if block requires a separate driver for host mode)
  <block>_device_driver.sv (Only if block requires a separate driver for device mode)
  <block>_monitor.sv
  <block>_sequencer.sv
  seq_lib/
    <block>_base_seq.sv
```


### `uvm_env` directory structure
* All DV code for the UVM environment, UVM tests, and top level block
  testbench will be placed in a directory `<dut>/dv/` in three corresponding
  subdirectories `<dut>/dv/env/, <dut>/dv/tests/, <dut>/dv/tb/`.
* The `<dut>_env_pkg` should import the `<dut>_agent_pkg`.
* All UVM virtual sequences for a given DUT will be placed under `env/seq_lib/`,
  along with a `<dut>_vseq_list.sv` file that `` `include``-es all of the virtual
  sequence files. This virtual sequence list file must be `` `include``-ed by
  `env/<dut>_env_pkg.sv`.
* If applicable, `env/<dut>_env_pkg.sv` must import the `<dut>_agent_pkg`.
* `tests/<dut>_test_pkg.sv` must import the `<dut>_env_pkg` package.
* All files `` `include``-ed into a single package must be in the same single directory.

```systemverilog
<dut>/dv/
  env/
    <dut>_env.sv
    <dut>_env_pkg.sv
    <dut>_env_cfg.sv
    <dut>_env_cov.sv
    <dut>_scoreboard.sv
    <dut>_virtual_sequencer.sv
    seq_lib/
      // These allow basic functionality to be added to the environment
      <dut>_base_vseq.sv
      <dut>_sanity_vseq.sv
  tb/
    tb.sv
  tests/
    <dut>_base_test.sv
    <dut>_test_pkg.sv
```
`<block>_agent_pkg.sv, <dut>_env_pkg.sv, <dut>_test_pkg.sv, and tb/tb.sv`
  must include these `` `include``-es and imports:

```systemverilog
`include "dv_macros.svh"
`include "uvm_macros.svh"
import uvm_pkg::*;
```

An example environment level package file for an arbitrary block `newblock` should look similar to this:

```systemverilog
package newblock_env_pkg;
  // dep packages
  import uvm_pkg::*;
  import newblock_agent_pkg::*;

  // macro includes
  `include "uvm_macros.svh"
  `include "dv_macros.svh"

  // define any parameters
  parameter int NEW_BLOCK_PARAMETER = 10;

  // define any local types/functions/tasks
  typedef enum bit {
    NewblockBooleanZero,
    NewblockBooleanTrue
  } newblock_boolean_e;

  // Functions declared within packages must be automatic
  function automatic bit newblock_test_function(newblock_boolean_e test_boolean);
    //function definition
  endfunction
endpackage
```


## UVM Guidelines

Always call `run_test()` without the test name argument. This allows command
line test overrides to be performed using `+UVM_TESTNAME=<testname>` without
having to recompile and allows testlists to be created for regression runs.


### Class Definitions

1.  After variable declarations, register any objects and components with the
    factory using the `uvm_component_utils` and `uvm_object_utils` macros.
    *   Declare any member variables to be registered with the UVM factory field
    automation macros `uvm_field_int, uvm_field_enum, uvm_field_object,
    uvm_field_array_int`, etc. within `uvm_object/component_utils_begin` and
    `uvm_object/component_utils_end`.
2.  For all field automation macros, use `UVM_DEFAULT` as the flag argument.
    For example, if it is desired that everything except comparing and printing
    be enabled for some `rx_delay` field, the macro would look like this:

    ```systemverilog
    `uvm_field_int(rx_delay, UVM_DEFAULT | UVM_NOCOMPARE | UVM_NOPRINT)
    ```

3.  After factory registration, a constructor must be defined. Use the
    `uvm_object_new` macro for UVM objects and the `uvm_component_new` macro for UVM components.
    Both of these macros are defined in `dv_macros.svh`.
4.  Any constructors written manually should contain a call to
    `super.new(name [,parent])` as the first line. Any other lines in the
    constructor are optional.
5.  Do not add any extra arguments to the constructor other than `name` and `parent`.
6.  All objects should generally be instantiated in the `build_phase()` function
    for components, or in the beginning of the `body()` task for sequences, unless
    there is a good reason not to.
    This is to enable proper usage of `uvm_factory` overrides.
7.  When extending any user-defined component class, `super.<phase_name>_phase`
    must be called for every phase method that is being overridden.
8.  Transaction classes are usually logical representations of the data
    communicated over one or more interfaces executing a particular protocol.
    These classes:
    *   Should extend `uvm_sequence_item`.
    *   Should contain only information that is relevant to the data being
        transmitted over an interface, and not to the means of transmit.
    *   Should result in a legal transaction when randomized.


### UVM `new()` functions

The `new()` function of any class derived from `uvm_object` must have the
default value of its `name` argument set to `""`, since the default name
argument will be set by `type_id::create()`.
The `new()` function of any class not derived from `uvm_object` must not
have default values set for its arguments.
Examples:

:+1:
```systemverilog
// class derived from uvm_object with correct default value
function new (string name = "");
  super.new(name);
endfunction : new

// class derived from uvm_component with no default values
function new(string name, uvm_component parent);
  super.new(name, parent);
endfunction : new
```

:-1:
```systemverilog
// Incorrect default value used for class derived from uvm_object
function new(string name = "my_rand_seq");
  super.new(name);
endfunction : new

// Incorrect because default values for class derived from uvm_component are provided
function new(string name = "", uvm_component parent);
  super.new(name, parent);
endfunction : new
```


### `uvm_scoreboard` usage

Use `uvm_tlm_analysis_fifo` - can be directly connected to analysis ports,
and has an unbounded size so that writes will always succeed without blocking,
in situations where transactions need to be stored for some time before being
consumed by the scoreboard for processing.


### `uvm_agent` usage

1.  Interface agents should contain only a sequencer, driver, monitor, and config object.
2.  Use one agent for every interface of the DUT, with an optional sequencer and
    driver whose instantiation is determined by the value of `<agent_cfg>.is_active`.
    This is placed in the agent config object to allow for better access throughout the testbench.
3.  For any DUT interfaces that have multiple operation modes
    (such as host and device modes), two drivers should be created, one for each mode,
    and arbitration done in the `uvm_agent::build_phase` function to determine
    which driver to instantiate.
    Each driver should have different sequences that correspond to it.


### `uvm_driver` usage

1.  A driver must only communicate with one sequencer on one SystemVerilog
    interface, and should not have any analysis ports.
2.  A driver should not randomize any data received from transaction
    items sent through analysis ports.
3.  A driver must assign `X` to any buses it controls during "don't-care" clock
    cycles in which no valid transaction is present.


### Macro Usage

The UVM library provides several macros as shortcuts for commonly used sequences
of code. In addition to this, the provided `dv_macros.svh` file defines other
utility macros for even more common code sequences.

#### Guidelines for UVM library macros

1.  All lowercase `` `uvm_... `` expressions denote library macros, and do not
    require trailing semicolons.
2.  `uvm_object_*` and `uvm_component_*` macros
    *   Use these macros to register classes with the `uvm_factory`.
3.  `uvm_field_*` macros
    *   Use these macros for members of sequence items. While these macros
        implement expensive versions of common functions required for sequence items,
        the overhead is generally not worth implementing custom versions
        for every sequence item. Additionally, these macros add a great deal of
        readability and help avoid coding mistakes with custom implementations.
        Only implement custom versions of these macros if performance becomes a
        very noticeable issue. Do not use these macros with classes derived from `uvm_component`.
4.  Do not use `uvm_do` macros. Instead, start sequence items on a sequencer
    using the following high level steps:
    *   `start_item(<sequence_item>, <priority>)`.
    *   Randomize the sequence item.
    *   `finish_item(<sequence_item>, <priority>)`.
    *   `get_response(<response>)`.

#### Guidelines for macros within `dv_macros.svh`

1.  UVM print macros - `gfn`, `gtn`, `gn`, `gmv`
    *   These macros provide a concise wrapper for `get_full_name()`,
        `get_type_name()`, `get_name()`, and `uvm_reg.get_mirrored_value()`, respectively.
        By using these it is possible to drastically reduce line length
        and improve readability.
2.  `downcast(<extended_class_handle>, <base_class_handle>)`
    *   Use this macro to cast a handle for any base class object that
        contains a handle to an extended class to the extended class type, useful
        for situations that require polymorphism.
3.  `uvm_object_new` and `uvm_component_new`
    *   Use these macros to include the `new()` constructor for classes
        derived from `uvm_object` and `uvm_component`, respectively.
        If any additional logic must be included in the constructor,
        these macros must not be used, and a constructor must be written manually.
4.  `uvm_create_obj` and `uvm_create_comp`
    *   Use these macros to call `create(<instance_name>)` and
        `create(<instance_name>, this)`, respectively.
5.  `DV_CHECK_*` comparison checking macros
    *   These macros must be used to perform checks on the input arguments
        and invoke one of the `uvm_info`, `uvm_error`, or `uvm_fatal`
        reporting macros if the check fails.
        These are much more concise than hand-writing the comparison checks and
        the corresponding UVM report macro and provide efficient readability.
    *   The severity of the invoked report macro can be specified with the
        `<uvm_severity>` argument, the default reporting macro is `uvm_error`.
    *   Append `_FATAL` to the end of these macro names to invoke variants of
        these macros that will automatically throw fatal errors upon check failure
        (e.g. `DV_CHECK_EQ_FATAL`, `DV_CHECK_LE_FATAL`, etc..).
6.  Randomization macros
    *   `dv_macros.svh` provides three sets of macros which must be used for
        all randomization functionality:
        *   `DV_CHECK_RANDOMIZE_FATAL` - Shorthand for `foo.randomize()`
            with a `uvm_fatal` check.
        *   `DV_CHECK_STD_RANDOMIZE_FATAL` - Shorthand for
            `std::randomize(foo)` with a `uvm_fatal` check.
        *   `DV_CHECK_MEMBER_RANDOMIZE_FATAL` - Shorthand for
            `this.randomize(foo)` with a `uvm_fatal` check.
    *   Variants of all three macros that allow constraints to be specified
        also exist, these macros are invoked by inserting `WITH` into the macro name:
        `DV_..._RANDOMIZE_WITH_FATAL`.
7.  `DV_EOT_PRINT_..._CONTENTS` TLM fifo display macros
    *   It is recommended to use these macros to display the status of the
        various TLM fifos and queues in the scoreboard at the end of the test
        to ensure that those data structures have been emptied. There should be no
        pending transactions that have not yet been compared before the simulation
        ends.
        If used, these macros must be used during the `check_phase`.
8.  `DV_SPINWAIT`
    *   It is recommended to use this macro in situations that require an
        isolation fork to disable some process after waiting for a specified
        amount of time, as it provides proper process isolation such as the example
        below:

        ```systemverilog
        // forked threads should be labeled if they serve a specific purpose,
        // like the isolation thread below
        fork : isolation_fork
          begin
            fork
              begin
                <specified code to execute>
              end
              begin
                <watchdog timer logic with error reporting>
              end
            join_any
            disable fork;
          end
        join
        ```

#### General macro use guidelines

1.  Macro Naming:
    *   If the macro wraps some fundamental UVM functionality and deals
        directly with UVM functions, like `uvm_component_new`, and so on, the
        macro must be named all in `lower_snake_case`.
    *   If the macro expands into a block of code that does not deal directly
        with UVM functionality, and is for convenience, like the
        `DV_CHECK_*` macros, the macro must be named in `UPPER_SNAKE_CASE`.
2.  Macro vs. Parameters:
    *   If the intent is to define a constant that is a basic data type or
        that is the result of an expression involving basic data types, a
        parameter (or localparam) must be used, depending on where the
        constants are being defined.
    *   If the intent is to abstract away a code snippet or to represent a
        slice of an array, a macro must be used.
        If the macro is defined in a local perspective, it must be undefined at
        the end of the file, otherwise it will pollute the global namespace.

#### Typical gotchas with macro usage:

While it is better to avoid macros altogether, they do make our lives easier by
shortening pieces of repeated code, in a manner that cannot be achieved with a
function or a task. In general, the following describes the best practices to
avoid common gotchas with macro usage.

1.  Always wrap macro arguments in parentheses. If the macro evaluates to an
    expression, also wrap the whole expansion in parentheses.The following
    example illustrates how failing to do this can cause an unexpected
    behavior.

    :-1:
    ```systemverilog
    `define AND(a, b) a && b

    assign x = `AND(p || q, r) || s;
    // Bad: Wrong operator precedence! This results in: p || (q && r) || s
    //      which is not equal to ((p || q) && r) || s.
    ```
    :+1:
    ```systemverilog
    `define AND(a, b) ((a) && (b))

    assign x = `AND(p || q, r) || s;
    // Good: Output in-line with the expectation.
    ```
2.  Macro usage scope and avoiding namespace collisions.

    Macros have a global scope regardless of where they are defined. Once
    compiled, they are visible to ALL subsequent sources in the same
    compilation unit as well as to all subsequent compilation units. Hence,
    care must be taken to prevent macro re-definitions. Some tools do not even
    warn about name collisions, causing unnecessary debug overhead.

    *   Macro names must be appropriately prefixed with a thematically chosen
        string. It is typical to use the file name or the package name as the
        prefix string for all macros defined within that source.

        :-1:
        ```systemverilog
        // file: dv_macros.h
        `define STRINGIFY(s) `"s`"
        // Bad: UVM library code also defines a macro with the same name.
        //      Depending on the tool, this may be flagged as an error.
        ```

        :+1:
        ```systemverilog
        // file: dv_macros.h
        `define DV_STRINGIFY(s) `"s`"
        ```

    *   If the scope of the macros defined are global in nature (useful for
        the entire testbench), then place them in a separate SystemVerilog
        header file with `` `ifndef`` guards. It must only contain macro
        definitions and nothing else. The file name must be suffixed with
        `_macros` and have the `.svh` extension to denote that it is a header
        file.

        :-1:
        ```systemverilog
        // file: dv_macros.h
        automatic function void get_nco();
          ...
        endfunction
        // Bad: Putting functions and tasks in a header file that is potentially
        //      included multiple times.

        `define DV_FOO(...) ...
        // Bad: No ifndef guard. Macro re-definition warnings will be thrown if
        //      this header file is included in multiple sources.
        ```

        :+1:
        ```systemverilog
        // file: dv_macros.svh
        `ifndef DV_FOO
          `define DV_FOO(...) ...
        `endif

        `ifndef DV_BAR
          `define DV_BAR xyz
        `endif
        ```

    *   If the scope of the macros defined is limited to a package or source
        (individual interface, module or a class), then place them at the top of
        the file. Always `` `undef`` them at the end of the file so that they
        are no longer visible to subsequent sources during compilation. Do not
        put `` `ifndef`` guards on these macros, so that the re-definitions
        if encountered, are flagged.

        :-1:
        ```systemverilog
        // file: foo_init_seq.sv
        `ifndef GET_DATA
          `define GET_DATA(baz) ...
        `endif
        // Bad: Macro-redefinition if encountered, will be ignored due to the
        //      ifndef guard. Previously defined macro is invoked instead.
        // Bad: Macro name is not specific enough.

        class foo_init_seq;
          ...
        endclass
        // EOF
        // Bad: Macro is not undefined at the end of the file.
        ```

        :+1:
        ```systemverilog
        // file: foo_init_seq.sv
        `define GET_FOO_TX_FIFO_DATA_AFTER_OP(baz) ...

        class foo_init_seq;
          ...
        endclass

        `undef GET_FOO_TX_FIFO_DATA_AFTER_OP
        ```

        :+1:
        ```systemverilog
        // file: foo_env_pkg.sv
        `define GET_FOO_TX_FIFO_DATA(baz) ...

        ...

        // Package sources
        `include "foo_env_cfg.sv"
        ...
        `include "foo_scoreboard.sv"
        `include "foo_env.sv"


        `undef GET_FOO_TX_FIFO_DATA
        // Good: Only the package sources can invoke this macro. It is not
                 visible in any other code.
        ```

3.  Wrap macros that resolve to multiple statements within `begin` and `end`.

    Multi-statement macros are typically problematic when invoked in a single
    line conditional expression without `begin`..`end`. It is better to preempt
    this bug by wrapping the macro definition itself in `begin`..`end`.

    :-1:
    ```systemverilog
    `define UPDATE_VALUES(a, b) \
      a = 88;                   \
      b = 1955;

    if (power_gw == 1.21) `UPDATE_VALUES(speed, year)
    // Great Scott! The year is unconditionally set to 1955.
    ```

    :+1:
    ```systemverilog
    `define UPDATE_VALUES(a, b) \
      begin                     \
        a = 88;                 \
        b = 1955;               \
      end

    if (power_gw == 1.21) `UPDATE_VALUES(speed, year)
    // Good: Both values are set only if the condition is met.
    ```

Please see this [paper](https://www.veripool.org/papers/Preproc_Good_Evil_SNUGBos10_pres.pdf)
for more details and examples.

### Factory

To use the UVM factory, these guidelines should be followed:

1.  All classes must be registered with the factory. Use the
    `uvm_object_utils` and `uvm_component_utils` macros to register all
    non-parameterized classes, and use `uvm_object_param_utils` and
    `uvm_component_param_utils` to register all parameterized classes.
2.  The `create()` API must be used to create all objects and components.
    Exceptions for this rule are any TLM related objects and covergroup objects,
    which should be created by directly calling `new()`, as they should not be
    registered with the factory.
3.  When using the factory to create objects with `<type_name>::type_id::create()`,
    the `<type_name>` must be explicitly declared, parameters representing types
    must not be used here.
4.  When using the factory to create objects, it is recommended to use the
    same name for both the name of the object and the variable used as a handle to
    the object to prevent unnecessary confusion from error messages.

    ```systemverilog
    // create a single agent
    m_myblock_agent = myblock_agent::type_id::create("m_myblock_agent", this);
    ```

5.  When using the factory to create an array of objects, decorate the name of
    the object in the `create()` call with the index of the object in the array.

    ```systemverilog
    // create an array of objects
    foreach (myobject_array[i]) begin
      myobject_array[i] = myobject::type_id::create($sformatf("myobject_array[%0d]", i), this);
    end
    ```

6.  For classes derived from `uvm_component`, the `parent` argument should be
    the keyword `this`, so that the object hierarchy can be built correctly.

    ```systemverilog
    m_axi_driver = axi_driver::type_id::create("m_axi_driver", this);
    ```

7.  For classes derived from `uvm_object`, use `gfn` as part of the name
    argument to append the full hierachy name, since `uvm_object` hierarchies are
    not automatically built. Note that this only applies to class instances that
    are not instantiated within a `uvm_component` in the test hierarchy,
    as in this case the full hierarchy name will already be provided.

    ```systemverilog
    m_my_txn = my_txn::type_id::create({`gfn, ".m_my_txn"});
    ```

8.  All type or instance overrides must be in place before creating the class instance.
    Factory usage examples:

    ```systemverilog
    // Override all drivers in testbench with my_driver
    factory.set_type_override_by_type(current_driver::get_type(), my_driver::get_type());

    // Override all drivers in testbench with my_driver - alternative syntax
    current_driver::type_id::set_type_override(my_driver::get_type());

    // Override specific instance of current_driver with my_driver
    factory.set_inst_override_by_type("env1.m_agent1.driver", current_driver::get_type(), my_driver::get_type());

    // Override specific instance of current driver - alternative syntax
    current_driver::type_id::set_inst_override(my_driver::get_type(), "env.m_agent1.driver");
    ```


### Configuration Mechanism

When using the UVM configuration database, these guidelines should be followed:

1.  The `uvm_config_db` API must be used. Do not use `uvm_resource_db`. Do not use
    `set/get_config_object()`, `set/get_config_string()`, or `set/get_config_int()`,
    these have been deprecated.
2.  In general, a designated configuration object extended from `uvm_object`
    should be used for every `uvm_env` and `uvm_agent` component in the testbench.
3.  It is recommended that the environment's configuration object contain the
    agent's configuration object.
4.  The `uvm_config_db` API should be used to pass configuration objects to
    locations where they are needed. It should not be used to pass integers,
    strings, or other basic data types, since it is much easier for namespace
    collisions to occur when using lower level types.
5.  Checks for success must always be performed when using the `uvm_config_db`.
    If the lookup fails, issue a `uvm_fatal` to terminate the simulation.
    This should be done as follows:

    ```systemverilog
    if (!uvm_config_db#(...)::get(...)) begin
      `uvm_fatal(...)
    end
    ```
6.  Wildcards must not be used in the field argument of `uvm_config_db::set(...)` calls.
    It is allowed to use wildcards in the inst_name argument of these calls,
    with proper precautions.


### Sequences

1.  It is recommended to keep sequences as generic as possible and to avoid
    writing directed sequences unless absolutely necessary.
2.  It is recommended to avoid explicit delays in terms of absolute time to
    make code emulation friendly. Delays in terms of clock cycles are allowed.
3.  The `body()` method should only execute the raw functional behavior of
    the sequence.
    Any associated housekeeping code should be placed elsewhere, such as in the
    `pre_start()` and `post_start()` methods.
    *   `pre_start()` and `post_start()` must be used instead of `pre_body()`
        and `post_body()`, as these will always be called, even for subsequences
        called from a parent sequence.
4.  When creating a sequence object, always randomize the sequence before starting it.
5.  Virtual sequences must be used to coordinate the behavior of multiple agents and multiple sequences.
6.  Virtual sequences must be started on the null sequencer or on a valid virtual sequencer.


### Sequence Items

It is recommended, but not necessary, to create small "unit tests" for
transactions objects to ensure the randomization constraints are working as
intended.


### RAL Usage

When using the UVM RAL model, care must be taken when using `uvm_reg::set()` and
`uvm_reg::update()`, as these are both non-atomic operations that can easily
cause race conditions if there are any parallel threads.


### Objections and Coordinating End of Test


As a general principle, a phase should not end when either there is more stimulus to
send during that phase or the DUT has yet to respond to some stimulus that has
been sent.

To prevent a phase from ending when there is more stimulus to send, a test class
must raise and lower objections.

To prevent a phase from ending before the DUT has responded to some stimulus,
classes derived from `uvm_component` should raise and lower objections  in their
`phases_ready_to_end()` method.

It is also recommended to only include objections in the monitor component.
When using this approach, these guidelines should be followed:

1.  The monitor's `phase_ready_to_end()` method should implement a watchdog timer
    that raises an objection if any traffic is seen on the bus and lowers it once
    no traffic is seen within the timeout period. If traffic is seen, the
    watchdog resets its timer.
    Refer to the example below:

    ```systemverilog
    // Example code for the monitor class.
    // This code assumes that the associated configuration object has an integer
    // property "ok_to_end_delay_ns", which gives the window timeout in
    // nanoseconds.

    protected bit ok_to_end = 1'b1;
    protected bit watchdog_done = 1'b0;

    function void phase_ready_to_end(uvm_phase phase);
     if (phase.is(uvm_run_phase::get())) begin
       if (watchdog_done) fork
         monitor_ready_to_end();
       join_none
       if (!ok_to_end || !watchdog_done) begin
         phase.raise_objection(this, $sformatf("%s objection raised", `gfn));
         fork
           begin
             // wait until ok_to_end is set
             watchdog_ok_to_end();
             phase.drop_objection(this, $sformatf("%s, objection dropped",
             `gfn));
           end
         join_none
       end
     end

    endfunction

    // This watchdog waits for ok_to_end_delay_ns nanoseconds while checking for
    // any traffic seen on the bus during this period.
    // If traffic is seen, the watchdog will restart until it sees no traffic.
    task watchdog_ok_to_end();
      fork
        begin : isolation_fork
          bit watchdog_reset;
          fork
            forever begin
              // check bus interface for traffic
              @(ok_to_end or watchdog_reset);
              if (!ok_to_end && !watchdog_reset) watchdog_reset = 1;
            end
            forever begin
              #(cfg.ok_to_end_delay_ns * 1ns);
              if (!watchdog_reset) begin
                break;
              end else begin
                watchdog_reset = 0;
              end
            end
          join_any;
          disable fork;

          watchdog_done  = 1;
        end : isolation_fork
      join
    endtask

    // This task is invoked as a non-blocking thread when phase_ready_to_end()
    // is first entered.
    //
    // This task should monitor the DUT's interface and control ok_to_end: it
    // should be cleared whenever there are any pending transactions and
    // set to 1 if there are no pending transactions.
    virtual task monitor_ready_to_end();
    endtask
    ```

2.  It is recommended not to use other objections in the scoreboard; the
    scoreboard should just make use of information already available to it
    (like fifo size, etc...) to determine when it is ready to end, as the
    monitor has already guaranteed that no more traffic is seen on the bus.
    These fifo checks should be checked during its `check_phase()`.

The more general guidelines below should be followed when dealing with
end-of-test timeouts or objections:

1.  When calling `raise_objection` or `drop_objection`, a string must be passed
    as a second argument to describe the objection to help with debugging.

    ```systemverilog
    phase.raise_objection(this, $sformatf("%s objection raised", `gfn));
    ...
    phase.drop_objection(this, $sformatf("%s objection dropped", `gfn));
    ```

2.  It is recommended to use the built-in UVM timeout mechanism,
    `uvm_root::set_timeout(<time_limit>, 1)`.
    The base test in every testbench should specify a default timeout limit,
    which can be overridden by every derivative test.
    A standard plusarg `+UVM_TIMEOUT=...` is recommended to be used for this
    override mechanism.


### Logging and Print Messages

1.  Print messages sent to the console should be kept to a minimum.
    Logs should be as concise as possible, by only writing key information when
    activity is present.
2.  Whenever possible, log any data directly to log files instead of the console.
    It is recommended that only critical information/errors be sent to both the
    console and the log files.
3.  Logging messages should be architected to be parsing "friendly"
    *   All related information should be kept on a single line.
    *   Only the relevant data should be logged, for example, there is no need
    to log all fields of a transaction object when only a small subset is
    valid and relevant to the transaction being performed.
    *   Log files should format lines consistently, starting with the
    simulation time that the logging event occurs, enabling easier post-processing.
    *   The build in `print()` function of `uvm_object` classes is not
    parsing "friendly", it is recommended not to use this function.
4.  Do not use the `uvm_report_*` messaging functionality, or `$display` to
    print any messages.
    The `uvm_{info/error/fatal}` macros must be used instead.
5.  Do not use the `uvm_warning` macro, use `uvm_error` or `uvm_fatal` instead.
6.  The first argument to these report macros must be one of the `gfn` or
    `gtn` shorthand macros used in place of `get_full_name()` and `get_type_name()`.
    This will allow for easier debugging should something go wrong.
7.  The `UVM_NONE` and `UVM_FULL` verbosity levels must not be used for any
    report messages.
    Recommended criteria for the choice of `VERBOSITY` value for `uvm_info`
    messages:
    *   `UVM_LOW` - important messages that occur a small number of times
        during a simulation.
        *   Instantiation of testbench components.
        *   Reset assert/deassert, start/end of DUT initialization.
        *   Start/end of traffic generation.
    *   `UVM_MEDIUM` - slightly more detailed messages.
        *   Contents of CSR fields during CSR transactions.
        *   Reporting of significant events inside the DUT - fifo full,
        fifo empty, and so on.
    *   `UVM_HIGH` - Any information about DUT activity, reference model,
        and testbench that could be helpful for monitoring stimulation progress.
        *   Parsing/interpretation of fields in requests to the DUT and reference model.
        *   Read/write operations to a memory model.
        *   Status of transactions over SystemVerilog interfaces.
        *   Values of data output by the DUT.
    *   `UVM_DEBUG` - Extremely specific information used for debugging
        failures.
        *   Any CSR exclusions.
        *   Interface buses used to drive the interface protocol.


### DPI and C Connections

1.  DPI and native SystemVerilog calls have the same semantics and
    are thus indistinguishable.
    To preserve readability, a decorator must be added as such:
    *   Functions/tasks exported from SV to C must be prefixed with `sv_dpi_`.

        ```systemverilog
        export "DPI" sv_dpi_adder = adder_function;
        ```

    *   Functions imported from C to SV must be prefixed with `c_dpi_`.

        ```systemverilog
        extern int c_dpi_adder(const int a, const int b);
        ```

2.  DPI usage is recommended in the following scenarios:
    *   Interfacing with C/C++ based reference models.
    *   Usage of optimized C/C++ libraries.
    *   Emulation.
3. DPI usage is not recommended when the functionality can be handled natively
    in SV.
4.  General DPI coding guidelines
    *   Do not cross the language boundary frequently.
    *   If a SV->C call consumes time and the C routine is not thread safe,
    protect the call with a semaphore to avoid synchronization issues.
    *   In general, pass only basic data types as arguments to DPI calls.


## SystemVerilog Language Features

### Function Declarations

Functions declared within packages, as well as any other static entities such as
modules and interfaces, must be explicitly scoped with either the `static` or
`automatic` keyword.

### Randomization

Good constraint coding style is important for faster simulations and better
simulation performance.

1.  Give all constraints a name that matches with what is being constrained, and
    suffix the constraint name with `_c`.

    ```systemverilog
    constraint num_packets_c {
      num_packets < 8;
    }
    ```
2.  Avoid using loops in constraints whenever possible. In some cases, `foreach`
    can be replaced by `inside`.

    :-1:
    ```systemverilog
    constraint con_c {
      foreach (data[i]) {
        a != data[i];
      }
    }
    ```

    :+1:
    ```systemverilog
    constraint con_c {
      !(a inside {data});
    }
    ```

3.  If using loops in constraints is necessary, avoid doing calculations inside
    the loops.
4.  Whenever possible, use bitmasking operations over modulus operations, as
    bitmasking operations are much faster.
5.  If a constraint becomes too complex, it is highly recommended to split the
    constraint and separately randomize all relevant variables.
6.  When randomizing an array of objects, it is generally recommended to not
    directly randomize the array, but to iterate through it and directly
    randomize each object to see better simulation performance.


### Enums

Enum names should be written in `lower_snake_case`, and should include the block
name for clarity.
The enumerated values should be written in `UpperCamelCase` and should include
the name of the enum for clarity.

```systemverilog
typedef enum bit {
  UartInterruptFrameErr,
  UartInterruptRxBufferFull,
  ...
} uart_interrupt_e;
```


### Interfaces, Clocking Blocks, Modports

1.  Do not use the `program` construct.
2.  Clocks and reset must be generated in SystemVerilog modules, interfaces,
    or in the UVM component hierarchy.
    Do not do this generation in `uvm_object` derived classes.
3.  Use clocking blocks in a SystemVerilog interface to sample and drive
    synchronous DUT interfaces.
4.  It is recommended to implement a reusable interface scheme for block level
    testbenches that will be included in integration/system level testbenches.


### Loop Operators

It is highly recommended to use `foreach` loops as often as possible over any
equivalent loop operators, as they are much more concise and less prone to any
subtle errors.


### Code Within Asserts

For any randomization that is required, the macros specified in the
[Randomization Macros section](#macro-usage) must be used. Do not use `assert()`
to check manual randomization calls, use the provided macros instead.
More broadly, do not place any expression with side effects into an
`assert()` statement.

:-1:
```systemverilog
// Do not assert the output of randomize() calls
// 1.
ret = item.randomize() with {<constraints>};
assert(ret);
// 2.
assert(randomize(<variable>) with {<constraints>});
// 3.
assert(item.randomize() with {<constraints>});
// 4.
assert(function_that_has_side_effects());
```

### Wait and Fork

Always put `wait fork` and `disable fork` constructs inside of an
`isolation fork...join` block to avoid erroneous waiting.
It is recommended to use the `DV_SPINWAIT` macro whenever possible,
[as described here](#macro-usage)


### Void Casts

Do not void cast any function/task calls that have useful return values,
EXCEPT the following:
*   system functions.
*   RAL `predict()` calls.
*   fifo/queue methods.


### Associative Arrays

Do not use wildcard indexed `[*]` associative arrays.
Always specify a particular index type.


### Bind Statements

`bind` statements are the preferred approach for using assertion-based monitors.
For these situations, using `.*` is allowed for making implicit port
declarations when binding a module whose ports are all inputs.


### Simulator-specific Code

Always wrap simulator-specific code inside preprocessor guards.
The macro names `VCS`, `INCA`, `XCELIUM` are defined for the VCS,
Incisive/NCsim/Irun, and Xcelium/xrun simulators respectively,
these must be used to wrap relevant code, as shown below:

```systemverilog
`ifdef VCS
  $stack();
`endif
```


### Forbidden System Tasks and Functions

Do not use the following system functions

*   `$psprintf`, this is not in the SystemVerilog LRM. Use `$sformatf` instead.
*   `$random` and `$dist_*`, these functions are not part of the SystemVerilog
    random stability model and can break simulation reproducibility.
    Use `$urandom` or a randomization macro instead.
*   `$srandom`, this is not part of the SystemVerilog standard.
    Use `process::self().srandom()` instead.

