# stream-example

![Node.js](https://img.shields.io/badge/Node.js-%3E%3D10-339933?logo=node.js&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-ES2015%2B-F7DF1E?logo=javascript&logoColor=black)
![npm](https://img.shields.io/badge/npm-lockfile%20v1-CB3837?logo=npm&logoColor=white)
![nodemon](https://img.shields.io/badge/dev-nodemon-76D04B?logo=nodemon&logoColor=white)
![License: ISC](https://img.shields.io/badge/license-ISC-blue)
![CI](https://img.shields.io/badge/CI-not%20configured-lightgrey)

A study project that compares **asynchronous control-flow patterns in Node.js** by building the same small web spider several times.

Each implementation crawls the article listing pages of [antropogenez.ru](http://antropogenez.ru/articles/p/1), follows every article link, pulls out the article body, and saves it to disk. The crawling logic stays the same across implementations. Only the way the asynchronous work is ordered and limited changes: plain callbacks, the `async` library, Promises, generators with `co`, and custom task queues with a concurrency limit.

> **Each implementation lives on its own branch.** `master` contains only this README. Check out a branch to get the runnable code.

## Key features

- **One problem, nine branches.** Every branch solves the same crawl so you can compare them side by side.
- **Sequential vs. parallel execution**, including the trade-off between speed and overloading the target server.
- **Limited concurrency** with hand-written `TaskQueue`/`Queue` classes (callback-based and Promise-based).
- **Generator-based flow control** using `co` and `thunkify`, plus a hand-written `functionToPromise` promisifier.
- **Duplicate protection**: a `Map` of visited URLs avoids downloading the same page twice.
- **Two-level crawl**: listing pages are nesting level `1`, and the articles they link to are level `0` and get saved.

## Branches

| Branch | Approach | Concurrency | Author's notes |
| --- | --- | --- | --- |
| [`sequential-execution`](../../tree/sequential-execution) | Plain callbacks, recursive `iterate()` | One article at a time per page; listing pages start together | Fine while there are only a few pages. Goes round-robin: 1st article of each page, then 2nd article of each page, and so on. |
| [`parallel-execution`](../../tree/parallel-execution) | Callbacks + global `TaskQueue` | **2** downloads at a time | **Works best.** Goes page by page, at most two articles at once. |
| [`async`](../../tree/async) | [`async`](https://caolan.github.io/async/) library (`eachSeries`, `series`) | Fully sequential | Slow but stable. |
| [`promise`](../../tree/promise) | Promise chain (`promise = promise.then(...)`) | Sequential within a page | The server gets overloaded and responds with **503**. |
| [`parallel-promise`](../../tree/parallel-promise) | `Promise.all` | Unlimited | The server gets overloaded and responds with **503**. |
| [`parallel-promise-restriction`](../../tree/parallel-promise-restriction) | Promises + global Promise-based `Queue`, `functionToPromise` helper | **3** at a time | Works well. Goes page by page, three articles at a time. |
| [`sequential-generator`](../../tree/sequential-generator) | Generators + `co` + `thunkify` | Fully sequential | Works well. Goes page by page. |
| [`parallel-generator`](../../tree/parallel-generator) | Generators + `co`; `spiderLinks` returns a thunk that starts all article downloads at once | Unlimited within a page | Works well. Goes page by page. |
| `master` | No code | — | Landing branch with this README. |

The original notes on each branch are in Russian and are in that branch's `README.md`.

## Prerequisites

- **Node.js** and **npm**. The code was written in February 2019 for Node 10/11 with npm 6 (`package-lock.json` uses `lockfileVersion: 1`).
- **Git**, to switch between branches.
- Internet access to `http://antropogenez.ru` (plain HTTP).

### Node version compatibility

Dependencies install and load on current Node (tested with Node 24 / npm 11). Newer npm prints warnings because `request` is deprecated, and npm 11 skips nodemon's `postinstall` script. Neither one stops the app from running.

If you want the environment the code was written for, use [nvm](https://github.com/nvm-sh/nvm) (macOS/Linux) or [nvm-windows](https://github.com/coreybutler/nvm-windows):

```bash
nvm install 10
nvm use 10
```

## Running locally

```bash
# 1. Clone the repository
git clone https://github.com/ptitsynamv/stream-example.git
cd stream-example

# 2. Check out the implementation you want to run
git checkout parallel-execution        # or any branch from the table above

# 3. Install dependencies
npm install

# 4. Run the spider
npm start                              # runs `nodemon index`
# or, without restarting on file changes:
node index.js
```

Articles are saved as `articles/<article title>.txt`. The `articles/` folder is in `.gitignore`. Each file holds the inner HTML of the article's content block, not plain text.

`npm start` runs the script under **nodemon**, which restarts it whenever a file changes. Because the spider writes into `articles/` inside the project, use `node index.js` if you only want a single clean run.

### Configuration

There are no environment variables or config files. Everything is set at the top of `index.js`:

- **`urls`**: the listing pages to crawl (`http://antropogenez.ru/articles/p/1` … `/p/21`). Most are commented out by default. Uncomment more to crawl more pages.
- **Concurrency**: `new TaskQueue(2)` on `parallel-execution`, `new Queue(3)` on `parallel-promise-restriction`.
- **Nesting depth**: every entry point calls `spider(url, 1)`. Level `1` pages are parsed for links, and level `0` pages are saved.

### Things to know

- There are **no tests, linters, or build step**. `npm start` is the only script.
- Parsing depends on the target site's markup (`#content`, `.articles_items`, `.articles_list_subject`, `.tx-antropedia-pi1`). If the site has changed since 2019, the spider may find no links or crash while parsing.
- `mongoose`, `mkdir`, `fs` and `path` are listed in `package.json` but are either not used or are Node built-ins. They don't affect running the app.
- Article titles are used directly as file names. A title containing characters your filesystem doesn't allow (for example `:` or `?` on Windows) will cause that write to fail.

## Project structure

Every code branch has the same layout:

```text
stream-example/
├── index.js                     # The whole spider: URL list, HTML parsing, download/save, control flow
├── utilities/
│   └── functionToPromise.js     # Callback-to-Promise wrapper (parallel-promise-restriction only)
├── package.json                 # Dependencies and the `start` script
├── package-lock.json
├── README.md                    # Notes on this branch's approach (in Russian)
└── .gitignore                   # node_modules/, .idea/, articles/
```

`index.js` has the same building blocks on every branch. Only the control-flow functions change between branches:

| Function | Role |
| --- | --- |
| `getArticlesData(body)` | Parses a listing page into `{ date, subject, url }` entries |
| `getArticleText(body)` | Pulls the article body out of an article page |
| `saveFile` / `downloadArticle` | Fetch an article and write it to `articles/` (creating the folder with `mkdirp`) |
| `spider(linkData, nesting)` | Fetches a page, then saves it (`nesting === 0`) or passes it to `spiderLinks` |
| `spiderLinks(...)` | Crawls a listing page's links. **This is the function that differs between branches.** |
| `TaskQueue` / `Queue` | Concurrency limiters (`parallel-execution`, `parallel-promise-restriction`) |

### Dependencies

| Package | Used for |
| --- | --- |
| [`request`](https://www.npmjs.com/package/request) | HTTP requests (deprecated, but still installs) |
| [`dom-parser`](https://www.npmjs.com/package/dom-parser) | Lightweight HTML parsing |
| [`mkdirp`](https://www.npmjs.com/package/mkdirp) `0.5.x` | Creating the output folder (callback API) |
| [`async`](https://www.npmjs.com/package/async) | Flow control (`async` branch) |
| [`co`](https://www.npmjs.com/package/co) + [`thunkify`](https://www.npmjs.com/package/thunkify) | Generator-based flow control (`*-generator` branches) |
| [`nodemon`](https://www.npmjs.com/package/nodemon) | Dev runner for `npm start` |

## License

`package.json` declares the **ISC** license. The repository does not include a `LICENSE` file.
