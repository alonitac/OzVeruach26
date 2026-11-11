# Intro to UI Apps with Next.js

In [Task P1](../tasks/p1_registry_api.md) you built the registry API. It works, but a grower won't manage the farm with `curl` or with the Swagger page. AgroPilot needs a screen: the farms, their plots, and a form to add a new plot.

Frontend engineering is not the focus of this course. The coding agent will write most of the UI. Your job is to understand what it built, where each piece runs, and how to check it.

In this tutorial you'll learn how a UI app works, using an **AgroPilot UI** as the example: a thin Next.js app on top of `registry-service`. In the exercises, you'll build it and practice on it.

```text
 ┌───────────┐  POST /api/plots  ┌──────────────────┐   GET/POST /v1/...   ┌────────────────────┐
 │  Browser  │ ────────────────► │  Next.js server  │ ───────────────────► │  registry-service  │ ──► registry.db
 │  (user)   │ ◄──────────────── │  (services/ui)   │ ◄─────────────────── │  (FastAPI, :8000)  │
 └───────────┘   HTML, JSON      └──────────────────┘   JSON, problem+json └────────────────────┘
```

The UI has no database. The farms and plots are stored in `registry-service`.

## The client and the server

A web app runs in two places.

The **client** is the user's browser. It draws the page, and it reacts when the user types and clicks.

The **server** is a program on a machine you run. In the diagram above, that program is the Next.js server. Notice it's a second server: `registry-service` is still there, behind it.

Each piece of the UI runs in one of those two places. You'll see which piece is which later.

The browser understands three languages: **HTML**, **CSS**, and **JavaScript**.

**HTML** is structure and content. In the AgroPilot UI that is the farm headings, the plots table, the form fields, and the Save button.

**CSS** is the look: colors, fonts, sizes, and layout. The table rows have a line between them. The error from the API is in a red box.

**JavaScript** is behavior. It runs when you click Save, shows "Saving..." while you wait, and shows the error when the registry rejects the plot.

| Language | Job |
| --- | --- |
| **HTML** | Structure and content |
| **CSS** | Look |
| **JavaScript** | Behavior |

## Node.js: JavaScript on the server

The browser runs JavaScript. On the server we can use any language we want: Python, Go, etc. `registry-service` is Python.
In 2009, Node.js was created to run JavaScript on the server.

The benefit of using Node.js for the UI is that the whole UI is one language: the page, the button, and the UI's own API routes.

## TypeScript

Plain JavaScript, often called **vanilla JavaScript**, is messy and hard to write directly. **TypeScript** lets developers write the same app in a much clearer and stricter way. Files of backend logic end in `.ts`, and files that also contain UI components end in `.tsx`.

Before the app runs, that TypeScript goes through **transpilation**, a translation into vanilla JavaScript. Node.js then runs that JavaScript on the server, and the browser runs it on the client: TypeScript is only a better way to write, and the code that actually runs is JavaScript.

```ts
function plotLabel(name: string, areaM2: number): string {
  return name + " (" + areaM2 + " m²)";
}
```

transpiles to:

```js
function plotLabel(name, areaM2) {
  return name + " (" + areaM2 + " m²)";
}
```

The `: string` and `: number` marks are there to help the writing, like type hints in Python. Transpilation removes them, and what remains is vanilla JavaScript.

You write the `.ts` and `.tsx` files, and you leave the generated JavaScript alone. In a Next.js project, Next.js does the transpilation when you start the app, and puts the result in a folder named `.next/`. Git ignores that folder.


## The pieces of a frontend project

### npm

**npm** is the package manager, the way `uv` is for Python. It installs code other people wrote, and it runs the project's commands.

- **`package.json`** lists the `dependencies` (needed to run the app), the `devDependencies` (needed only while developing: TypeScript, ESLint), and the `scripts`. It's the `pyproject.toml` of the UI.
- **`npm install`** reads `package.json` and downloads everything into **`node_modules/`**, the `.venv` of the UI. The folder is huge. Never edit it or commit it. You can delete it and run `npm install` again.
- **`package-lock.json`** records the exact versions, like `uv.lock`. Commit it, so everyone gets the same ones.
- **`npm run <script>`** runs a script from `package.json`:

| Command | What it does |
| --- | --- |
| `npm run dev` | Development server. Next.js transpiles on each save, and the page updates. |
| `npm run build` | Type check, then a production build (another transpilation, stricter than `dev`). |
| `npm run start` | Runs the production build. |
| `npm run lint` | ESLint: checks the code against a set of rules, and reports likely bugs and bad patterns. |

> [!IMPORTANT]
> Code that works in `npm run dev` can fail the type check in `npm run build`, or fail `npm run lint`. Always ask the agent to run `build` and `lint` before it says it's done.

