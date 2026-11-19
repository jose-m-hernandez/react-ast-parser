# react-ast-parser

A CLI that maps React component dependencies from the Babel AST and draws them as Graphviz graphs. It is for someone reading an unfamiliar React codebase who wants the component tree and the Redux wiring in one picture.

## What it does

- Starts from one component file and follows its relative imports to the child components it renders, recursively.
- Reports, per component, the state it declares (class `state`, `useState`, `useReducer`), the Redux store keys it reads, the Redux actions it binds, its children, and the props it passes to each child.
- Prints a text tree and JSON of that data, and writes a Graphviz graph (`graph.dot`, plus `graph.svg` when Graphviz is installed). With `--cypher` it also writes a Cypher script for Neo4j (see "Graph database export").

## Requirements

- Node.js. It was tested on the current LTS (`.nvmrc`); the test suite needs Node 18 or newer.
- Graphviz, only for the SVG. Without it the run still writes `graph.dot`, prints how to install Graphviz and exits 0. Downloads: https://graphviz.org/download/ (macOS `brew install graphviz`, Ubuntu/Debian `sudo apt-get install graphviz`, Windows `choco install graphviz`).

## Install

The package is not published to npm. Install it from a clone:

```bash
git clone https://github.com/jose-m-hernandez/react-ast-parser.git
cd react-ast-parser
npm install
npm install -g .
```

Or run it from the clone without installing:
```bash
node main.js ./path/to/component.jsx
```

## Run

From a clone, run the bundled fixture:

```bash
node main.js test/fixtures/tree/App.jsx
```

It prints a tree of the components it found:

```
Analyzing component structure...

- App
  -> TodoList
  -> Counter
  - TodoList
    -> TodoItem
    - TodoItem
  - Counter
```

then the component data as JSON, of the form `{"components": [...]}`, where each component has the fields `name`, `state`, `reduxProps`, `actions`, `children`, `props`, `importMap` and `filePath`. Last it prints `Generating graph...` and writes `graph.dot` and `graph.svg`.

`graph.dot` and `graph.svg` (and `graph.cypher` with `--cypher`) are written into the current directory and overwrite files of the same name.

## Options and exit codes

```bash
node main.js <component-file> [options]
```

| Option | Effect |
| --- | --- |
| `--debug` | Show detailed parsing information, including how each import is resolved. |
| `--quiet` | Suppress console output during analysis. |
| `--graph-only` | Generate only the graph; skip the JSON. |
| `--json-only` | Print only the JSON; skip graph generation. |
| `--cypher` | Also write `graph.cypher`, a Cypher script for Neo4j (see "Graph database export"). |

Exit status: 0 on success; 1 for a usage error or a missing input file; 2 when any file failed to parse (all output is still written).

## What the graph shows

- Each component is a rounded box with a blue background, its name, and a `State:` line listing its state (`None` when it has none).
- A grey arrow runs from a component to each child it renders, labelled with the props passed to that child.
- A node named "Redux Store" is joined to components by orange arrows: store to component, labelled with a store key the component reads; component to store, labelled with an action it binds.
- A file that failed to parse is a pink box that begins `Error:`.

What is recorded as Redux data: `connect()` gives store keys (from `mapStateToProps`) and actions (from `mapDispatchToProps`); `useSelector` gives the store path it reads. `useDispatch` is not recorded, so actions dispatched through that hook do not appear. State comes from class `state`, `this.state` in a constructor, `useState` and `useReducer`.

## Graph database export

`--cypher` writes `graph.cypher` into the working directory, next to `graph.dot`,
in addition to whatever `--json-only` or `--graph-only` select. With `--quiet`
it prints nothing new; otherwise it prints `✓ Generated graph.cypher`. The file
is a plain Cypher script (Neo4j 5 syntax). The tool needs no Neo4j driver and no
running server.

