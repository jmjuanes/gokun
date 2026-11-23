# gokun

![npm version](https://badgen.net/npm/v/gokun?labelColor=1d2734&color=21bf81)
![license](https://badgen.net/github/license/jmjuanes/gokun?labelColor=1d2734&color=21bf81)

A minimal, dependency-free task runner. Define your tasks in a plain JavaScript file, connect `src`, `each` and `dest` into pipelines, and run them with the CLI or with plain `node`.

## Installation

Install it locally in your project, so your tasks file and the CLI share the same copy of the library.

```bash
npm install --save-dev gokun
```

Requires Node.js 22 or later.

## Usage

Create a `gokun.config.js` file in your project:

```js
import { src, dest, each, pipeline, task, series, parallel } from "gokun";

task("pages", () => {
    return pipeline(
        src("src/**/*.html"),
        each((file) => {
            file.content = file.content.toString().toUpperCase();
        }),
        dest("dist"),
    );
});

task("assets", () => pipeline(src("src/assets/**/*"), dest("dist/assets")));

task("build", parallel("pages", "assets"));
task("default", series("build"));
```

Then run it:

```bash
npx gokun              # runs the "default" task
npx gokun build        # runs "build"
npx gokun pages assets # runs "pages", then "assets"
npx gokun --list       # lists the available tasks
```

### CLI options

| Option | Description |
| --- | --- |
| `-f`, `--file <path>` | Tasks file to load. Default: `gokun.config.js` |
| `-l`, `--list` | List the available tasks |
| `-h`, `--help` | Show the help |

### Running without the CLI

You don't need the CLI. Put your tasks in any file, call `run()` at the end, and execute it with Node.js:

```js
// tasks.js
import { src, dest, pipeline, task, series, run } from "gokun";

task("build", () => pipeline(src("src/**/*"), dest("dist")));
task("default", series("build"));

await run();
```

```bash
node tasks.js              # runs the "default" task
node tasks.js build        # runs "build"
node tasks.js pages assets # runs "pages", then "assets"
```

It plays well with `package.json` scripts and with Node.js built-in tools:

```json
{
    "scripts": {
        "build": "node tasks.js build"
    }
}
```

## API

### `task(name, fn)`

Registers a task. `fn` can be sync or async. If it returns a promise, or a stream, the task finishes when that promise or stream does. Always `return` it, otherwise `gokun` cannot wait for it.

### `run([args])`

Runs the tasks named in `args` and returns a promise that resolves when they finish. Use it as the last line of your tasks file when you run it with `node` instead of the CLI.

- `args`: array of task names. Default: `process.argv.slice(2)`

If `args` has no task names, the `default` task runs. If a task is missing or fails, the error is printed and `process.exitCode` is set to `1`, so your scripts and CI fail as expected.

```js
await run();                       // uses the command line arguments
await run(["build", "assets"]);    // runs these tasks, ignoring the command line
```

### `pipeline(...streams)`

Connects the streams, and returns a promise that resolves when the last one finishes. If any stream fails, the others are closed and the promise rejects. It is the `pipeline` function of `node:stream/promises`, re-exported for convenience.

### `src(patterns, [options])`

Creates a stream with the files that match one or more glob patterns. The paths are listed first, but the content of each file is read only when the stream needs it.

- `patterns`: a glob string or an array of them. Files are streamed in pattern order.
- `options.cwd`: directory used to resolve the patterns. Default: `.`

Each file in the stream is an object with these properties:

| Property | Description |
| --- | --- |
| `path` | Absolute path of the file in the source tree |
| `base` | Directory the pattern is relative to |
| `relative` | Path relative to `base`. Change it to rename or move the file |
| `content` | File content, as a `Buffer` or a `string` |

### `each(fn)`

Creates a stream that runs `fn` for every file, one at a time and in order. `fn` receives a file and can:

- return nothing, to keep the (possibly mutated) file
- return a file, to replace it
- return `null`, to drop it

`fn` can be async.

```js
each((file) => file.relative.endsWith(".test.js") ? null : file);
```

### `all(fn)`

Creates a stream that waits for all the files and gives them to `fn` as an array. `fn` returns the new array, or nothing to keep the files. Because every file is in memory at once, use it only for what really needs to see them all, like concatenating.

```js
all((files) => files.slice(0, 10));
```

### `dest(directory)`

Creates a stream that writes each file inside `directory`, keeping its relative path. It is the end of a pipeline.

### `series(...tasks)` and `parallel(...tasks)`

Create a task that runs the given tasks one after another, or at the same time. Each item can be a task function or the name of a registered task.

## License

Released under the [MIT License](./LICENSE).