> [!WARNING]
> npm packages run code on your computer when installed, and attackers publish look-alike names. When the agent adds a package, check the `package.json` diff. `npm audit` lists known vulnerabilities.

### The project

A Next.js app for AgroPilot, next to `registry-service`, could look like this:

```text
ArgoPilot/
└── services/
    ├── registry-service/            # the API from Task P1
    └── ui/
        ├── src/
        │   ├── app/                     # every folder is a URL
        │   │   ├── layout.tsx           # the frame around every page: <html>, fonts
        │   │   ├── page.tsx             # the page at /: the farms and their plots
        │   │   ├── loading.tsx          # shown while the page loads its data
        │   │   ├── globals.css          # global styles + Tailwind
        │   │   ├── plots/new/page.tsx   # the page at /plots/new: the create-plot form
        │   │   └── api/plots/route.ts   # the endpoint POST /api/plots
        │   ├── components/
        │   │   └── PlotForm.tsx         # the form: fields, Save, the error box
        │   └── lib/
        │       └── registry.ts          # the calls to registry-service
        ├── package.json                 # dependencies and scripts
        ├── package-lock.json            # exact versions; commit this
        ├── tsconfig.json                # TypeScript settings
        ├── eslint.config.mjs            # the ESLint rules
        ├── next.config.ts               # Next.js settings
        ├── .env.example                 # which variables are needed (committed)
        └── .env.local                   # the real values (NOT committed)
```

In Next.js, **folders are URLs**: `src/app/page.tsx` is `/`, `src/app/plots/new/page.tsx` is `/plots/new`, and `src/app/api/plots/route.ts` is the endpoint `/api/plots`.

Notice that the UI has its own small API (`/api/plots`), separate from the registry API (`/v1/plots`). We'll see why soon.

`.env.local` holds the UI's settings, like the address of `registry-service`: `REGISTRY_URL=http://127.0.0.1:8000`. It's the `.env` of the UI.


## How a page gets to the screen

The browser still receives only HTML, CSS, and JavaScript. Next.js transpiles your TypeScript into that JavaScript. Tailwind turns short class names into CSS. React components describe the HTML.

### Components

A **component** is a reusable piece of UI, written as a function that returns HTML-like code (**TSX**). The plots table in `page.tsx` could be:

```tsx
{plots.map((plot) => (
  <tr key={plot.id} className="border-b hover:bg-gray-50">
    <td className="px-3 py-2 font-medium">{plot.name}</td>
    <td className="px-3 py-2">{plot.crop_id}</td>
    <td className="px-3 py-2 text-right">{plot.area_m2} m²</td>
  </tr>
))}
```

For every plot from `GET /v1/plots`, draw a table row. The fields are the ones from the API (`name`, `crop_id`, `area_m2`), so when the API changes a field name, the UI breaks.

`PlotForm` also has **state**: values that change while you use it.

```tsx
const [name, setName] = useState("");                     // what you typed in "Name"
const [areaM2, setAreaM2] = useState("");                 // what you typed in "Area"
const [saving, setSaving] = useState(false);              // show "Saving..."?
const [error, setError] = useState<string | null>(null);  // the `detail` from the API
```

When the state changes, React **re-renders**: it runs the component again, and updates only what changed on the screen.

### Tailwind CSS

Instead of a separate CSS file, you add short class names to each element:

| Class | Means |
| --- | --- |
| `p-4`, `px-3`, `mt-6` | Padding (space inside), margin (space outside) |
| `text-sm`, `font-bold`, `text-gray-500` | Font size, weight, color |
| `bg-red-50`, `rounded`, `border` | Background, rounded corners, border |
| `flex`, `gap-2`, `w-64` | Layout: a row or column, space between, width |
| `sm:...`, `hover:...`, `dark:...` | Only on screens ≥ 640px / on mouse over / in dark mode |

## Server and client components

The most important concept when you vibe-code with Next.js. Every component runs in **one** of two places:

| | **Server component** (the default) | **Client component** (`"use client"` at the top) |
| --- | --- | --- |
| Runs | On the Next.js server | In the user's browser |
| Can | Fetch data, reach internal services, use secrets from `.env.local` | React to clicks and typing (`useState`, `onClick`) |
| Can't | React to clicks and typing | Keep secrets: its code is sent to the browser |

In the AgroPilot UI:

- `page.tsx` is a **server** component: it fetches the farms and the plots from `registry-service`.
- `plots/new/page.tsx` is a **server** component too: it fetches the farms and the catalogues for the dropdowns, and passes them to `PlotForm`.
- `PlotForm.tsx` is a **client** component: it reacts to typing and to Save.
- When you click Save, `PlotForm` (in the browser) calls **our** endpoint, `/api/plots`, and `route.ts` (on the server) calls `POST /v1/plots` on `registry-service`. This pattern is called **BFF** (Backend for Frontend). The browser never talks to `registry-service`.

