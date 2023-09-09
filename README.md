gvirq
=====

General purpose IRQ handler for GBA homebrew in assembly.

This small library is heavily inspired from [libugba](https://github.com/AntonioND/libugba)'s IRQ
handler.

The IRQ handler supports nested interrupts, which is challenging to get correct.

It works by installing a general purpose IRQ handler, which will then call specific handlers set by
the user.

It can be configured to look for the specific handlers at different locations in memory, and the
prioritization of handlers is configurable at compile-time.

Assembler
---------

The gvirq project is designed to be used with the [gvasm](https://github.com/velipso/gvasm)
assembler.

It should work with gvasm 2.3.0+.

Demo
----

The repo includes a basic demo, showing off an Hblank IRQ handler.

You can compile it via:

```
gvasm make main.gvasm
```

This will output `main.gba`, which shows a basic scrolling gradient effect.

How to Use
----------

You will need the files in `src/`:

* `src/config.gvasm` -- configuration
* `src/irqInit.gvasm` -- installs the IRQ handler
* `src/irqHandler.gvasm` -- the IRQ handler itself

### Configuration

You need to provide a location in RAM to store the user IRQ handlers. By default, it is stored at
the beginning of IWRAM (`0x03000000`):

```
.struct IRQHandler = 0x03000000
  ...
.end
```

Next, you can configure which handlers are checked, and in which order:

```
  export IRQPriority = {
    // highest priority
    ...
    // lowest priority
  }
```

The default is for Hblank and Vcount to be the highest priority, since those are usually the most
time critical.

If you know that your program will never use a handler, you can comment it out in the `IRQPriority`
list, which will mean the IRQ handler will never check for it.

### IRQ handler

It's recommended that you copy the IRQ handler function `irqHandler` (inside `irqHandler.gvasm`)
into IWRAM at run-time.

This is a little complicated, because you will need to copy the code to IWRAM when your program
starts.

You will probably have other code that needs to be installed into IWRAM too (for example, sound
routines), so this can happen at the same time.

You will see this happen in the demo.

### Initialization

The `irqInit.gvasm` file provides a Thumb function, `irqInit`. This function does _not_ need to be
installed in IWRAM, and can be run from the ROM directly.

You should call this once, at the start of your program, after copying over `irqHandler`. It will
install the general purpose IRQ handler, clear the user handlers, clear `REG_IE` and `REG_IF`, and
enable interrupts via `REG_IME`.

### Custom handlers

After returning from `irqInit`, interrupts will be enabled, but none will actually fire until you
enable them specifically (and optionally install a specific handler).

In order to do that:

1. Disable interrupts by setting `REG_IME` to 0.
2. Enable any I/O registers needed for the interrupt (ex: `REG_DISPSTAT` for Hblank)
3. Enable the appropriate bit in `REG_IE`.
4. (Optional) Save your handler's address to `IRQHandler.<event>` (ex: `IRQHandler.hblank`).
    * Note: If your handler is Thumb, then you should store the address + 1
5. Enable interrupts by setting `REG_IME` to 1.

You can see this in the demo, commented as `Step 5-8` in the code.

Notice that in the demo, Vblank is enabled, but there is no Vblank handler installed. This is fine.
The IRQ handler will correctly handle the interrupt without a user handler.
