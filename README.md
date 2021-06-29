# SNKRX Analysis

## Requirements

TODO

## How to run this file

TODO

## Analysis

First, let's get the main file of the game.

```js repl--
import axios from 'axios'

const requestForMain = axios.default.get('https://raw.githubusercontent.com/a327ex/SNKRX/master/main.lua')
```

Parse the main file and get the raw data.

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
    preClassRecords,
    preCharacterClassRecords,
  } = characterClasses?.reduce((
      { preClassRecords, preCharacterClassRecords },
      [name, classes]
    ) => ({
      preClassRecords: [
        ...preClassRecords,
        ...classes,
      ],
      preCharacterClassRecords: [
        ...preCharacterClassRecords,
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
      preClassRecords: [],
      preCharacterClassRecords: []
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

Let's see which characters belong in each class.

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

Looking into the `out` directory after running this, you'll find the JSON file we created above: `classToCharacters.json`. That's good info but it's a little hard to read.

Let's try emitting a Markdown table of characters, grouped by class in alphabetical order, then grouped again by tier in numerical order.

```js repl--
import tablemark from 'tablemark'

dbPromise.then(({
  characterRecords,
  classRecords,
  characterClassRecords
}) => {
  const headers = ['Class', 'Tier', 'Character']

  const classSubtables = [...characterClassRecords]
    .reduce((acc, rec) => {
      const char = [...characterRecords].find(charRec => charRec.name === rec.character)
      const charRow = { class: rec.class, tier: char.tier, character: char.name }
      const classMatchesRecord = ([cls]) => cls === rec.class

      return [
        ...acc.filter(x => !classMatchesRecord(x)),
        [ rec.class, [ ...acc.find(classMatchesRecord)?.[1] ?? [], charRow ] ]
      ]
    }, [])
    .sort(([cls1], [cls2]) => cls1 !== cls2 ? cls1 > cls2 ? 1 : -1 : 0)

  const classTierSubtables = classSubtables.reduce((acc, [cls, rows]) => {
    const tierSubtable = rows
      .reduce((acc, charRow) => {
        const tierMatchesRecord = ([tier]) => tier === charRow.tier

        return [
          ...acc.filter(x => !tierMatchesRecord(x)),
          [ charRow.tier, [ ...acc.find(tierMatchesRecord)?.[1] ?? [], charRow ] ]
        ]
      }, [])
      .sort(([tier1], [tier2]) => tier1 !== tier2 ? tier1 > tier2 ? 1 : -1 : 0)

    return [ ...acc, [ cls, tierSubtable ] ]
  }, [])

  const charactersTable = classTierSubtables
    .flatMap(([cls, tierSubtable]) =>
      tierSubtable.flatMap(([tier, rows]) => rows)
    )

  fs.writeFile('./out/charactersTable.md', tablemark(charactersTable, null, 4), () => {})
})
```

At time of writing, the above outputs the following.

| Class     | Tier  | Character     |
| --------- | ----- | ------------- |
| conjurer  | 2     | saboteur      |
| conjurer  | 2     | hunter        |
| conjurer  | 2     | carver        |
| conjurer  | 3     | engineer      |
| conjurer  | 3     | illusionist   |
| curser    | 2     | barbarian     |
| curser    | 2     | jester        |
| curser    | 2     | silencer      |
| curser    | 3     | bane          |
| curser    | 3     | infestor      |
| curser    | 3     | usurer        |
| enchanter | 2     | squire        |
| enchanter | 2     | chronomancer  |
| enchanter | 3     | stormweaver   |
| enchanter | 3     | flagellant    |
| enchanter | 4     | fairy         |
| explorer  | 1     | vagrant       |
| forcer    | 2     | sage          |
| forcer    | 2     | hunter        |
| forcer    | 3     | juggernaut    |
| forcer    | 3     | barrager      |
| forcer    | 4     | psykino       |
| forcer    | 4     | warden        |
| healer    | 1     | cleric        |
| healer    | 2     | carver        |
| healer    | 3     | psykeeper     |
| healer    | 4     | fairy         |
| healer    | 4     | priest        |
| mage      | 1     | magician      |
| mage      | 2     | wizard        |
| mage      | 2     | chronomancer  |
| mage      | 2     | cryomancer    |
| mage      | 3     | elementor     |
| mage      | 3     | spellblade    |
| mage      | 3     | pyromancer    |
| mage      | 4     | psykino       |
| mercenary | 1     | merchant      |
| mercenary | 2     | miner         |
| mercenary | 3     | usurer        |
| mercenary | 3     | gambler       |
| mercenary | 4     | thief         |
| nuker     | 2     | wizard        |
| nuker     | 2     | saboteur      |
| nuker     | 2     | sage          |
| nuker     | 3     | elementor     |
| nuker     | 3     | pyromancer    |
| nuker     | 4     | blade         |
| nuker     | 4     | cannoneer     |
| nuker     | 4     | plague_doctor |
| nuker     | 4     | vulcanist     |
| psyker    | 1     | vagrant       |
| psyker    | 2     | psychic       |
| psyker    | 3     | psykeeper     |
| psyker    | 3     | flagellant    |
| psyker    | 4     | psykino       |
| ranger    | 1     | archer        |
| ranger    | 2     | dual_gunner   |
| ranger    | 2     | hunter        |
| ranger    | 3     | barrager      |
| ranger    | 4     | cannoneer     |
| ranger    | 4     | corruptor     |
| rogue     | 1     | scout         |
| rogue     | 2     | outlaw        |
| rogue     | 2     | saboteur      |
| rogue     | 2     | dual_gunner   |
| rogue     | 2     | beastmaster   |
| rogue     | 2     | jester        |
| rogue     | 3     | spellblade    |
| rogue     | 3     | assassin      |
| rogue     | 4     | thief         |
| sorcerer  | 1     | arcanist      |
| sorcerer  | 2     | witch         |
| sorcerer  | 2     | silencer      |
| sorcerer  | 2     | psychic       |
| sorcerer  | 3     | illusionist   |
| sorcerer  | 3     | gambler       |
| sorcerer  | 4     | vulcanist     |
| sorcerer  | 4     | warden        |
| swarmer   | 2     | beastmaster   |
| swarmer   | 3     | host          |
| swarmer   | 3     | infestor      |
| swarmer   | 4     | corruptor     |
| voider    | 2     | cryomancer    |
| voider    | 2     | witch         |
| voider    | 3     | pyromancer    |
| voider    | 3     | assassin      |
| voider    | 3     | bane          |
| voider    | 3     | usurer        |
| voider    | 4     | plague_doctor |
| warrior   | 1     | swordsman     |
| warrior   | 2     | outlaw        |
| warrior   | 2     | squire        |
| warrior   | 2     | barbarian     |
| warrior   | 3     | juggernaut    |
| warrior   | 4     | blade         |
| warrior   | 4     | highlander    |
