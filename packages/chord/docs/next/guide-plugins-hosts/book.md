---
title: Hosts and Plugins: a tutorial for Chord v2
cover: Hosts / and Plugins
kicker: A guide · Chord v2 · 2026-10-31
subtitle: Build a small notes app step by step, from one host to plugins on a server and on workers, edited while it runs
stats:
  - 3 | kinds of host in an app: server, worker, client
  - 1 | file per host kind in a plugin
  - 0 | restarts to apply a plugin edit
  - 1 | rule: plugins talk only through object types and registries
note: |
  A tutorial for people building apps on Chord v2, written by the redesign session from `design.md`, `slice-7c.md`,
  `slice-10.md`, the decision log and the code in `packages/chord/src/next` and `src/next-node`. API names are the
  current ones; where slice 10's fix round or the app kit (`chord/next/app`, being designed) will change what app code
  writes, a note says so.
colophon: Hosts and Plugins · first edition · 2026-10-31 · Chord v2 slices 1-10
---

# Before we begin {.preface}

You want to build a notes app. You type a note in a browser tab, and every other tab sees it at once. Each project
gets its own notes in its own process, so a project that misbehaves cannot take the others down. And you, or an agent
you work with, add features while the app runs: a search, an export button, a word count, without restarting
anything.

Chord gives you three things for that. **Hosts**: the processes of your app (the server, a worker per project, the
browser tabs), connected to each other. **Objects**: the state and methods your app is made of, living on one host and
used from any other. **Plugins**: directories of code that declare objects for each kind of host, loaded and
reloaded while the app runs.

This book builds the app in that order. Each chapter adds one concept and shows complete code.

## Seven things to know

:::findings
1. **A host is one participant.** The server, each worker and each browser tab is a host; hosts connect over links.
2. **An object has a type, and a type is a contract.** The contract names the object's state and methods; any host can
   ask for an object by type and params, and Chord finds where it lives.
3. **A facet declares, an object acts.** A facet is a plugin's code for one host kind: it says which object types this
   host serves. The objects hold the state and do the work.
4. **A plugin is a directory.** A `package.json` with the plugin's name, and one file per host kind: `server.ts`,
   `worker.ts`, `web.tsx`.
5. **The server builds, every host evaluates.** The plugins component builds each facet into a small script; the
   server, workers and tabs load the scripts for their kind.
6. **Plugins meet through types and registries.** One plugin uses another's objects through its contract, or
   contributes to a registry another defines. Nothing else crosses between plugins.
7. **An edit is a rollout.** Change a file, build, roll out: the server, then every worker, swap the plugin's objects in
   place; a broken edit keeps the old code running and reports why.
:::

## Words used throughout

A **host** is one participant of the app, made with `createHost`. A **host kind** (`Server`, `Session`) describes
addressable hosts; an **address** (`Session({ id: "p1" })`) names one. An **object type** is a contract: `local` (used
on its own host only) or `remote` (used from any host). An **object** is one instance of a type, per params. A
**facet** is a plugin's declarations for one host kind. A **plugin** is a directory with a `package.json`. A
**reference** is what you get when you ask for an object (`host.get`, `self.get`, `ctx.get`). A **registry** is an
extension point: one plugin defines it, others contribute. A **rollout** applies a build to the hosts in order.

# One Host {#host lede="The smallest app: one process, one object type, one call."}

Start without plugins. One host, one object that keeps notes:

```ts
import { createHost, defineFacet, defineState, local, method, obj, arr, str } from "@earendil-works/chord/next";

// the contract: what a Notes object looks like from outside
const Notes = local("notes.store", {
	version: 1,
	params: {},                                        // one object per host: no params
	members: {
		list: obj({ items: arr(str()) }),             // state anyone can read and watch
		add: method(obj({ text: str() })),            // a method: arguments checked by Chord
	},
});

// the document the object keeps
const NotesDoc = defineState("notes", { schema: obj({ items: arr(str()) }), version: 1, initial: () => ({ items: [] }) });

// the facet: which types this host serves, and how to make each object
const facet = defineFacet((host) => {
	host.serve(Notes, (_params, self) => {
		const doc = self.state(NotesDoc);
		return {
			list: doc,
			add: async ({ text }, ctx) => self.tx(async (tx) => { (await tx.draft(doc)).items.push(text); }, ctx),
		};
	});
});

const server = createHost({ onError: console.error });
await server.load([{ name: "@app/notes", facet }], ctx);
const notes = await server.get(Notes, {}, ctx);        // a reference: live, with its state
await notes.add({ text: "buy milk" }, ctx);
console.log(notes.list.value);                          // { items: ["buy milk"] }
```

Four things happened. The **contract** `Notes` says what the object offers. The **facet** says this host serves
`Notes`, with a **factory** that declares the object's parts: a document, and a method writing it in a transaction.
`load` publishes the facet. `get` returns a **reference** once the object is live; its members are the contract's.

The factory only declares: it is synchronous and opens nothing. Chord activates the object (loads its documents, opens
what it asked to open) and then serves calls. That is why the same factory can run again when the code changes.

