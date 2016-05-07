
# essay

Generate your JavaScript library out of an essay!
Write your JavaScript library and explain your reasoning in a `README.md` file.

For example, you could write your library like this:

```js
// examples/add.js
export default (a, b) => a + b
```

And write a test for it:

```js
// examples/add.test.js
import add from 'example'
it('should add two numbers', () => {
  assert.equal(add(1, 2) === 3)
})
```


## usage

1. Create your README.md with fenced code blocks.

2. Run `essay`.

3. It should generate files in your "./lib" directory.



## development

You need to use `npm install --force`, because I use `essay` to write `essay`,
but `npm` doesn’t want to install a package as a dependency of itself
(even though it is an older version).



## implementation

### CLI

```js
// cli/index.js
import * as buildCommand from './buildCommand'

export function main () {
  // XXX: Work around yargs’ lack of default command support.
  const commands = [ buildCommand ]
  const yargs = commands.reduce(appendCommandToYargs, require('yargs')).help()
  const registry = commands.reduce(registerCommandToRegistry, { })
  const argv = yargs.argv
  const command = argv._.shift() || 'build'
  const commandObject = registry[command]
  if (commandObject) {
    const subcommand = commandObject.builder(yargs.reset())
    commandObject.handler(argv)
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


#### build command

```js
// cli/buildCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import dumpSourceCodeBlocks from '../dumpSourceCodeBlocks'
import transpileCodeBlocks from '../transpileCodeBlocks'

export const command = 'build'
export const description = 'Builds the README.md file into lib folder.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const codeBlocks = await obtainCodeBlocks()
  await dumpSourceCodeBlocks(codeBlocks)
  await transpileCodeBlocks(codeBlocks)
}
```


### obtaining code blocks

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

This function extracts all the code blocks:

```js
// extractCodeBlocks.js
export function extractCodeBlocks (data) {
  const codeBlocks = { }
  data.replace(/`[`]`js\s+\/\/\s*(\S+).*\n([\s\S]+?)`[`]`/g, (all, filename, contents) => {
    if (codeBlocks[filename]) throw new Error(filename + ' already exists!')
    codeBlocks[filename] = { contents }
    // TODO: add line number
  })
  return codeBlocks
}

export default extractCodeBlocks
```


### dumping code blocks to source files

We’re going to build each file one by one.

```js
// dumpSourceCodeBlocks.js
import extractCodeBlockToSourceFile from './extractCodeBlockToSourceFile'
import transpileFile from './transpileFile'

export async function dumpSourceCodeBlocks (codeBlocks) {
  const filenames = Object.keys(codeBlocks)
  for (const filename of filenames) {
    await extractCodeBlockToSourceFile(filename, codeBlocks[filename])
  }
}

export default dumpSourceCodeBlocks
```

```js
// transpileCodeBlocks.js
import transpileFile from './transpileFile'

export async function transpileCodeBlocks (codeBlocks) {
  const filenames = Object.keys(codeBlocks)
  for (const filename of filenames) {
    await transpileFile(filename)
  }
}

export default transpileCodeBlocks
```

Now, to write each code block to a file:

```js
// extractCodeBlockToSourceFile.js
import path from 'path'

import saveToFile from './saveToFile'

export async function extractCodeBlockToSourceFile (filename, { contents }) {
  const targetFilePath = path.join('src', filename)
  await saveToFile(targetFilePath, contents)
}

export default extractCodeBlockToSourceFile
```

### transpilation using Babel

```js
// transpileFile.js
import path from 'path'
import { transformFileSync } from 'babel-core'

import saveToFile from './saveToFile'

export async function transpileFile (filename) {
  const sourceFilePath = path.join('src', filename)
  const targetFilePath = path.join('lib', filename)
  const { code } = transformFileSync(sourceFilePath, {
    presets: [
      require('babel-preset-es2015'),
      require('babel-preset-stage-2'),
    ],
    plugins: [
      require('babel-plugin-transform-runtime')
    ]
  })
  await saveToFile(targetFilePath, code)
}

export default transpileFile
```


### utilities

This function saves files to the file system. Just a utility functions.

```js
// saveToFile.js
import fs from 'fs'
import path from 'path'
import mkdirp from 'mkdirp'

export async function saveToFile (filePath, contents) {
  mkdirp.sync(path.dirname(filePath))
  console.log('Writing %s…', filePath)
  fs.writeFileSync(filePath, contents)
}

export default saveToFile
```
