# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This is a Node.js study project. Each async control-flow technique is a **separate branch**, and `master` holds only docs. To work on code, check out a branch first (`sequential-execution`, `parallel-execution`, `async`, `promise`, `parallel-promise`, `parallel-promise-restriction`, `sequential-generator`, `parallel-generator`). `README.md` on master has a table comparing them.

## Commands (on a code branch)

- `npm install`, then `npm start` (runs `nodemon index`) or `node index.js` for a single run.
- There are no tests, linting or build step.

## Architecture

Each branch has a single `index.js` (plus `utilities/functionToPromise.js` on `parallel-promise-restriction`) with the same pipeline: the `urls` listing pages → `spider(url, 1)` → `spiderLinks` parses links with `getArticlesData` → `spider(link, 0)` → `downloadArticle`/`saveFile` writes `articles/<subject>.txt`. A `spideringMap` prevents duplicate fetches. Only `spider`/`spiderLinks` and the queue classes differ between branches. Keep the shared helpers identical when changing one branch so the branches stay comparable.
