# libopengl

OpenGL 1.1 is a library. You link `opengl32` or `libGL`, you call `glClear`, you move on. Everything after that is not a library. The 1.2+ symbols are not something you link. They are addresses a getproc (`glfwGetProcAddress`, `wglGetProcAddress`, `glXGetProcAddress`) hands you for the current context.

C's answer is GLAD. This is Echo's: a generator, written in Echo, that reads the Khronos `gl.xml` checked in next to it and emits `src/gl.eco`.

There is nothing to link. No `#[link:]`, no Darwin `framework "OpenGL"`. A forgotten `gl::load` is a `die`, not a jump to `0x0`. I think that is the correct default. Linking the 1.1 system library on Darwin would only hide the bug until you call something 1.1 never exported.

This is also not a window. You still need GLFW, SDL, or whatever you use to create a context and make it current. This module only talks to that context once it exists.

## The simple case

```bash
epm add echolang/libopengl --git https://github.com/echolang/libopengl --range ^0.1
```

That is all for the GL table. For a window and a context, add libglfw too:

```bash
epm add echolang/libglfw --git https://github.com/echolang/libglfw --range ^0.1
```

The `gl` namespace is there as soon as the module loads. After `glfw::makeContextCurrent`, you load the table, then you call it:

```echo
glfw::makeContextCurrent($window);
gl::load(&glfw::getProcAddress);
gl::ClearColor(0.1, 0.1, 0.1, 1.0);
gl::Clear(gl::COLOR_BUFFER_BIT);
```

That compiles. It dies at runtime if you call into it before `load`, or if you `load` with no current context.

`&name` of an `extern` function is legal, so GLFW is just that. SDL, raw WGL, whatever: if the function is `extern function<ptr<uint8>(ptr<const uint8>)>`, it is a getproc.

## Loading

`gl::load` takes a getproc and returns how many entry points resolved.

```echo
usize $n = gl::load(&glfw::getProcAddress);
```

A null from getproc leaves the die-stub that the table was constructed with. That is the whole of how a 4.1 macOS context is allowed to lack a 4.6 function: you hear about it when you call it, not when you load.

`load` has to run after the context is current. A getproc with no current context returns null for everything, and you are left with a table of dies.

The table is a static, initialized once. A second `load` overwrites the slots that resolve this time. A slot that comes back null is left alone, so a second load cannot unbind.

Here is the catch: `load` does not fail if half the table is missing. It counts. If you asked for 4.6 and the context is 4.1, that is your problem to notice. `versionMajor` exists for that.

I understand some people want a result per name. I don't. Calling a function the context does not have is a programmer error. The die is the check you forgot to write.

```echo
gl::Clear(gl::COLOR_BUFFER_BIT);
// die: gl: entry point called before gl::load, or unavailable in this context
```

That is a test, not a comment: `#[tests: expects death]`.

## Calling

Names are stripped the way gl-rs does it. `glDrawArrays` is `gl::DrawArrays`. `GL_TRIANGLES` is `gl::TRIANGLES`.

```echo
gl::DrawArrays(gl::TRIANGLES, 0, 3);
uint32 $err = gl::GetError();
```

There is no `GL_` prefix and there is no `GLenum` type. An enum is a `uint32` with a name. Khronos already thought of it as an int; we just stopped pretending.

Constants are `uint32`, except the ones the registry tagged `ull`. `TIMEOUT_IGNORED` is `uint64` because of that.

### Types

The C types collapse to Echo types. The ones you actually write:

- `GLenum` / `GLbitfield` / `GLuint` become `uint32`
- `GLint` / `GLsizei` become `int32`
- `GLboolean` becomes `uint8`, not `bool`. `gl::TRUE` is `1`. `gl::FALSE` is `0`.
- `GLfloat` becomes `float32`
- `GLsizeiptr` / `GLintptr` become `isize`
- `void *` and `GLsync` become `ptr<uint8>`
- `const GLchar *` becomes `ptr<const uint8>`
- `const GLchar *const*` becomes `ptr<ptr<const uint8>>`

```echo
gl::DepthMask(gl::TRUE);
```

`DepthMask` takes a `uint8`. Passing a `bool` will not compile.

