---
theme: default
title: Build Web App with LLM
author: Tan Phat Nguyen
presenter: false
colorSchema: auto
transition: slide-left
mdc: true
---

# Build Web App with LLM
Tan Phat Nguyen

<!--
Note
-->

---

<Toc minDepth="1" maxDepth="1" />

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
  "model": "llama3.2",
  "prompt": "Explain LLM in 3 sentences.',
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

console.log(response.message)
```

---

# Integration in App

- Generate card's definition
```ts
async function generateDef(c: Card) {
  c.def = await $generate(`"${c.term}": Provide a short, plain text definition without any redundant information.`)
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

// {
//   "cards": [
//     { "term": "Capital of Germany", "def": "Berlin" }
//     ...
//   ]
// }

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

// Here are generated cards as your instruction:
// {
//   "cards": [
//     { "term": "Capital of Germany", "def": "Berlin" }
//     ...
//   ]
// }

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

console.log(schema.parse(JSON.parse(output)))
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
layout: two-cols
---

# SQLite with Vector

- using [`sqlite-vec`](https://github.com/asg017/sqlite-vec) extension
```sql
create virtual table "sets" using vec0 (
  id integer primary key autoincrement,
  +title text,
  +cards text,
  +tags text,
  embedding float[768],
  +createAt text,
);
```

```sql
  -- Auxiliary column: unindexed, fast lookups
  +title text,

  -- Vector text embedding with 768 dimensions
  embedding float[768],
```

::right::

<h1 class="opacity-0">&npsb;</h1>

- selecting most matched set

```sql
select
      id,
      title,
      cards,
      tags,
      createAt,
      vec_distance_cosine(embedding, ?) as distance
from sets
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

<!-- Demo with chat in app -->

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

<!-- Demo with summary -->

---
layout: center
class: text-center
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
layout: center
class: text-center
---

PDF, QR Code

<PoweredBySlidev mt-10 />