# Several Hosts {#hosts lede="A server, a worker per project, browser tabs: how they find each other's objects."}

Now give each project its own process. Declare the kinds of addressable host, and make notes a `remote` type that
lives on its project's worker:

```ts
export const Server = hostKind("server");
export const Project = hostKind("project", { id: str() });

export const Notes = remote("notes", {
	version: 1,
	params: { project: str() },
	hostAddress: ({ project }) => Project({ id: project }),   // where an object of these params lives
	variants: { editor: { state: { list: NotesSchema }, methods: { add: method(AddArgs) } } },
});
```

`hostAddress` is the whole routing story. A tab asks for `Notes { project: "p1" }`. Its host computes the address,
`Project(p1)`, and sends the request to its parent, the server. The server has a **route** for `Project`: it starts (or
finds) the worker for `p1` and passes the request on. The worker serves `Notes`. The tab gets a reference whose state
follows the worker's document, one frame per change.

:::flow wide animate | A tab holds a project's notes: the server routes by address to the project's worker and relays, without reading the data.
row "Clients" blue
box tab1 "tab A" "holds Notes { project: p1 }"
box tab2 "tab B" "holds Notes { project: p1 }"
row "Server" accent
box server "server" "route Project → worker"
row "Workers" muted
box w1 "worker p1" "serves Notes { p1 }"
box w2 "worker p2" "serves Notes { p2 }"
arrow tab1 -> server "hold"
arrow tab2 -> server "hold"
arrow server -> w1 "relay"
:::

Each host is created the same way, with what it is:

```ts
// server.ts
const server = createHost<P>({ address: Server(), principal: { kind: "server" }, onError });

// worker.ts (started by the server for one project)
const worker = createHost<P>({ address: Project({ id }), principal: { kind: "worker", project: id }, onError });
worker.connect(parentLink, { principal: { kind: "server" }, trusted: true }, ctx);

// a browser tab: no address, it is a client
const tab = createHost<P>({ onError });
tab.connect(webSocketLink(new WebSocket("/chord")), { principal: { kind: "server" } }, ctx);
```

`P` is your app's type of **principal**, who is calling: an owner, a guest, the server, a worker. Links stamp it;
a `remote` object is made per principal, so a guest can get a read-only variant of the same notes.

:::note | Coming with the app kit
Starting workers, routing a host kind to them, the WebSocket entry and authentication are slice 10's components today
(`createWorkers`, `chord.http`), wired by the app. The opinionated app kit (`chord/next/app`, being designed) will do
this wiring for you: `startApp(...)` and `startWorker()`.
:::

# Plugins {#plugins lede="Move the code into a directory that can be added, edited and removed while the app runs."}

So far the facets were part of the app's own code. A **plugin** is the same code in its own directory:

```text
plugins/notes/
  package.json        { "name": "@app/notes" }
  contract.ts         the Notes contract (other plugins import it)
  worker.ts           the facet for workers: serves Notes
  server.ts           the facet for the server: an export route
  web.tsx             the facet for browser tabs: a notes panel
```

The plugin's name is its `package.json` name. Each file named after a **host kind** is that kind's facet, its
default export a `defineFacet(...)`. A plugin has only the files it needs: a plugin that only adds a route has a
`server.ts` and nothing else.

```ts
// plugins/notes/worker.ts
import { defineFacet } from "@app/core";
import { Notes } from "./contract.ts";

export default defineFacet((host) => {
	host.serve(Notes, ({ project }, principal, self) => {
		const doc = self.state(NotesDoc);
		return { variant: "editor", members: { list: doc, add: async ({ text }, ctx) => /* … */ } };
	});
});
```

`@app/core` is your app's **base code**, imported by every plugin as a host package (the plugin's build leaves it
out; every host passes its own copy in). It holds what plugins share and never reloads: host kinds, the principal
type, a `defineFacet` bound to it.

# Loading Plugins {#loading lede="Discover, build, evaluate, load: what happens between a directory and a running object."}

The **plugins component** does the work. You tell it where plugins are and what each host kind needs:

```ts
const plugins = createPlugins({
	roots: ["./plugins"],
	kinds: {
		server: { platform: "node", hostPackages: ["@earendil-works/chord/next", "@app/core"] },
		worker: { platform: "node", hostPackages: ["@earendil-works/chord/next", "@app/core"] },
		web: { platform: "browser", hostPackages: ["@earendil-works/chord/next", "@app/core", "react", "react/jsx-runtime"] },
	},
	onError,
}, ctx);

const report = await plugins.load(server, {
	kind: "server",
	core: [{ name: "@app/core", facet: coreServerFacet }],
	modules: { "@earendil-works/chord/next": chord, "@app/core": core },
}, ctx);
console.log(report.failed);   // plugins left out, with why: [] when all loaded
```

