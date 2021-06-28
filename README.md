# SNKRX Analysis

First let's get the main file of the game

```js repl--
import axios from 'axios'

const requestForMain = axios.get('https://raw.githubusercontent.com/a327ex/SNKRX/master/main.lua')
```

Parse the main file and get the raw data

```js repl--
import luaparse from 'luaparse'

const DEPRECATED_CHARACTERS = [ 'lich', 'launcher' ]

const dbPromise = requestForMain.then(({ data }) => {
  const ast = luaparse.parse(data)
  const initAst = ast?.body?.find(node =>
    node?.type === 'FunctionDeclaration' &&
    node?.identifier?.name === 'init'
  )
  const characterClassesAst = initAst?.body?.find(node =>
    node?.type === 'AssignmentStatement' &&
    node?.variables?.find(variable => variable?.name === 'character_classes')
  )
  const characterTiersAst = initAst?.body?.find(node =>
    node?.type === 'AssignmentStatement' &&
    node?.variables?.find(variable => variable?.name === 'character_tiers')
  )

  const characterClasses = characterClassesAst?.init?.[0]?.fields?.reduce((acc, node) => {
    const key = node?.key?.raw
    const value = node?.value?.fields?.reduce((acc, node) =>
      [...acc, node?.value?.raw],
      []
    )
  
    return key && value
      ? [ ...acc, [ key.replace(/'/g, ''), value.map(v => v.replace(/'/g, '')) ] ]
      : acc
  }, [])

  const characterTiers = characterTiersAst?.init?.[0]?.fields?.reduce((acc, node) => {
    const key = node?.key?.raw
    const value = node?.value?.value
  
    return key && value
      ? [ ...acc, [ key.replace(/'/g, ''), value ] ]
      : acc
  }, [])

  const characterRecords = new Set(characterTiers
    ?.filter(([name]) => !DEPRECATED_CHARACTERS.includes(name))
    ?.map(([name, tier]) => ({ name, tier }))
  )

  const {
    classRecords: preClassRecords,
    characterClassRecords: preCharacterClassRecords,
  } = characterClasses?.reduce((
      { classRecords, characterClassRecords },
      [name, classes]
    ) => ({
      classRecords: [
        ...classRecords,
        ...classes,
      ],
      characterClassRecords: [
        ...characterClassRecords,
        ...(!DEPRECATED_CHARACTERS.includes(name)
          ? classes.map(cls => ({
            class: cls,
            character: name
          }))
          : []
        )
      ]
    }),
    {
      classRecords: [],
      characterClassRecords: []
    }
  )

  const classRecords = new Set([...(new Set(preClassRecords))].map(name => ({ name })))
  const characterClassRecords = new Set(preCharacterClassRecords)

  return {
    characterRecords,
    classRecords,
    characterClassRecords
  }
})
```

Let's see how which characters belong in each class

```js repl--
import fs from 'fs'

dbPromise.then(({
  characterRecords,
  classRecords,
  characterClassRecords
}) => {
  const classToCharacters = [...characterClassRecords].reduce((acc, rec) => ({
    ...acc,
    [rec.class]: acc[rec.class]
      ? [ ...acc[rec.class], [...characterRecords].find(charRec => charRec.name === rec.character) ]
      : [ [...characterRecords].find(charRec => charRec.name === rec.character) ]
  }), {})

  fs.writeFile('./out/classToCharacters.json', JSON.stringify(classToCharacters, null, 4), () => {})
})
```