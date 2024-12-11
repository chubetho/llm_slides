---
theme: default
title: Build Web App with LLM
titleTemplate: '%s'
author: Tan Phat Nguyen
info: false
keywords: LLM,Ollama,SQLite,Nuxt
download: true
presenter: true
twoslash: false
monacoTypesSource: none
contextMenu: false
transition: slide-left
mdc: true
hideInToc: true
favicon: '/bot.png'
drawings:
  enabled: false
export:
  format: pdf
  withToc: true
exportFilename: 'llm'
---

# Build Web App with LLM
Tan Phat Nguyen

<!--
Note
-->

---
hideInToc: true
---

# Contents

<Toc />

---

# Techstacks

- [Nuxt](https://nuxt.com/) <logos-nuxt-icon />
  - Full-stack framework built on [VueJS](https://vuejs.org)

- [SQLite](https://www.sqlite.org/) <logos-sqlite />
  - Lightweight and fast
  - Vector support with [extension](https://github.com/asg017/sqlite-vec)

- [Ollama](https://ollama.com/) <IconOllama class="size-6"/>
  - Model management
  - Easy to use

---

# Ollama CLI

- Download CLI [here](https://ollama.com/download)
- Find a model [here](https://ollama.com/search)
```sh {none|1|3|5|all}
ollama pull llama3.2:3b

ollama show llama3.2:3b

ollama run llama3.2:3b
```

- Served at [http://localhost:11434](http://localhost:11434)
```sh {none|all}
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2:3b",
  "prompt": "Explain LLM in 3 sentences.",
  "stream": false
}'
```

---

# Ollama-JS

- `ollama.generate`
```ts {monaco-run}{autorun:false}
import ollama from 'ollama/browser'

const response = await ollama.generate({
  model: 'llama3.2:3b',
  prompt: 'Explain LLM in 3 sentences.',
  stream: false
})

console.log(response.response)
```

---
hideInToc: true
---

# Ollama-JS

- `ollama.generate` with `steam = true`
```ts {monaco-run}{autorun:false}
import ollama from 'ollama/browser'

let output = ''
const response = await ollama.generate({
  model: 'llama3.2:3b',
  prompt: 'Explain LLM in 3 sentences.',
  stream: true
})

for await (const r of response) {
  output += r.response
  console.clear()
  console.log(output)
}
```

---
hideInToc: true
---

# Ollama-JS

- `ollama.chat`
```ts {monaco-run}{autorun:false}
import ollama from 'ollama/browser'

const response = await ollama.chat({
  model: 'llama3.2:3b',
  messages: [
    { role: 'user', content: 'Explain LLM in 3 sentences.' }
  ],
  stream: false
})

console.log(response.message.content)
```

---
hideInToc: true
---

# Ollama-JS

- `ollama.chat`
```ts {monaco-run}{autorun:false}
import ollama from 'ollama/browser'

const response = await ollama.chat({
  model: 'llama3.2:3b',
  messages: [
    { role: 'system', content: `You explain like I'm five years old.` },
    { role: 'user', content: 'Explain LLM in 3 sentences.' }
  ],
})

console.log(response.message.content)
```

---

# Integration in App

- Generate card's definition
```ts
async function generateDef(card: Card) {
  card.def = await $generate(`
    "${card.term}": Provide a short, plain text definition without any redundant information.
  `)
}
```

<v-click>

- `$generate` function

```ts
async function $generate(promt: string) {
  const response = await ollama.generate({
    model: 'llama3.2:3b',
    prompt,
    stream: false
  })

  return response.response
}
```
</v-click>

---
hideInToc: true
---

# Integration in App

- Generate cards

````md magic-move
```ts
const response = await $generate(
  `Using the reference cards: ${set.value.cards}, generate new, meaningful cards in JSON format using following
  JSON schema:
  {
    "cards": [
      { "term": "string", "def": "string" }
    ]
  }`
)
```

```ts
const response = await $generate(
  `Using the reference cards: ${set.value.cards}, generate new, meaningful cards in JSON format using following
  JSON schema:
  {
    "cards": [
      { "term": "string", "def": "string" }
    ]
  }`
)

// "{
//   "cards": [
//     { "term": "Capital of Germany", "def": "Berlin" }
//     ...
//   ]
// }"

const cards = JSON.parse(response).cards // Array of cards
```

```ts
const response = await $generate(
  `Using the reference cards: ${set.value.cards}, generate new, meaningful cards in JSON format using following
  JSON schema:
{
  "cards": [
    { "term": "string", "def": "string" }
  ]
}`
)

// "Here are generated cards as your instruction:
// {
//   "cards": [
//     { "term": "Capital of Germany", "def": "Berlin" }
//     ...
//   ]
// }"

const cards = JSON.parse(response).cards // SyntaxError: Unexpected token 'H', "Here are  "... is not valid JSON
```
````

---

# Validation with `zod`

```ts {monaco-run}
import { z } from 'zod'

const schema = z.object({
  cards: z.object({
    term: z.string().describe('This is term'),
    def: z.string().describe('This is def')
  }).array()
})

const output = `{
  "cards": [
    { "term": "Capital of Germany", "def": "Berlin" }
  ]
}`

const cards = schema.parse(JSON.parse(output)).cards
console.log(cards)
```

<!--
Change Berlin to null then number or not exists
-->

---

# Create JSON Schema with `zod-to-json-schema`

```ts {monaco-run}
import { z } from 'zod'
import { zodToJsonSchema } from 'zod-to-json-schema'

const schema = z.object({
  cards: z.object({
    term: z.string().describe('This is term'),
    def: z.string().describe('This is def')
  }).array()
})

console.log(zodToJsonSchema(schema))
```
<style>
.slidev-monaco-container {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
}
</style>

---

# Upgrade `$generate` function

````md magic-move
```ts
async function $generate(promt: string) {
  const response = await ollama.generate({
    // ...
  })

  return response.response
}
```

```ts
async function $generate(prompt: string, schema?: ZodSchema) {
  const format = schema ? zodToJsonSchema(schema) : undefined

  const response = await ollama.generate({
    // ...
    format,
  })

  return response.response
}
```

```ts
async function $generate(prompt: string, schema?: ZodSchema) {
  const format = schema ? zodToJsonSchema(schema) : undefined

  const generate = async () => {
    const response = await ollama.generate({
      // ...
      format,
    })

    return response.response
  }

  return generate()
}
```

```ts{*|10-11|12-18|21|*}
async function $generate(prompt: string, schema?: ZodSchema) {
  const format = schema ? zodToJsonSchema(schema) : undefined

  const generate = async () => {
    const response = await ollama.generate({
      // ...
      format,
    })

    if(!schema)
      return response.response

    try {
      return schema.parse(JSON.parse(response.response))
    }
    catch (error) {
      return generate()
    }
  }

  return generate()
}
```
````

<!--
Of course, there muss be a stop in this recursive funciton. But because of the limitation of places
I will not show it here. And also some error handling when recursive stop such as notify user with it
-->

---
layout: center
class: text-center
hideInToc: true
---

# Demo

<style>
h1{
  font-size: 5rem;
}
</style>

<!--
Demo with generate tests on set
-->

---
disabled: true
---

# SQLite with Vector

- [`vec0`](https://alexgarcia.xyz/sqlite-vec/features/vec0.html#metadata) virtual tables

| Column | Description | Benefits | Limitations|
| - | - | - | - |
| Metadata | Boolean, integer, float, or text data | Can be used in the `WHERE` of KNN query | Slower full scan, slightly inefficient with long strings (`> 12` characters) |
| Auxiliary | Any kind of data | Eliminates need for an external `JOIN` | Cannot appear in the `WHERE` of KNN query |
| Partition Key | Internally shards vector index on a given key | `SELECT` query much faster | Can cause oversharding and slow KNN if not used carefully. |

---
layout: two-cols
hideInToc: true
---

# SQLite with [extension](https://alexgarcia.xyz/sqlite-vec)

- creating `sets` table
```sql
create virtual table "sets" using vec0 (
  id integer primary key autoincrement,
  title text partition key,
  +cards text,
  +tags text,
  embedding float[768],
  createAt text partition key,
);
```

```sql
  -- Auxiliary columns
  +cards text
  +tags text

  -- Vector columns
  embedding float[768],

  -- Partition Key columns:
  title text partition key
  createAt text partition key
```

::right::

<h1 class="opacity-0">&npsb;</h1>

<v-clicks>

- selecting most matched set

```sql
select
      id,
      title,
      cards,
      tags,
      createAt,
      vec_distance_cosine(embedding, input) as distance
from sets
limit 5
order by distance;
```

<div style="text-align: left;">
$$
\begin{aligned}
\text{Cosine Distance} &= 1 - \cos(\theta) \\
&= 1 - \frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\| \|\mathbf{v}\|}
\end{aligned}
$$
</div>

</v-clicks>

<style>
.slidev-layout {
  gap: 2rem
}

.katex-html{
  text-align: left
}
</style>

---

# Embeddings

```ts {monaco-run}
import ollama from 'ollama/browser'

const response = await ollama.embed({
  model: 'nomic-embed-text',
  input: 'Hello from the other side.',
})

console.log('Length:', response.embeddings[0].length)
console.log('First 10:', response.embeddings[0].slice(0, 10))
```

```ts
// Before inserting new set into DB
set.embedding = await $embed(JSON.stringify({
  title: set.title,
  cards: set.cards,
  tags: set.tags,
}))
```

---
layout: center
class: text-center
hideInToc: true
---

# Demo

<style>
h1{
  font-size: 5rem;
}
</style>

<!--
Demo with search for document based on vector
-->

---

# Chat with LLM

- Basic

````md magic-move
```ts
const messages = [
  { role: 'system', content: `You are an assistant for my studies.` }
]
```

```ts {3|6-7}
const messages = [
  { role: 'system', content: `You are an assistant for my studies.` },
  { role: 'user', content: 'Explain LLM in 3 sentences.' },
]

const response = await $chat(messages)
messages.push({role: 'assistant', content: response.content })
```

```ts {4|5|9-10|6|*}
const messages = [
  { role: 'system', content: `You are an assistant for my studies.` },
  { role: 'user', content: 'Explain LLM in 3 sentences.' },
  { role: 'assistant', content: 'A Large Language Model (LLM) is a type of...' },
  { role: 'user', content: 'Give me a real-case example.' },
  // ...
]

const response = await $chat(messages)
messages.push({role: 'assistant', content: response.content })
```
````

<!--
Demo with chat in app
-->

---
hideInToc: true
---

# Chat with LLM

- Upload file

<v-click>

````md magic-move
```ts
type Message = {
  role: 'system' | 'assistant' | 'user',
  content: string
}
```

```ts {*|3,4}
type Message =
(
  { role: 'system', type: 'text' | 'hidden' } |
  { role: 'system', type: 'file', name: string } |
  { role: 'assistant' | 'user' }
) & { content: string }
```
````

</v-click>

<v-click>

```ts
messages.value.push({
  role: 'system',
  content: 'You are a helpful assistant knowledgeable about the following document:',
  type: 'file',
  name: file.name,
})

for (const c of $chunk(content.toString(), { max: 2048 })) {
  messages.value.push({
    role: 'system',
    content: c,
    type: 'hidden',
  })
}
```
</v-click>

<!--
Demo with summary in chat
-->

---
layout: center
class: text-center
hideInToc: true
---

# Demo

<style>
h1{
  font-size: 5rem;
}
</style>

<!--
Demo with summary from pdf or youtube
-->

---

# Our Journey

<v-clicks every="1">

1. <Link to="3">Find Technology Stack</Link> (Nuxt, SQLite, Ollama)

2. <Link to="4">Experiment with Ollama</Link> (CLI, JS library)

3. <Link to="9">Integrate Ollama into the Application</Link> (Structure Output)

4. <Link to="16">Develop a Simple Use Case with Embeddings</Link> (Search for Sets)

5. <Link to="18">Build Chat Functionality</Link> (Basic, File Upload)

6. <Link to="20">Add Summary Functionality</Link> (Based on Chat Features)

</v-clicks>

<style>

ol{
  @apply pt-12 flex flex-col gap-4;
}
</style>

---
hideInToc: true
---

# Thank you

<div class="flex gap-32 mt-16">
  <div class="shrink-0 grow flex flex-col items-center">
    <img src="/qr.png" class="size-64" />
    <a class="mt-6" href="https://chubetho.github.io/llm_slides">https://chubetho.github.io/llm_slides</a>
    <p class="mt-3 text-black/50 dark:text-white/50"> Slides </p>

  </div>

   <div class="shrink-0 grow flex flex-col items-center">
    <img src="/project.png" class="size-64" />
    <a class="mt-6" href="https://github.com/chubetho/LLM">https://github.com/chubetho/LLM</a>
    <p class="mt-3 text-black/50 dark:text-white/50"> Source Code </p>

  </div>
</div>

---
layout: center
---

<PoweredBySlidev />