Why not call `registry-service` straight from the browser? In production, the registry is an internal service. The browser can't even reach it, and later in the course the services will require an API key or a token. A key in the browser would be visible to every user. And browsers block requests to other addresses, unless that server allows it (**CORS**). You'll see that block in the exercises.

> [!WARNING]
> Variables that start with `NEXT_PUBLIC_` are copied into the JavaScript sent to the browser. Agents sometimes "fix" a client component by renaming a variable to `NEXT_PUBLIC_...`, or by adding CORS to the backend so the browser can call it directly. Never put a secret in `NEXT_PUBLIC_`, and don't let the agent move the registry calls into the browser.

## Component libraries

You don't need to reinvent the wheel and design each UI element by hand. There are many component libraries that you can use to build your app. Some of the most popular ones are:

- **shadcn/ui** is a library that copies each component into your project.
- **Material UI (MUI)** is a package you install. The look follows Google's Material Design.
- **Ant Design** is good for admin dashboards: tables, forms, and filters, which is most of what AgroPilot needs. Same idea as MUI: install the package, import the component.


# Exercises


### :pencil2: Build the AgroPilot UI

Work in your own AgroPilot repo (`ArgoPilot`), on top of your Task P1 work. The UI is one more service, in `services/ui`.

1. Install **Node.js** (the LTS version) from [https://nodejs.org/en/download](https://nodejs.org/en/download). Windows users: install it inside WSL. Check:

    ```bash
    node --version
    npm --version
    ```

2. Start `registry-service` in its own terminal, with a fresh database. Keep it running for all the exercises:

    ```bash
    cd services/registry-service
    bash seed_registry.sh
    uv run uvicorn main:app --host 127.0.0.1 --port 8000
    ```

3. In another terminal, create an empty Next.js app in `services/ui`:

    ```bash
    cd services
    npx create-next-app@latest ui --ts --tailwind --eslint --app --src-dir --use-npm --yes
    ```

4. From the repo root, start a new Claude Code session and send:

    > In `services/ui`, build a simple UI for registry-service (its API is at http://127.0.0.1:8000/docs). The home page lists every farm with its plots in a table. A "+ New plot" link opens a form to create a plot, with dropdowns for the farm, crop, soil, and emitter. The browser never calls registry-service: the form posts to the UI's own API route, and the Next.js server calls registry-service at `REGISTRY_URL` from `.env.local`. Add `.env.example`. When registry-service returns an error, show its `detail` to the user. Don't change registry-service. Run `npm run build` and `npm run lint` when you're done.

5. Run it:

    ```bash
    cd services/ui
    cp .env.example .env.local      # check that REGISTRY_URL is http://127.0.0.1:8000
    npm run dev
    ```

    Open [http://localhost:3000](http://localhost:3000). You should see the two seeded farms, *Demo cooperative* and *East farm*, with their four plots. Click **+ New plot** and add one.

6. Read what the agent built, and find in the code:
    - Which components are **server** components, and which are **client** components? How can you tell?
    - The UI's own API route (the **BFF**). Which registry endpoint does it call?
    - Where `REGISTRY_URL` is read. Does it ever reach the browser?
    - The `.next/` folder. Is it in `.gitignore`?

7. Commit `services/ui`. Check that `node_modules/`, `.next/`, and `.env.local` are **not** in the commit.

### :pencil2: The browser DevTools

The **DevTools** are the most useful tool for understanding and debugging any web app. In Chrome or Edge, right click on the page > **Inspect** (or `F12`).

1. Go to the tab where you see the HTML and CSS (**Elements**), and try to change some visual properties of the page (a farm name, a background color, a border, etc.). Take a look at how the changes are reflected in the page (they're saved only in your browser, until you refresh).
2. Open the **Network** tab, and refresh `/`. Do you see any request to port `8000`? Why not? Now add a plot. Click on the request the form sent, and take a look at the headers, the payload, and the response.
3. **Errors.** Add a plot with an area of `0`. In the Network tab, what's the status of that request? What's the `Content-Type` of the response? Do you recognize the body? What does the user see on the screen?
4. **Slow network.** In the Network tab, change **No throttling** to **Slow 4G**, and add a plot. What does the user see while waiting? Is it a good user experience?
5. **The registry is down.** Stop `registry-service` (`Ctrl+C`), and refresh `/`. What does the user see? Where is the real error, in the browser **Console** or in the **terminal** of `npm run dev`?
6. **CORS.** Start `registry-service` again. On `http://localhost:3000`, open the **Console** tab, and call the registry straight from the browser:

    ```js
    await fetch("http://127.0.0.1:8000/v1/farms").then((r) => r.json())
    ```

    What happens? Now run `curl http://127.0.0.1:8000/v1/farms` in a terminal. Why does `curl` work and the browser doesn't?
7. **Phone.** Click the **device toolbar** icon (a phone and a tablet), and pick a phone. A grower checks the plots in the field. Is the app usable?

When you report a UI bug to the agent, give it what you found here: the error from the Console or the terminal, the failed request from the Network tab.

Did you find something the user shouldn't see, like a blank page or a raw error? Ask the agent to fix it.



### :pencil2: Make it beautiful: with and without the `frontend-design` skill

Can you also tell when a UI was designed by an agent?
Agents, without guidance, all design the same way: a white page, a purple gradient, the same font, the same rounded cards. People call it **"AI slop"**.

Anthropic's great **`frontend-design`** skill teaches the agent to make deliberate design choices.

**Round 1: without the skill.** Branch `design-no-skill`. New Claude Code session (`/clear`):

> Redesign the AgroPilot UI in `services/ui`. Growers and agronomists use it to manage their farms and drip-irrigated plots, in the office and in the field on a phone. Make it look professional and pleasant to use, on desktop and on a phone. Don't change how it works, and don't change registry-service. Run `npm run build` and `npm run lint` when you're done.

Take screenshots, and commit.

**Round 2: with the skill.** `git switch main`, then branch `design-with-skill`. Install the skill:

```bash
npx skills add anthropics/skills --skill frontend-design -a claude-code
```

New session (`/clear`), the same prompt, with one more line at the beginning: *"Use the frontend-design skill."* Take screenshots, and commit.

Compare. Which would you give to the growers? Merge the one you prefer into `main`.

### :pencil2: Try component libraries

Try to use one of the component libraries mentioned in the previous section.

To revert to the original version, you can run `git restore .` (and `git clean -fd services/ui/` for new files).

Use this prompt, and change only the library name:

> Rebuild the AgroPilot UI in `services/ui` with **shadcn/ui | Material UI | Ant Design**. Keep every feature working. Don't change how the UI calls registry-service, and don't change registry-service. Run `npm run build` and `npm run lint`.



### :pencil2: Edit and delete plots

A grower made a typo in a plot name, or replaced the drippers on a block. Today the only way to fix it is `curl`.

The registry API already supports it: `PATCH /v1/plots/{id}` changes only the fields in the body, and `DELETE` removes a plot or a farm. No change to `registry-service` is needed, only to the UI.

1. New session (`/clear`):

    > Add Edit and Delete to every plot row in the AgroPilot UI. Edit opens the same form, filled with the plot's values, and saves with `PATCH /v1/plots/{id}`, sending only the fields that changed. Delete asks for confirmation, then calls `DELETE /v1/plots/{id}`. Add Delete to every farm too. The browser calls only the UI's own API routes, never registry-service. When registry-service returns an error, show its `detail` to the user. Don't change registry-service. Run `npm run build` and `npm run lint`.

2. Check it:
    - Change the emitter of *North block*, and refresh. Is the change still there? In the Network tab, which request did the browser send, and what's in its payload? Only the field that changed?
    - Rename *East block* to *North block*. They're on the same farm. What does the user see?
    - Delete *Demo cooperative*. It still has plots. What does the user see?
    - Check the registry itself:

        ```bash
        curl -s http://127.0.0.1:8000/v1/plots/c8f3d1b0-4d2e-5a1b-8b22-1d6e3f702002
        ```

        Do you see the new `emitter_id`?

3. Run `bash seed_registry.sh` to get the original data back.



### :pencil2: Accessibility

**Accessibility** (a11y) means that everyone can use the app: people who use a screen reader, people who can't use a mouse, people who see poorly or are color blind.

The common standard is **WCAG 2.2**, level **AA**. Agents don't take care of it unless you ask.

1. **Keyboard only.** Put the mouse aside. Use `Tab`, `Shift+Tab` and `Enter`. Can you open the form, pick a farm, a crop, a soil, and an emitter, save a plot, and see the error when the area is `0`? Can you always see where you are?
2. **Lighthouse.** DevTools > **Lighthouse** > select only **Accessibility** > **Analyze page load**. Run it on the home page and on the form. What's the score? What did it find?
3. **Fix.** Give the agent what Lighthouse found, and ask it to fix the problems and to make the app fully usable with the keyboard. Also: does a screen reader announce the registry's error when it appears?

Automated tools like Lighthouse find only part of the problems. Checking with the keyboard, done by a human, is still needed.