| Kind | Label or type | Properties |
| --- | --- | --- |
| Node | `Component` | `name`, `file` (relative to the directory of the analyzed file, forward slashes; absent for a child that never resolved), `parseError` (boolean), `state` (list; omitted when none) |
| Node | `StoreKey` | `name`, one per Redux prop key |
| Node | `Action` | `name`, one per action name |
| Relationship | `(Component)-[:RENDERS]->(Component)` | `props` (list, may be empty) |
| Relationship | `(Component)-[:SELECTS]->(StoreKey)` | none |
| Relationship | `(Component)-[:DISPATCHES]->(Action)` | none |

The script creates a uniqueness constraint on `name` for each label
(`IF NOT EXISTS`), then uses `MERGE` for every node and relationship, so loading
it twice gives the same graph. Output is sorted and byte-identical for the same input.

The script
only adds and updates (it is `MERGE`-only), so components, edges and properties
that you have since removed from the source **persist** after a re-load. The one
exception is a property of a node that still exists: when a Component's `state`
is null or its `file` is absent, its statement ends in `REMOVE n.state` or
`REMOVE n.file`, so a re-load clears a stale value. To mirror a changed
codebase exactly, clear the database before loading (for example
`MATCH (n) DETACH DELETE n;`, or start from an empty database).

Load it with:

```bash
cypher-shell -u <user> -p <password> -f graph.cypher
```

or paste it into Neo4j Browser. For example, the components that select the
store key `user`:

```cypher
MATCH (c:Component)-[:SELECTS]->(:StoreKey {name: 'user'}) RETURN c.name, c.file;
```

**Known limit:** a Component is keyed by `name`, the same key the DOT output
uses. Two components with the same name in different files merge into one node
(the first one found supplies `file` and `state`).

The tests check the generated script's text and shape.

### Live Neo4j check (opt-in)

`test/neo4j.live.test.js` runs the script against a real Neo4j.
It is skipped unless `NEO4J_URL` is set, so `npm test` makes no network call.
Set `NEO4J_URL` (for example `http://localhost:7474`) and `NEO4J_PASSWORD`; `NEO4J_USER` (default `neo4j`) and `NEO4J_DATABASE` (default `neo4j`) are optional.
It needs an empty database and refuses to run otherwise; afterwards it deletes every node and drops only the constraints it created.
It loads the `tree` fixture's script twice, then checks node and relationship counts and sample queries, and that the second load changed nothing.
It was run against Neo4j Community 2026.09.0 on a local machine on 2026-10-22.

## What it does not do

- It follows only imports whose path resolves, relative to the importing file, to a file (extensions `.jsx`, `.js`, `.tsx`, `.ts` are tried) or to a directory with an `index` file of those types. Imports of npm packages are not followed.
- Path aliases (for example `@/components`) are not resolved.
- Dynamic `import()` and `require()` are not followed.
- A child whose import does not resolve is not analysed: it has no entry in the JSON `components` list and no state or children, though its parent still lists it and the graph still draws the arrow to it. The run prints a warning naming it.
- Components are keyed by name, so two components with the same name in different files merge.
- Only the Babel `jsx` and `typescript` plugins are enabled, so Flow syntax does not parse.

## Troubleshooting

- `Graphviz is not installed`: install Graphviz (see Requirements). `graph.dot` was still written; `dot -Tsvg graph.dot -o graph.svg` converts it later.
- `Circular import detected: <file>`: a file imports, directly or through others, a file already visited. It is parsed once and the cycle is not followed again.
- `Error parsing file <path>` and `Parse error in <file>`: Babel could not parse that file. It appears as a pink box in the graph and the run exits 2; every other file is still processed.
- `Could not resolve component '<Name>'`: the import for that JSX element did not resolve to a file (see "What it does not do"). Run with `--debug` to see how each import was resolved and which imports were available.

## Tests

After `npm install`, `npm test` runs the `node --test` suite in `test/`. It needs no Graphviz and makes no network call.
One test, the live Neo4j check described above, is skipped unless `NEO4J_URL` is set.

## License

ISC. See [LICENSE](LICENSE).