```echo
ptr<const uint8> $src = $glsl->cstr();
gl::ShaderSource($shader, 1, &$src, null);
```

The third argument of `ShaderSource` is a pointer to pointers. You take the address of a `cstr`. You do not pass the string itself. `null` for `$length` means the strings are 0-terminated, which is the OpenGL rule, not ours.

Interior consts in the C declarator are dropped (ABI-identical). Array parameters decay to pointers, the way C already does at a function boundary.

Wrappers are `#[inline]`. They do one thing: call the function pointer in the table.

## What the context actually has

The committed file is 4.6 core plus `GL_KHR_debug`. Your context might not be. `supports`, `versionMajor`, and `versionMinor` are thin over `GetStringi` / `GetIntegerv` after load:

```echo
echo gl::versionMajor();          // 4, on a 4.x context
echo gl::supports('GL_KHR_debug');
```

They go through the same table as everything else, so they only mean something after `load`. An unloaded `versionMajor` dies the same way `Clear` does.

`supports` takes the Khronos name, with the `GL_` prefix, because that is what `glGetStringi(GL_EXTENSIONS, i)` returns.

## Generating

The committed `src/gl.eco` is core profile **4.6**, plus the extensions passed to the generator. The checked-in set starts with `GL_KHR_debug`. It is 4.6 full, not a 1.0 subset: every command in that set is a field, a load block, and an `#[inline]` wrapper.

Core is the point. Immediate-mode `glBegin` is gone. Good. Compatibility is a flag you can pass. I would not.

The generator is a target of this module, written in Echo:

```bash
echoc run --target generate -- --version 4.6 --profile core --ext GL_KHR_debug
```

The `--` is echoc's. Everything after it is argv for generate. If you drop it, `--version` is echoc's flag, not ours.

`--check` regenerates in memory, byte-compares against `src/gl.eco`, writes nothing, and exits 1 on drift:

```bash
echoc run --target generate -- --version 4.6 --profile core --ext GL_KHR_debug --check
```

That is the standing guard, not a hook. Generated source is a cache input, and Echo has no build-script step. I want the generated file committed and the check to fail the run, not a silent rewrite during compile.

Defaults are `--api gl --version 4.6 --profile core`. `--ext` is repeatable and empty unless you pass it. If you regenerate without `--ext GL_KHR_debug`, you drop debug. The banner at the top of `src/gl.eco` records the SHA and the invocation so you can see what you last asked for. It does not record a timestamp, so a byte compare stays stable.

If `src/gl.eco` is stale enough to refuse to compile, delete it and rerun. `src/loader.eco` is hand-written, names no generated symbol, and keeps the `src/*.eco` glob non-empty.

```bash
rm src/gl.eco
echoc run --target generate -- --version 4.6 --profile core --ext GL_KHR_debug
```

Flags:

```
--xml PATH        Khronos registry (default gl.xml)
--out PATH        generated module (default src/gl.eco)
--api NAME        api to fold (default gl)
--version X.Y     maximum version (default 4.6)
--profile NAME    profile (default core)
--ext NAME        extension to include, repeatable
--check           regenerate in memory, compare, write nothing
```

`--api` is `gl`. GLES is a different fold, and mixing them in one emit is not what this module ships.

The fold walks features in document order, keeps `api == gl` whose version is at most the one you asked for, applies requires and removes for the profile, then each `--ext`. Sorted order is the one canonical order for everything emit writes.

## The registry

`gl.xml` is pinned to Khronos commit `0eab18f308d29bb54aa4667013ae505d73a72194`. Regeneration is deliberate. To bump it:

```bash
curl -L -o gl.xml https://raw.githubusercontent.com/KhronosGroup/OpenGL-Registry/<SHA>/xml/gl.xml
```

The SHA lives in two places: this README and `REGISTRY_SHA` in `gen/main.eco`. Put the new one in both, then regenerate. The banner of `src/gl.eco` has to match.

## Tests

GPU-free. `echoc test` from this directory compiles the generated file, the loader, and the generator in one build.

```bash
echoc test
```

An unloaded call is a `die`. Each test is a fresh process, so `load` in one test cannot leak into the next.
