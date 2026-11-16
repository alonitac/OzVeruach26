# Intro to UI

Building user interfaces is not the focus of this course.

Usually, a UI app is the main layer in front of your services, but it stores nothing of its own. 
Usually the browser talks to it, and it talks to the APIs, but there are cases where users directly talk to the APIs via their browsers. 

## The client and the server

A web app runs in two places.

The **client** is the user's browser. It draws the page, and it reacts when the user types and clicks.

The **server** is a program on a machine you run. It can call internal services, and send the browser a finished page.

The browser understands three languages:

| Language | Job | Example |
| --- | --- | --- |
| **HTML** | Structure and content | A heading, a table, a form, a button |
| **CSS** | Look | Colors, fonts, spacing, layout |
| **JavaScript** | Behavior | Click Save, show "Saving...", show an error |

## Node.js and TypeScript

The browser runs JavaScript. On the server you can use any language (Python, Go, etc.). **Node.js** is a language that runs JavaScript on te server, too, so a UI can be one language end to end: the page, the button, and the UI's own API routes. This is why it's the most popular language for building UIs.

Plain JavaScript is loose. **TypeScript** is the same language with types, closer to Python type hints. 

Before the app runs, TypeScript is **transpiled** into JavaScript: 

```ts
function label(name: string, count: number): string {
  return name + " (" + count + ")";
}
```

becomes:

```js
function label(name, count) {
  return name + " (" + count + ")";
}
```

`.ts` files are transpiled into `.js` files to be run in the server side, while `.tsx` files are transpiled into `.js` files to be run in the browser.

A **web framework** is the toolkit around those files: pages, URLs, and the server that serves them. We use **Next.js**, a React framework. 
In a Next.js project, Next.js transpiles when you start the app and puts the result in `.next/`. Git ignores that folder.

## A frontend project

**npm** is the package manager, the way `uv` is for Python.

- **`package.json`** lists dependencies, dev dependencies (TypeScript, ESLint), and scripts. 
- **`npm install`** downloads them into **`node_modules/`**. The folder is huge. Never edit it or commit it. You can delete it and run `npm install` again.
- **`package-lock.json`** pins the exact versions, like `uv.lock`. Commit it.
- **`npm run <script>`** runs a script from `package.json`.

| Command | What it does |
| --- | --- |
| `npm run dev` | Development server. Rebuilds on each save. |
| `npm run build` | Type check, then a production build. |
| `npm run start` | Runs the production build. |
| `npm run lint` | ESLint: likely bugs and bad patterns. |

> [!IMPORTANT]
> Code that works in `npm run dev` can fail `npm run build` or `npm run lint`. Ask the agent to run both before it says it's done.

A small Next.js app looks like this. Notice that the **folders are URLs**: `src/app/page.tsx` is `/`, `src/app/items/new/page.tsx` is `/items/new`, and `src/app/api/items/route.ts` is `POST /api/items`.

```text
ui/
├── src/
│   ├── app/
│   │   ├── layout.tsx          # the frame around every page
│   │   ├── page.tsx            # the page at /
│   │   ├── globals.css
│   │   ├── items/new/page.tsx  # the page at /items/new
│   │   └── api/items/route.ts  # the endpoint POST /api/items
│   ├── components/
│   │   └── ItemForm.tsx
│   └── lib/
│       └── api.ts              # calls to your backend
├── package.json
├── package-lock.json
├── tsconfig.json
├── .env.example                # which variables are needed (committed)
└── .env.local                  # the real values (not committed)
```

`.env.local` holds settings such as the backend URL. It is the `.env` of the UI.

## How a page gets to the screen

The browser still receives only HTML, CSS, and JavaScript. Next.js turns your TypeScript into that JavaScript. **Tailwind** turns short class names (`p-4`, `text-sm`, `flex`) into CSS. **React** builds the page out of components.

A **component** is a function that returns HTML-like code (**TSX**). You pass it data, called **props**, and you can reuse it. 

Let's see an example of a form component.


```tsx
"use client";

import { useState } from "react";

function ItemForm() {
  const [name, setName] = useState("");                  // the text in the field
  const [saving, setSaving] = useState(false);           // waiting for the server?
  const [error, setError] = useState<string | null>(null);

  async function onSave() {
    setSaving(true);
    setError(null);
    const res = await fetch("/api/items", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name }),
    });
    setSaving(false);
    if (!res.ok) setError("Could not save");
  }

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault();
        onSave();
      }}
    >
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <button disabled={saving}>{saving ? "Saving..." : "Save"}</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

A form has to remember what the user is doing. Those remembered values are **state**. `useState` returns the current value and a function that replaces it. Calling that function makes React **re-render**: it runs the component again and updates the screen.

For example, the field shows `name` starts empty. Each keystroke calls `setName`, React runs `ItemForm` again, and the field shows the new text. Click Save and `setSaving(true)` runs it once more: the button reads "Saving..." and ignores further clicks. If the request fails, `setError` runs it again and "Could not save" appears under the button. `"use client"` marks this as browser code, because state and clicks happen there. 

Component libraries give you buttons, forms, tables, and dialogs already built. Common ones: **shadcn/ui** (copies each component into your project), **Material UI**, and **Ant Design**.
