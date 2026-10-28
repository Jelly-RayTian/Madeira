# UTF-8 launch working directory

The iOS bridge supplies `MADEIRA_INITIAL_CWD` using UTF-8. Wine process
parameters require an NT UTF-16 path. Widening each byte produces an invalid
path for non-ASCII folders: a launch under `C:\TaskbarList-iPad测试\` was
reported as `C:\TaskbarList-iPad\ffe6\ffb5\ff8b\ffe8\ffaf\ff95\` and Wine
fell back to its Windows directory. Absolute EXE loading can still succeed,
while relative config, asset or save lookups use the wrong directory.

The override now uses the existing Wine UTF-8 decoder, including surrogate
pairs, and appends the trailing separator using the decoded UTF-16 length.
ASCII paths keep the same result. An absent/empty override, invalid UTF-8 or
allocation failure leaves the existing Unix/PWD/default-directory path in charge.
No code-page, path normalization, profile or game-specific behavior is changed.

Host regression, requiring the pinned Wine source:

```sh
python3 tests/host/check-initial-cwd-utf8.py
```

It compiles the actual override and Wine's actual decoder under ASan/UBSan,
compares 15 multilingual paths against independently generated UTF-16 values,
and checks missing/empty/OOM/invalid-input fallbacks. It does not exercise iOS
filesystem mappings or establish game compatibility.

For a matching rebuilt Debug package on iPad, compile:

```sh
TC=toolchains/llvm-mingw-20260421-ucrt-macos-universal/bin
"$TC/x86_64-w64-mingw32-clang" -O2 -Wall -Wextra -Werror \
  tests/x64/initial-cwd.c -o initial-cwd-x64.exe
```

Place the EXE in a folder such as `C:\Games\测试🎮\`, leave any explicit per-game
working-folder override empty, enable JIT and launch it directly as x64.
It creates a uniquely named probe beside the EXE, reads it by a relative name,
compares its contents and deletes it. Acceptance is `[initial-cwd] PASS relative
resource lookup`, guest exit0 and no working-directory fallback in the session
log. An intentionally different working-folder setting does not satisfy this
fixture's assumption. Repeat with an ASCII folder as a control.

## Startup parameter storage for long directories

Decoding the override is separate from reserving space for it. Both initial
process startup and reconstruction of a child's server-supplied parameters
previously reserved only `MAX_PATH` WCHARs for the current directory. Copying
260 or more UTF-16 code units (before the NUL) crossed that slot, and the DLL
path/image/command-line strings packed immediately after it overwrote the
copied directory. A sufficiently long input also exceeded the total allocation
in an exact-size host allocator.

The iOS adapter now reserves the larger of the existing `MAX_PATH` slot and
the actual current-directory bytes including NUL, uses that same capacity for
`MaximumLength`, and starts the following strings after that capacity. The
initial route measures the DOS path after removing the NT prefix; the child
route uses the server's byte length. Short directories keep their old capacity.

```sh
python3 tests/host/check-startup-cwd-layout.py
```

This compiles the production allocation expressions, packing code and string
helpers under ASan/UBSan. Twenty-four cases cover initial and child startup,
259/260/261-unit boundaries, long UTF-16 paths with surrogate pairs, extended
UNC spellings, an empty child directory, and a representable 32,766-unit string.
It checks that the complete directory and the following strings survive packing,
that capacity includes NUL, and that child input advancement stays correct.
The harness replaces NT allocation with exact-size host allocation; it does not
run Wine, map a filesystem or launch a game.

This is a startup serialization fix, not a claim of complete Windows long-path
support. Other Wine path APIs and `RtlCreateProcessParametersEx` have independent
limits; paths outside `UNICODE_STRING`'s representable range are not covered.
Device launch and gameplay need validation with a matching rebuilt package.
