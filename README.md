# julia-repl: run an inferior Julia REPL in Emacs

[![MELPA](https://melpa.org/packages/julia-repl-badge.svg)](https://melpa.org/#/julia-repl)
[![build](https://github.com/tpapp/julia-repl/workflows/check/badge.svg)](https://github.com/tpapp/julia-repl/actions?query=workflow%3Acheck)

This is a minor mode for interacting with a Julia REPL running inside Emacs. The `julia` process is started in an ANSI terminal (`term`), which allows text formatting and colors, and interaction with the help system and the debugger.

It is recommended that you use this minor mode with [julia-mode](https://github.com/JuliaEditorSupport/julia-emacs).

<img src="./screenshot.png" width="70%" alt="screenshot">

## Installation and loading

**Please make sure you have at least Emacs 25**. The `term` code changed a bit since Emacs 24, and the package does not support that and earlier versions. For example, Ubuntu has had Emacs 25 since 18.04LTS.

Place this in your **Emacs initialization files** (eg `.emacs`):
```emacs-lisp
(add-to-list 'load-path path-to-julia-repl)
(require 'julia-repl)
(add-hook 'julia-mode-hook 'julia-repl-mode) ;; always use minor mode
```
If you want to use a Julia executable other than `julia` in your path, see [below](#julia-executables).

If you are experiencing problems with [Unicode characters](https://docs.julialang.org/en/v1/manual/unicode-input/) in the Julia REPL, try setting the relevant coding/language environment, eg
```emacs-lisp
(set-language-environment "UTF-8")
```

## Usage

`M-x julia-repl`, or `C-c C-z` from a buffer in which the `julia-repl` minor mode is active starts a new inferior Julia process. The keys below can be used to interact with this process.

| key           | action                                                      |
|---------------|-------------------------------------------------------------|
| `C-c C-a`     | activate if there is a `Project.toml` in parent directories |
| `C-u C-c C-a` | activate home project                                       |
| `C-c C-b`     | send whole buffer to REPL (using include)                   |
| `C-u C-c C-b` | send whole buffer to REPL (directly)                        |
| `C-c C-c`     | send region (when applicable) or line to REPL               |
| `C-c C-d`     | invoke `@doc` on symbol                                     |
| `C-c C-e`     | invoke `@edit` on region (when applicable) or line          |
| `C-c C-l`     | list methods of a function                                  |
| `C-c C-m`     | expand macro                                                |
| `C-c C-p`     | change directory to that of the buffer                      |
| `C-c C-s`     | prompt for buffer name suffix                               |
| `C-c C-t`     | send whole buffer to REPL (using Revise.includet)           |
| `C-c C-v`     | prompt for Julia executable                                 |
| `C-c C-z`     | raise the REPL or create a new one (or switch back from REPL – only in `vterm` backend) |
| `C-RET`       | send line to REPL (without bracketed paste)                 |

All actions that send something to the REPL terminate with a **newline**, triggering evaluation. If you want to avoid sending a newline (eg maybe because you want to edit an expression), use prefix arguments (`C--` or `C-u`, currently both have the same effect). This of course does not apply to `C-c C-b`.

All commands send code using [bracketed paste](https://cirw.in/blog/bracketed-paste). When Julia is waiting for input, control characters like `^[[200~` may show up in your buffer, this is innocuous. If you input takes a long time to evaluate, you can step through it line-by-line with `C-RET`.

## Environment variables

You can set environment variables directly from your `init.el` in Emacs, eg

```emacs-lisp
(setenv "JULIA_NUM_THREADS" "4")
```

## Buffer-local inferior REPL and Julia executable

The minor mode allows the user to select a particular Julia executable and optionally a different inferior buffer for each source code buffer. This allows running two versions (eg stable and master) of Julia simultaneously, and/or running multiple inferior REPLs of the same Julia version. A typical use case is trying out something quickly, without changing the state of the current process.

### Julia executables

Set `julia-repl-executable-records` to a list of keys and executables. For example,
```elisp
(setq julia-repl-executable-records
      '((default "julia")                  ; in the executable path
        (master "~/src/julia-git/julia"))) ; compiled from the repository
```
provides two executables. The first entry is always the default (it can have any other key).

Use `C-c C-v` to select one of these (`julia-repl-prompt-executable`). You can also set the value of `julia-repl-executable-key` directly to a key in the `julia-repl-executable-records`, eg using [file variables](https://www.gnu.org/software/emacs/manual/html_node/emacs/Specifying-File-Variables.html), but make sure you select a correct value.

The name of the inferior buffer will reflect your choice: the default is `*julia*` (indicator omitted), while the `master` executable would map to `*julia-master*`, and so on.

### Executable suffix

You can also set a *suffix* for the inferior buffer, if you want multiple ones in parallel. This can be a number, which will show up as `<number>`, or a symbol, which appears as `-symbol`.

It is recommended that you use `C-c C-s` (`julia-repl-prompt-inferior-buffer-name-suffix`), which prompts for a string by default. Prefix arguments modify it like this:

- numerical prefixes select that integer: eg `C-3 C-c C-s` set the suffix to `3`.

- the negative prefix picks the next unused integer: eg `C- C-c C-s` sets the suffix to `4` if `1`, `2`, `3` are in use.

### Switches

Switches to the `julia` process can be provided in the global variable `julia-repl-switches`, for example

```elisp
(setq julia-repl-switches "-p 4")
```
The function `julia-repl-prompt-switches` will prompt for new switches, you  can bind it to a key.

### File local variables

If you are using the same settings for a specific file, consider using [file variables](https://www.gnu.org/software/emacs/manual/html_node/emacs/Specifying-File-Variables.html). For example, if you use `add-file-local-variable` to create a block at the end of the Julia source file similar to

```julia
# Local Variables:
# julia-repl-executable-key: master
# julia-repl-inferior-buffer-name-suffix: tests
# julia-repl-switches: "-p 4"
# End:
```

then the next time you open a REPL, it will have the name `*julia-master-tests*`, and 4 worker processes.

## Terminal backends

`julia-repl` can use the terminal in different ways. The default is `ansi-term`, which is included in Emacs. There is experimental support for [`vterm` via `emacs-libvterm`](https://github.com/akermu/emacs-libvterm).

### Some hints on interacting with `term`

Note some keybindings for `term`:

1. `C-x C-j` switches to *line mode*, where you can kill/yank, move around the buffer, use standard Emacs keybindings,
2. `C-c C-k` switches back to *char mode*,
3. for scrolling, use `S-<prior>` and `S-<next>`.

See the help of `term` for more.

### Using `vterm` (experimental)

Support for the vterm backend is WIP. In the long run it is hoped that it will replace `ansi-term` as the default backend for this package, fixing many outstanding issues.

1. Install [`emacs-libvterm`](https://github.com/akermu/emacs-libvterm) and make sure you have a working installation (eg `M-x vterm`) should start a terminal

2. Evaluate `(julia-repl-set-terminal-backend 'vterm)` in your config file *after* you load `julia-repl`, but *before* you use it (and of course `vterm` should be loaded at some point). Switching terminal backends with already running Julia processes is not supported.

3. You may want to `(setq vterm-kill-buffer-on-exit nil)` to prevent the buffers associated with terminated Julia processes being killed automatically. This allows you to retain output and see error messages if the process does not start.

## Using the @edit macro

The `@edit` macro can be called with `C-c C-e` when the `julia-repl-mode` minor mode is enabled. The behavior depends on the value of the `JULIA_EDITOR` envoronment variable in the Julia session. The command `julia-repl-set-julia-editor` is provided to conveniently control this from emacs.

To use "emacsclient" as a default in each Julia REPL you open in emacs, add the following code to your `~/.julia/config/startup.jl`:

```julia
if haskey(ENV, "INSIDE_EMACS")
    ENV["JULIA_EDITOR"] = "emacsclient"
end
```

If you cannot edit your `startup.jl`, you can configure the editor in each repl after starting it with:

```elisp
(add-hook 'julia-repl-hook #'julia-repl-use-emacsclient)
```

## More colors

Julia uses more colors than the ones supported by `term` by default. To get the full range of colors, use [eterm-256color](https://github.com/dieggsy/eterm-256color), available from MELPA.

## Note for Windows users

`ansi-term` is **not supported** on some (most?) Windows versions of Emacs. It has been confirmed to work with Emacs running in Cygwin. You can test whether your version works with `M-x ansi-term`; if this errors then this package will **not** work for you.

Cygwin may require some rewriting of paths for `include` to work. After loading this package, customize the rewrite rules as
```emacs-lisp
(setq julia-repl-path-rewrite-rules julia-repl-cygwin-path-rewrite-rules)
```
as a reasonable default, or write custom rules for `julia-repl-path-rewrites`.

## Limitations

See the [issues](https://github.com/tpapp/julia-repl/issues).

## Comparison to ESS

A well-known alternative is [ESS](https://ess.r-project.org/), which also supports Julia. ESS is however based on `comint`, which does not allow a fully functioning terminal and therefore does not support the Julia REPL modes (shell, Pkg, debuggers...) and features. `julia-repl` is instead based on `term`, and so gets all the features from Julia's native REPL for free.
