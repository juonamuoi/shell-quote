#set -e

git clone https://github.com/Juonamuoi9/getpumppilot.git
cd getpumppilot
git switch -c perf/quick-wins

cat > perf.patch <<'PATCH'
diff --git a/package.json b/package.json
index 0000000..0000000 100644
--- a/package.json
+++ b/package.json
@@ -1,12 +1,14 @@
 {
   "name": "tanstack_start_ts",
   "private": true,
   "sideEffects": false,
   "type": "module",
   "scripts": {
+    "build:core": "vite build",
+    "ci:full": "bun run sync:route-seo && bun run gen:asset-og && bun run gen:social-hashes && bun run gen:sitemap && bun run test:security && bun run test:seo:social && vite build && bun run ping:indexing",
     "dev": "vite dev",
     "build": "bun run sync:route-seo && bun run gen:asset-og && bun run gen:social-hashes && bun run gen:sitemap && bun run test:security && bun run test:seo:social && vite build && bun run ping:indexing",
     "preview": "vite preview",
     "lint": "eslint .",
     "format": "prettier --write .",
     "cap:sync": "cap sync",
diff --git a/tsconfig.json b/tsconfig.json
index 0000000..0000000 100644
--- a/tsconfig.json
+++ b/tsconfig.json
@@ -1,15 +1,17 @@
 {
   "include": ["src/**/*.ts", "src/**/*.tsx", "vite.config.ts", "eslint.config.js"],
   "compilerOptions": {
     "target": "ES2022",
     "jsx": "react-jsx",
     "module": "ESNext",
     "lib": ["ES2022", "DOM", "DOM.Iterable"],
     "types": ["vite/client"],
 
     /* Bundler mode */
     "moduleResolution": "Bundler",
     "allowImportingTsExtensions": true,
     "verbatimModuleSyntax": false,
     "noEmit": true,
+    "incremental": true,
+    "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo",
 
     /* Linting */
     "skipLibCheck": true,
     "strict": true,
     "noUnusedLocals": false,
PATCH

git apply perf.patch

mkdir -p .github/workflows
cat > .github/workflows/perf-ci.yml <<'YAML'
name: perf-ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: Cache bun + node deps
        uses: actions/cache@v4
        with:
          path: |
            ~/.bun
            ~/.cache/bun
            node_modules
          key: ${{ runner.os }}-deps-${{ hashFiles('**/package-lock.json','**/bun.lock') }}
          restore-keys: |
            ${{ runner.os }}-deps-

      - name: Install dependencies
        run: npm install

      - name: Build core
        run: npm run build:core
YAML

npm install
npm run build:core

git add package.json tsconfig.json .github/workflows/perf-ci.yml
git commit -m "perf: add fast build, TS incremental cache, and CI cache"

echo "Local changes committed successfully."
echo "Next:"
echo "git push --set-upstream origin perf/quick-wins"
echo "gh pr create --title \"perf: quick wins\" --body \"Add a fast local build path, incremental TypeScript caching, and CI cache.\" --base main" shell-quote <sup>[![Version Badge][npm-version-svg]][package-url]</sup>

[![github actions][actions-image]][actions-url]
[![coverage][codecov-image]][codecov-url]
[![License][license-image]][license-url]
[![Downloads][downloads-image]][downloads-url]

[![npm badge][npm-badge-png]][package-url]

Parse and quote shell commands.

# example

## quote

```js
var quote = require('shell-quote/quote');
var s = quote([ 'a', 'b c d', '$f', '"g"' ]);
console.log(s);
```

output

```
a 'b c d' \$f '"g"'
```

## parse

```js
var parse = require('shell-quote/parse');
var xs = parse('a "b c" \\$def \'it\'\\\'\'s great\'');
console.dir(xs);
```

output

```
[ 'a', 'b c', '$def', "it's great" ]
```

## parse with ANSI-C quoting

```js
var parse = require('shell-quote/parse');
var xs = parse("echo $'it\\'s a \\n test'");
console.dir(xs);
```

output

```
[ 'echo', "it's a \n test" ]
```

## parse with an environment variable

```js
var parse = require('shell-quote/parse');
var xs = parse('beep --boop="$PWD"', { PWD: '/home/robot' });
console.dir(xs);
```

output

```
[ 'beep', '--boop=/home/robot' ]
```

## parse with custom escape character

```js
var parse = require('shell-quote/parse');
var xs = parse('beep ^--boop="$PWD"', { PWD: '/home/robot' }, { escape: '^' });
console.dir(xs);
```

output

```
[ 'beep', '--boop=/home/robot' ]
```

## parse with unquoted variable splitting

```js
var parse = require('shell-quote/parse');
var xs = parse('a $T', { T: 'c d' }, { splitUnquoted: true });
console.dir(xs);
```

output

```
[ 'a', 'c', 'd' ]
```

## parsing shell operators

```js
var parse = require('shell-quote/parse');
var xs = parse('beep || boop > /byte');
console.dir(xs);
```

output:

```
[ 'beep', { op: '||' }, 'boop', { op: '>' }, '/byte' ]
```

## parsing shell comment

```js
var parse = require('shell-quote/parse');
var xs = parse('beep > boop # > kaboom');
console.dir(xs);
```

output:

```
[ 'beep', { op: '>' }, 'boop', { comment: ' > kaboom' } ]
```

# methods

```js
var quote = require('shell-quote/quote');
var parse = require('shell-quote/parse');
```

## quote(args)

Return a quoted string for the array `args` suitable for using in shell
commands.

Each entry of `args` may be a string, or one of the object shapes that
`parse` emits: `{ op }` (where `op` is one of the control operators
`||`, `&&`, `;;&`, `;;`, `;&`, `|&`, `<(`, `>(`, `<<<`, `<<-`,
`<<`, `>>`, `>&`, `>|`, `&>>`, `&>`, `<&`, `<>`, `&`, `;`, `(`,
`)`, `|`, `<`, `>`),
`{ op: 'glob', pattern }`, or `{ comment }`. Any
other object shape, an unrecognized `op`, a `pattern`/`comment`
containing line terminators, or a string containing line terminators
anywhere after a `{ comment }` throws a `TypeError`.

The output is POSIX shell (`sh`/`bash`) quoting.
It is not valid for Windows `cmd.exe` or PowerShell,
whose rules differ and, for `cmd.exe`,
are not solvable in the general case. On Windows,
do not build a shell command string from this output;
instead pass an argument array to a non-shell API such as
`child_process.execFile` or `spawn`
(or the `cross-spawn` package),
which does no shell parsing and needs no quoting.

Use the returned string verbatim as shell input.
It is already a complete, escaped shell word (or words);
do not wrap it in additional quotes or embed it in `eval '...'`.
Re-quoting the output (for example, placing it inside single quotes)
turns its backslash escapes into literal characters and corrupts the value.

## parse(cmd, env={})

Return an array of arguments from the quoted string `cmd`.

Interpolate embedded bash-style `$VARNAME` and `${VARNAME}` variables with
the `env` object which like bash will replace undefined variables with `""`.

By default an expanded variable is a single token even when unquoted.
Pass `{ splitUnquoted: true }` to split an unquoted expansion into multiple tokens the way a shell performs field splitting,
using the default `IFS` (space, tab, newline).
Pass a string to use its characters as the `IFS` instead
(for example `{ splitUnquoted: ':' }`).
A quoted expansion (`"$VAR"`) is never split.

Bash [ANSI-C quoting](https://www.gnu.org/software/bash/manual/html_node/ANSI_002dC-Quoting.html)
(`$'...'`) is supported: escape sequences (such as `\n`, `\t`, `\\`, `\'`, `\"`,
`\xHH`, `\uHHHH`, `\UHHHHHHHH`, `\nnn`, and `\cx`) are decoded, an unrecognized
escape keeps its backslash, and the contents are never word-split and never
variable-expanded, so `${VAR}` inside `$'...'` stays literal. As in bash,
`$'...'` is not special inside double or single quotes, and a NUL ends the string.
Since the result is a JavaScript string, `\xHH` and `\nnn` produce the character
with that code (U+0000 to U+00FF), not a raw byte.

Only simple `$VARNAME` and `${VARNAME}` interpolation is supported.
Bash parameter expansion beyond a plain variable name is not evaluated:
forms such as array subscripts (`${arr[i]}`), length (`${#arr[@]}`),
and modifiers (`${var:-default}`, `${var/a/b}`)
are treated as an unknown variable and expand to `""`,
while arithmetic (`$((...))`) and command substitution (`$(...)`)
are not interpreted.
Whitespace inside `${...}` throws a `Bad substitution` error.

`env` is usually an object but it can also be a function to perform lookups.
When `env(key)` returns a string, its result will be output just like `env[key]` would.
When `env(key)` returns an object, it will be inserted into the result
array like the operator objects.

When a bash operator is encountered,
the element in the array with be an object with an `"op"` key set to the operator string.
For example:

```
'beep || boop > /byte'
```

parses as:

```
[ 'beep', { op: '||' }, 'boop', { op: '>' }, '/byte' ]
```

The here-document operators (`<<` and `<<-`) are emitted like any other operator,
followed by the delimiter word.
The here-document body is not recognized:
a newline is whitespace to `parse`,
so the lines after the command are parsed as ordinary tokens,
subject to quoting, comments, globs, and variable expansion.

# install

With [npm](http://npmjs.org) do:

```
npm install shell-quote
```

# license

MIT

[package-url]: https://npmjs.org/package/shell-quote
[npm-version-svg]: https://versionbadg.es/ljharb/shell-quote.svg
[deps-svg]: https://david-dm.org/ljharb/shell-quote.svg
[deps-url]: https://david-dm.org/ljharb/shell-quote
[npm-badge-png]: https://nodei.co/npm/shell-quote.png?downloads=true&stars=true
[license-image]: https://img.shields.io/npm/l/shell-quote.svg
[license-url]: LICENSE
[downloads-image]: https://img.shields.io/npm/dm/shell-quote.svg
[downloads-url]: https://npm-stat.com/charts.html?package=shell-quote
[codecov-image]: https://codecov.io/gh/ljharb/shell-quote/branch/main/graphs/badge.svg
[codecov-url]: https://app.codecov.io/gh/ljharb/shell-quote/
[actions-image]: https://img.shields.io/github/check-runs/ljharb/shell-quote/main
[actions-url]: https://github.com/ljharb/shell-quote/actions