:::steps | What `plugins.load` does
+++ Discover
It lists `./plugins` one level deep and reads each `package.json`. A directory without one, or with two `server`
files, is reported, not loaded.
+++ Build
For each plugin with a `server` facet, esbuild builds one script from `server.ts` and everything it imports: the
plugin's own files, another plugin's contract files, its npm dependencies. Host packages stay out. A syntax error is a
build failure, never a crash.
+++ Evaluate
The script runs and returns the facet. Its `import "@app/core"` gets the server's own `@app/core`, so a `ChordError`
from a plugin is the same class as the server's.
+++ Load
`host.load` runs every facet's setup and publishes all of them at once. A plugin whose build, evaluation or setup
fails is left out and listed in `report.failed`; the others load. Its objects' users wait, and the next edit loads it.
:::

A worker never builds: the server builds the worker facets too, and each worker evaluates the scripts it is sent.

# Plugins Using Plugins {#together lede="Two ways one plugin uses another, and the one it may not."}

A search plugin wants to search the notes. It **imports the contract** and asks for the object:

```ts
// plugins/search/worker.ts
import { Notes } from "../notes/contract.ts";   // copied into search's script: contracts only

host.serve(Search, ({ project }, principal, self) => {
	const notes = self.get(Notes, { project });   // a dependency: Search goes live once Notes is
	const index = self.derive([notes.list], ([list]) => buildIndex(list.items));
	return { variant: "main", members: { find: async ({ q }) => lookup(index.value, q) } };
});
```

`self.get` in a factory declares a dependency: Chord makes `Search` live only once `Notes` is, and if the notes
plugin is missing, `Search` waits with a clear reason ("waiting for notes: not served here") instead of failing.

The other way is a **registry**: an extension point one plugin defines and others contribute to. The app's core
defines `Panels`, and every plugin with a UI contributes one:

```tsx
// plugins/notes/web.tsx
export default defineFacet((host, self) => self.contribute(Panels, { title: "Notes", Component: NotesPanel }));
```

What a plugin may not do is reach into another's data. Search never reads notes' documents; it uses the `Notes`
object's state. That rule is what lets the notes plugin change its storage without breaking search.

# Changing a Plugin {#reload lede="Edit, build, roll out: how new code takes over while every tab stays connected."}

You change the notes plugin: notes now have a timestamp. The watcher sees the saved file and marks the plugin dirty.
You (or an agent) build and roll out:

```ts
const built = await plugins.build(plugins.state.value.dirty, ctx);
const result = await plugins.rollout([plugins.step("server", server), workers.step("workers")], built, ctx);
```

The rollout goes host by host, server first, then every running worker. On each host Chord compares the new plugin
set with the loaded one, runs the new factories against a recording `self` to see what each object would declare, and
then gives every object of the changed plugin its fate:

| The object | What happens |
| --- | --- |
| holds no natives (open files, sockets), the new code neither | **swapped** in place: new methods, same documents; running calls finish on the old code |
| holds natives | **restarted**: calls wait, running ones finish, natives close and reopen with the new code |
| its type was removed | **ended**; its users wait until something serves it again |

The tab's reference follows the new object and gets one frame with the difference. Nothing reconnects. If the edit
is broken (a syntax error, a factory that throws), that plugin keeps its old code on that host and the report says why;
every other plugin of the rollout applies.

:::note | Versions
A change to a contract (a new field in `Notes`'s state) gets a new `version`. A tab built with the old contract keeps
working if the provider maps the old version (`accepts`); otherwise it waits with "version mismatch" until it reloads.
:::

# Base Code and Plugins {.epilogue lede="What reloads, what restarts, and where the line is."}

Everything in `plugins/` reloads: edit, build, roll out. Everything in your base code (`src/`, `@app/core`: host
kinds, principals, the core facet, the app's startup) changes only with a restart, and the watcher tells you so.
Keep base code small: host kinds, principals, and what every plugin needs. Put features in plugins.

The app kit being designed next (`chord/next/app`) will make the base code nearly empty: `startApp` and `startWorker`
with Chord's conventions for hosts, users, authentication and storage, so an app is mostly plugins.

# Reference {.appendix}

| Call | What it does |
| --- | --- |
| `createHost<P>({ address?, principal?, store?, onError })` | a host: addressable with an address, else a client |
| `host.load([{ name, facet }], ctx)` / `host.reload(...)` | publish facets / apply a new plugin set |
| `host.get(Type, params, ctx)` / `host.hold(...)` | a live reference / a reference at once that follows its object |
| `local(name, { version, params, members })` / `remote(name, { version, params, hostAddress, variants })` | contracts |
| `defineFacet((host, self) => …)`, `host.serve(Type, factory)`, `self.contribute(R, value)` | a facet |
| factory `self`: `state`, `derive`, `get`, `registry`, `open`, `on`, `onLive`, `job`, `tx` | an object's parts |
| `createPlugins({ roots, kinds, onError }, ctx)` | the plugins component |
| `plugins.load(host, { kind, core, modules }, ctx)`, `build(names)`, `rollout(steps, built)` | load, build, roll out |
| `hostKind(name, params?)`, `Kind(params)` | host kinds and addresses |
