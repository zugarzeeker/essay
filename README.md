
# essay
[![NPM version][npm-svg]][npm]
[![Build status][travis-svg]][travis]
[![Code coverage][codecov-svg]][codecov]

[travis]: https://travis-ci.org/dtinth/essay
[travis-svg]: https://img.shields.io/travis/dtinth/essay.svg?style=flat
[npm]: https://www.npmjs.com/package/essay
[npm-svg]: https://img.shields.io/npm/v/essay.svg?style=flat
[codecov]: https://codecov.io/gh/dtinth/essay/src/master/README.md
[codecov-svg]: https://img.shields.io/codecov/c/github/dtinth/essay.svg

__Generate your JavaScript library out of an essay!__

- __Write code in your README.md file__ in fenced code blocks, with a comment indicating the file’s name.

- __Write code using ES2015 syntax__. `essay` will use [Babel](https://babeljs.io/) to transpile them to ES5.

- __Test your code__ using [Mocha](https://mochajs.org/) and [power-assert](https://github.com/power-assert-js/power-assert).

- __Measures your code coverage__. `essay` generates [code coverage report for your README.md file][codecov] using [Istanbul](https://github.com/gotwarlost/istanbul) and [babel-plugin-\_\_coverage\_\_](https://github.com/dtinth/babel-plugin-__coverage__).

- __Examples__ of JavaScript libraries/articles written using `essay`:

  | Project | Description |
  | ------- | ----------- |
  | [circumstance](https://github.com/dtinth/circumstance) | BDD for your pure state-updating functions (e.g. Redux reducers) |
  | [positioning-strategy](https://github.com/taskworld/positioning-strategy) | A library that implements an algorithm to calculate where to position an element relative to another element |
  | [code-to-essay](https://github.com/zugarzeeker/code-to-essay) | Turns your code into an essay-formatted Markdown file. |
  | [timetable-calculator](https://github.com/zugarzeeker/timetable-calculator) | An algorithm for computing the layout for a timetable, handling the case where items are overlapped. |
  | [impure](https://github.com/taskworld/impure) | A simple wrapper object for non-deterministic code (like IO monads in Haskell) |

  - Using __essay__ to write a library/article? Feel free to add your link here: please submit a PR!


## synopsis

For example, you could write your library in a fenced code block in your README.md file like this:

```js
// examples/add.js
export default (a, b) => a + b
```

And also write a test for it:

```js
// examples/add.test.js
import add from './add'

// describe block called automatically for us!
it('should add two numbers', () => {
  assert(add(1, 2) === 3)
})
```


### getting started

1. Make sure you already have your README.md file in place and already initialized your npm package (e.g. using `npm init`).

2. Install `essay` as your dev dependency.

    ```
    npm install --save-dev essay
    ```

3. Ignore these folders in `.gitignore`:

    ```
    src
    lib
    lib-cov
    ```

4. Add `"files"` array to your `package.json`:

    ```json
    "files": [
      "lib",
      "src"
    ]
    ```

5. Add the `"scripts"` to your `package.json`:

    ```json
    "scripts": {
      "prepublish": "essay build",
      "test": "essay test"
    },
    ```

6. Set your main to the file you want to use, prefixed with `lib/`:

    ```json
    "main": "lib/index.js",
    ```


### building

When you run:

```
npm run prepublish # -> essay build
```

These code blocks will be extracted into its own file:

```
src
└── examples
    ├── add.js
    └── add.test.js
lib
└── examples
    ├── add.js
    └── add.test.js
```

The `src` folder contains the code as written in README.md, and the `lib` folder contains the transpiled code.


### testing

When you run:

```
npm test # -> essay test
```

All the files ending with `.test.js` will be run using Mocha framework. `power-assert` is included by default (but you can use any assertion library you want).

```
  examples/add.test.js
    ✓ should add two numbers
```

Additionally, test coverage report for your README.md file will be generated.



## development

You need to use `npm install --force`, because I use `essay` to write `essay`,
but `npm` doesn’t want to install a package as a dependency of itself
(even though it is an older version).



## commands

### `essay build`

```js
// cli/buildCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import dumpSourceCodeBlocks from '../dumpSourceCodeBlocks'
import transpileCodeBlocks from '../transpileCodeBlocks'
import getBabelConfig from '../getBabelConfig'

export const command = 'build'
export const description = 'Builds the README.md file into lib folder.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const babelConfig = getBabelConfig()
  const targetDirectory = 'lib'
  const codeBlocks = await obtainCodeBlocks()
  await dumpSourceCodeBlocks(codeBlocks)
  await transpileCodeBlocks({ targetDirectory, babelConfig })(codeBlocks)
}
```


### `essay test`

```js
// cli/testCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import dumpSourceCodeBlocks from '../dumpSourceCodeBlocks'
import transpileCodeBlocks from '../transpileCodeBlocks'
import getTestingBabelConfig from '../getTestingBabelConfig'
import runUnitTests from '../runUnitTests'

export const command = 'test'
export const description = 'Runs the test.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const babelConfig = getTestingBabelConfig()
  const targetDirectory = 'lib-cov'
  const codeBlocks = await obtainCodeBlocks()
  await dumpSourceCodeBlocks(codeBlocks)
  await transpileCodeBlocks({ targetDirectory, babelConfig })(codeBlocks)
  await runUnitTests(codeBlocks)
}
```

### `essay lint`

```js
// cli/lintCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import runLinter from '../runLinter'

export const command = 'lint'
export const description = 'Runs the linter.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const codeBlocks = await obtainCodeBlocks()
  await runLinter(codeBlocks, argv)
}
```


## obtaining code blocks

This IO function reads your README.md and extracts the fenced code blocks.

```js
// obtainCodeBlocks.js
import extractCodeBlocks from './extractCodeBlocks'
import fs from 'fs'

export async function obtainCodeBlocks () {
  const readme = fs.readFileSync('README.md', 'utf8')
  const codeBlocks = extractCodeBlocks(readme)
  return codeBlocks
}

export default obtainCodeBlocks
```


### code block extraction

This function takes a string (representing your README.md) and extracts the fenced code block. Returns an object of code block entries, which contains the contents and line number in README.md.

See the test (below) for more details:

```js
// extractCodeBlocks.js
export function extractCodeBlocks (data) {
  const codeBlocks = { }
  const regexp = /(`[`]`js\s+\/\/\s*(\S+).*\n)([\s\S]+?)`[`]`/g
  data.replace(regexp, (all, before, filename, contents, index) => {
    if (codeBlocks[filename]) throw new Error(filename + ' already exists!')

    // XXX: Not the most efficient way to find the line number.
    const line = data.substr(0, index + before.length).split('\n').length

    codeBlocks[filename] = { contents, line }
  })
  return codeBlocks
}

export default extractCodeBlocks
```

Here’s the test for this function:

```js
// extractCodeBlocks.test.js
import extractCodeBlocks from './extractCodeBlocks'

const END = '`' + '`' + '`'
const BEGIN = END + 'js'

const example = [
  'Hello world!',
  '============',
  '',
  BEGIN,
  '// file1.js',
  'console.log("hello,")',
  END,
  '',
  '- It should work in lists too!',
  '',
  '  ' + BEGIN,
  '  // file2.js',
  '  console.log("world!")',
  '  ' + END,
  '',
  'That’s it!'
].join('\n')

const blocks = extractCodeBlocks(example)

it('should extract code blocks into object', () => {
  assert.deepEqual(Object.keys(blocks).sort(), [ 'file1.js', 'file2.js' ])
})

it('should contain the code block’s contents', () => {
  assert(blocks['file1.js'].contents.trim() === 'console.log("hello,")')
  assert(blocks['file2.js'].contents.trim() === 'console.log("world!")')
})

it('should contain line numbers', () => {
  assert(blocks['file1.js'].line === 6)
  assert(blocks['file2.js'].line === 13)
})
```


## dumping code blocks to source files

```js
// dumpSourceCodeBlocks.js
import forEachCodeBlock from './forEachCodeBlock'
import saveToFile from './saveToFile'
import path from 'path'

export const dumpSourceCodeBlocks = forEachCodeBlock(async ({ contents }, filename) => {
  const targetFilePath = path.join('src', filename)
  await saveToFile(targetFilePath, contents)
})

export default dumpSourceCodeBlocks
```


## transpilation using Babel

```js
// transpileCodeBlocks.js
import forEachCodeBlock from './forEachCodeBlock'
import transpileCodeBlock from './transpileCodeBlock'

export function transpileCodeBlocks (options) {
  return forEachCodeBlock(transpileCodeBlock(options))
}

export default transpileCodeBlocks
```


### transpiling an individual code block

To speed up transpilation,
we’ll skip the transpilation process if the source file has not been modified
since the corresponding transpiled file has been generated (similar to make).

```js
// transpileCodeBlock.js
import path from 'path'
import fs from 'fs'
import { transformFileSync } from 'babel-core'

import saveToFile from './saveToFile'

export function transpileCodeBlock ({ babelConfig, targetDirectory } = { }) {
  return async function (codeBlock, filename) {
    const sourceFilePath = path.join('src', filename)
    const targetFilePath = path.join(targetDirectory, filename)
    if (await isAlreadyUpToDate(sourceFilePath, targetFilePath)) return
    const { code } = transformFileSync(sourceFilePath, babelConfig)
    await saveToFile(targetFilePath, code)
  }
}

async function isAlreadyUpToDate (sourceFilePath, targetFilePath) {
  if (!fs.existsSync(targetFilePath)) return false
  const sourceStats = fs.statSync(sourceFilePath)
  const targetStats = fs.statSync(targetFilePath)
  return targetStats.mtime > sourceStats.mtime
}

export default transpileCodeBlock
```


## babel configuration

```js
// getBabelConfig.js
export function getBabelConfig () {
  return {
    presets: [
      require('babel-preset-es2015'),
      require('babel-preset-stage-2'),
    ],
    plugins: [
      require('babel-plugin-transform-runtime')
    ]
  }
}

export default getBabelConfig
```

### additional options for testing

```js
// getTestingBabelConfig.js
import getBabelConfig from './getBabelConfig'

export function getTestingBabelConfig () {
  const babelConfig = getBabelConfig()
  return {
    ...babelConfig,
    plugins: [
      require('babel-plugin-__coverage__'),
      require('babel-plugin-espower'),
      ...babelConfig.plugins
    ]
  }
}

export default getTestingBabelConfig
```


## running unit tests

It’s quite hackish right now, but it works.

```js
// runUnitTests.js
import fs from 'fs'
import saveToFile from './saveToFile'
import mapSourceCoverage from './mapSourceCoverage'

export async function runUnitTests (codeBlocks) {
  // Generate an entry file for mocha to use.
  const testEntryFilename = './lib-cov/_test-entry.js'
  const entry = generateEntryFile(codeBlocks)
  await saveToFile(testEntryFilename, entry)

  // Initialize mocha with the entry file.
  const Mocha = require('mocha')
  const mocha = new Mocha({ ui: 'bdd' })
  mocha.addFile(testEntryFilename)

  // Now go!!
  prepareTestEnvironment()
  await runMocha(mocha)
  await saveCoverageData(codeBlocks)
}

function runMocha (mocha) {
  return new Promise((resolve, reject) => {
    mocha.run(function (failures) {
      if (failures) {
        reject(new Error('There are ' + failures + ' test failure(s).'))
      } else {
        resolve()
      }
    })
  })
}

function generateEntryFile (codeBlocks) {
  const entry = [ '"use strict";' ]
  for (const filename of Object.keys(codeBlocks)) {
    if (filename.match(/\.test\.js$/)) {
      entry.push('describe(' + JSON.stringify(filename) + ', function () {')
      entry.push('  require(' + JSON.stringify('./' + filename) + ')')
      entry.push('})')
    }
  }
  return entry.join('\n')
}

function prepareTestEnvironment () {
  global.assert = require('power-assert')
}

async function saveCoverageData (codeBlocks) {
  const coverage = global['__coverage__']
  if (!coverage) return
  const istanbul = require('istanbul')
  const reporter = new istanbul.Reporter()
  const collector = new istanbul.Collector()
  const synchronously = true
  collector.add(mapSourceCoverage(coverage, {
    codeBlocks,
    sourceFilePath: fs.realpathSync('README.md'),
    targetDirectory: fs.realpathSync('src')
  }))
  reporter.add('lcov')
  reporter.add('text')
  reporter.write(collector, synchronously, () => { })
}

export default runUnitTests
```


## running linter
```js
// runLinter.js
import fs from 'fs'
import saveToFile from './saveToFile'
import standard from 'standard'
import forEachCodeBlock from './forEachCodeBlock'

export const paddingText = (width, string, padding = ' ') => {
  return (width <= string.length) ? string : paddingText(width, string + padding, padding)
}

export const countLinterErrors = (results) => results[0].messages.length

export const removeEslintDisable = (remover) => (result) => {
  if (result.output) {
    result.output = result.output.replace(remover, '')
  }
  return result
}

export const runStandardLinter = (contents, fix) => (
  new Promise((resolve, reject) => {
    const disabledRules = ['no-undef', 'no-unused-vars', 'no-lone-blocks', 'no-labels']
    const eslintDisable = '/* eslint-disable ' + disabledRules.join(', ') + ' */\n'
    standard.lintText(eslintDisable + contents, { fix }, (err, { results }) => {
      if (err) reject(err);
      else resolve(results.map(removeEslintDisable(eslintDisable)))
    });
  })
);

export const formatLineLinterError = (error) => (
  paddingText(10, error.line + ':' + error.column + ': ') +
  paddingText(20, error.filename) +
  error.message + ' ' + '(' + error.ruleId + ')'
)

export const formatLinterError = (line, filename, output) => (error) => {
  error.line += line - 1
  error.filename = filename
  error.fixedCode = output
  return error
}

export const printLinterErrors = (errors) => {
  console.error(errors.map(formatLineLinterError).join('\n'))
}

export const isFix = (options) => !!options._ && options._[0] === 'fix'

export const insertCodeBlock = (code, filename) => {
  const END = '`' + '`' + '`'
  const BEGIN = END + 'js'
  return [
    BEGIN,
    '// ' + filename,
    code,
    END
  ].join('\n')
}

export const replaceAll = (text, pattern, replace) => text.split(pattern).join(replace)

export const fixLinterErrors = async (errors, codeBlocks, targetPath = 'README.md') => {
  let readme = fs.readFileSync(targetPath, 'utf8')
  errors.map(({ filename, fixedCode }) => {
    const code = codeBlocks[filename].contents.replace(/\n$/g, '')
    if (fixedCode) {
      readme = replaceAll(readme, insertCodeBlock(code, filename), insertCodeBlock(fixedCode, filename))
    }
  })
  await saveToFile(targetPath, readme)
}

export async function runLinter (codeBlocks, options) {
  let errors = []
  const fix = isFix(options)
  await forEachCodeBlock(async ({ contents }, filename, codeBlocks) => {
    const { line } = codeBlocks[filename]
    const results = await runStandardLinter(contents, fix)
    results.map(({ messages, output }) => {
      const newErrors = messages.map(formatLinterError(line, filename, output))
      errors = errors.concat(newErrors)
    })
  })(codeBlocks)
  if (fix) await fixLinterErrors(errors, codeBlocks)
  else printLinterErrors(errors)
}

export default runLinter
```

And its tests

```js
// runLinter.test.js
import {
  paddingText,
  runStandardLinter,
  formatLineLinterError,
  formatLinterError,
  countLinterErrors,
  isFix,
  insertCodeBlock,
  removeEslintDisable,
  replaceAll
} from './runLinter'

it('should create padding text', () => {
  assert(paddingText(5, 'hello') === 'hello')
  assert(paddingText(6, 'hello') === 'hello ')
  assert(paddingText(7, 'hello', '_') === 'hello__')
})

it('should format linter error', () => {
  const error = formatLinterError(5, 'example.js', 'const x = 5')({ line: 5 })
  assert(error.line === 9)
  assert(error.filename === 'example.js')
  assert(error.fixedCode === 'const x = 5')
})

it('should count linter errors', () => {
  assert(countLinterErrors([{ messages: [{}, {}] }]) === 2)
  assert(countLinterErrors([{ messages: [{}] }]) === 1)
})

it('should format line linter error', () => {
  const error = {
    line: 5,
    column: 10,
    message: 'message',
    filename: 'example.js',
    ruleId: 'ruleId'
  }
  const line = formatLineLinterError(error)
  assert(line === '5:10:     example.js          message (ruleId)')
})

it('should convert params fix to boolean', () => {
  assert(isFix({ _: ['fix'] }) === true)
  assert(isFix({}) === false)
})

it('should replace all matched substring with pattern', () => {
  assert(replaceAll('aaaaa|aa', 'aa', 'b') === 'bba|b')
})

it('should insert javascript code block', () => {
  assert(insertCodeBlock('const x = 5', 'example.js') === [
    '`' + '`' + '`js',
    '// example.js',
    'const x = 5',
    '`' + '`' + '`'
  ].join('\n'))
})

it('should remove eslint-disable line', () => {
  const eslintDisable = '/* eslint-disable some-rule */'
  const input = {
    output: [
      eslintDisable,
      '// OK'
    ].join('\n')
  }
  const input2 = { output: '// ok' }
  assert(removeEslintDisable(eslintDisable + '\n')(input).output === '// OK')
  assert(removeEslintDisable(eslintDisable + '\n')(input2).output === '// ok')
})

it('should lint text with standard linter', async () => {
  assert(countLinterErrors(await runStandardLinter('1 === 2\n', false)) === 0)
  assert(countLinterErrors(await runStandardLinter('1 === 2', false)) === 1)
  assert(countLinterErrors(await runStandardLinter('1 == 2', false)) === 2)
})

```


### the coverage magic

This module rewrites the coverage data so that you can view the coverage report for README.md. It’s quite complex and thus deserves its own section.

```js
// mapSourceCoverage.js
import path from 'path'
import { forOwn } from 'lodash'

export function mapSourceCoverage (coverage, {
  codeBlocks,
  sourceFilePath,
  targetDirectory
}) {
  const result = { }
  const builder = createReadmeDataBuilder(sourceFilePath)
  for (const key of Object.keys(coverage)) {
    const entry = coverage[key]
    const relative = path.relative(targetDirectory, entry.path)
    if (codeBlocks[relative]) {
      builder.add(entry, codeBlocks[relative])
    } else {
      result[key] = entry
    }
  }
  if (!builder.isEmpty()) result[sourceFilePath] = builder.getOutput()
  return result
}

function createReadmeDataBuilder (path) {
  let nextId = 1
  let output = {
    path,
    s: { }, b: { }, f: { },
    statementMap: { }, branchMap: { }, fnMap: { }
  }
  let empty = true
  return {
    add (entry, codeBlock) {
      const id = nextId++
      const prefix = (key) => `${id}.${key}`
      const mapLine = (line) => codeBlock.line - 1 + line
      const map = mapLocation(mapLine)
      empty = false
      forOwn(entry.s, (count, key) => {
        output.s[prefix(key)] = count
      })
      forOwn(entry.statementMap, (loc, key) => {
        output.statementMap[prefix(key)] = map(loc)
      })
      forOwn(entry.b, (count, key) => {
        output.b[prefix(key)] = count
      })
      forOwn(entry.branchMap, (branch, key) => {
        output.branchMap[prefix(key)] = {
          ...branch,
          line: mapLine(branch.line),
          locations: branch.locations.map(map)
        }
      })
      forOwn(entry.f, (count, key) => {
        output.f[prefix(key)] = count
      })
      forOwn(entry.fnMap, (fn, key) => {
        output.fnMap[prefix(key)] = {
          ...fn,
          line: mapLine(fn.line),
          loc: map(fn.loc)
        }
      })
    },
    isEmpty: () => empty,
    getOutput: () => output
  }
}

function mapLocation (mapLine) {
  return ({ start = { }, end = { } }) => ({
    start: { line: mapLine(start.line), column: start.column },
    end: { line: mapLine(end.line), column: end.column }
  })
}

export default mapSourceCoverage
```

And its tests is quite… ugh!

```js
// mapSourceCoverage.test.js
import mapSourceCoverage from './mapSourceCoverage'
import path from 'path'

const loc = (startLine, startColumn) => (endLine, endColumn) => ({
  start: { line: startLine, column: startColumn },
  end: { line: endLine, column: endColumn }
})
const coverage = {
  '/home/user/essay/src/hello.js': {
    path: '/home/user/essay/src/hello.js',
    s: { 1: 1 },
    b: { 1: [ 1, 2, 3 ] },
    f: { 1: 99, 2: 30 },
    statementMap: {
      1: loc(2, 15)(2, 30)
    },
    branchMap: {
      1: { line: 4, type: 'switch', locations: [
        loc(5, 10)(5, 20),
        loc(7, 10)(7, 25),
        loc(9, 10)(9, 30)
      ] }
    },
    fnMap: {
      1: { name: 'x', line: 10, loc: loc(10, 0)(10, 20) },
      2: { name: 'y', line: 20, loc: loc(20, 0)(20, 15) },
    },
  },
  '/home/user/essay/src/world.js': {
    path: '/home/user/essay/src/world.js',
    s: { 1: 1 },
    b: { },
    f: { },
    statementMap: { 1: loc(1, 0)(1, 30) },
    branchMap: { },
    fnMap: { }
  },
  '/home/user/essay/unrelated.js': {
    path: '/home/user/essay/unrelated.js',
    s: { 1: 1 },
    b: { },
    f: { },
    statementMap: { 1: loc(1, 0)(1, 30) },
    branchMap: { },
    fnMap: { }
  }
}
const codeBlocks = {
  'hello.js': { line: 72 },
  'world.js': { line: 99 }
}
const getMappedCoverage = () => mapSourceCoverage(coverage, {
  codeBlocks,
  sourceFilePath: '/home/user/essay/README.md',
  targetDirectory: '/home/user/essay/src'
})

it('should combine mappings from code blocks', () => {
  const mapped = getMappedCoverage()
  assert(Object.keys(mapped).length === 2)
})

const testReadmeCoverage = (f) => () => (
  f(getMappedCoverage()['/home/user/essay/README.md'])
)

it('should have statements', testReadmeCoverage(entry => {
  const keys = Object.keys(entry.s)
  assert(keys.length === 2)
  assert.deepEqual(keys, [ '1.1', '2.1' ])
}))
it('should have statementMap', testReadmeCoverage(({ statementMap }) => {
  assert(statementMap['1.1'].start.line === 73)
  assert(statementMap['1.1'].end.line === 73)
  assert(statementMap['2.1'].start.line === 99)
  assert(statementMap['2.1'].end.line === 99)
}))
it('should have branches', testReadmeCoverage(({ b }) => {
  assert(Array.isArray(b['1.1']))
}))
it('should have branchMap', testReadmeCoverage(({ branchMap }) => {
  assert(branchMap['1.1'].locations[2].start.line === 80)
  assert(branchMap['1.1'].line === 75)
}))
it('should have functions', testReadmeCoverage(({ f }) => {
  assert(f['1.1'] === 99)
  assert(f['1.2'] === 30)
}))
it('should have function map', testReadmeCoverage(({ fnMap }) => {
  assert(fnMap['1.1'].loc.start.line === 81)
  assert(fnMap['1.2'].line === 91)
}))
```


## acceptance test

```js
// acceptance.test.js
import mkdirp from 'mkdirp'
import fs from 'fs'
import * as buildCommand from './cli/buildCommand'
import * as testCommand from './cli/testCommand'
import * as lintCommand from './cli/lintCommand'

it('works', async () => {
  const example = fs.readFileSync('example.md', 'utf8')
  await runInTemporaryDir(async () => {
    fs.writeFileSync('README.md', example)
    await buildCommand.handler({ })
    assert(fs.existsSync('src/add.js'))
    assert(fs.existsSync('lib/add.js'))
    await testCommand.handler({ })
    assert(fs.existsSync('coverage/lcov.info'))
    await lintCommand.handler({ _: ['fix'] })
    assert(fs.existsSync('README.md'))
    assert(fs.readFileSync('README.md', 'utf8') === example)
  })
})

async function runInTemporaryDir (f) {
  const cwd = process.cwd()
  const testDirectory = '/tmp/essay-acceptance-test'
  mkdirp.sync(testDirectory)
  try {
    process.chdir(testDirectory)
    await f()
  } finally {
    process.chdir(cwd)
  }
}
```


## miscellaneous

### command line handler

```js
// cli/index.js
import * as buildCommand from './buildCommand'
import * as testCommand from './testCommand'
import * as lintCommand from './lintCommand'

export function main () {
  // XXX: Work around yargs’ lack of default command support.
  const commands = [ buildCommand, testCommand, lintCommand ]
  const yargs = commands.reduce(appendCommandToYargs, require('yargs')).help()
  const registry = commands.reduce(registerCommandToRegistry, { })
  const argv = yargs.argv
  const command = argv._.shift() || 'build'
  const commandObject = registry[command]
  if (commandObject) {
    const subcommand = commandObject.builder(yargs.reset())
    Promise.resolve(commandObject.handler(argv)).catch(e => {
      setTimeout(() => { throw e })
    })
  } else {
    yargs.showHelp()
  }
}

function appendCommandToYargs (yargs, command) {
  return yargs.command(command.command, command.description)
}

function registerCommandToRegistry (registry, command) {
  return Object.assign(registry, {
    [command.command]: command
  })
}
```


### saving file

This function wraps around the normal file system API, but provides this benefits:

- It will first read the file, and if the content is identical, it will not re-write the file.

- It displays log message on console.

```js
// saveToFile.js
import fs from 'fs'
import path from 'path'
import mkdirp from 'mkdirp'

export async function saveToFile (filePath, contents) {
  mkdirp.sync(path.dirname(filePath))
  const exists = fs.existsSync(filePath)
  if (exists) {
    const existingData = fs.readFileSync(filePath, 'utf8')
    if (existingData === contents) return
  }
  console.log('%s %s…', exists ? 'Updating' : 'Writing', filePath)
  fs.writeFileSync(filePath, contents)
}

export default saveToFile
```


### run a function for each code block

```js
// forEachCodeBlock.js
export function forEachCodeBlock (fn) {
  return async function (codeBlocks) {
    const filenames = Object.keys(codeBlocks)
    for (const filename of filenames) {
      await fn(codeBlocks[filename], filename, codeBlocks)
    }
  }
}

export default forEachCodeBlock
```
