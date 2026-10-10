# HACK THE DILLO

There is a live database. It holds one value. Only its owner can change it.

The owner is **bob@hackthedillo.dev**.

See if you can change the value.

Watch it live: https://ideas.29court.com/hack-the-dillo/

## The target

Read the live value right now, no auth needed:

```
curl -s -X POST https://hackthedillo.29court.com/v1/query/DilloValue
```

The page polls the database every 3 seconds. If the value ever changes, the page flips to **THE DILLO HAS BEEN HACKED**.

API notes (create-dillo 1.0.0):

- Reads go through `POST /v1/query/<Table>`. `GET /v1/tables/<Table>` without a row ID is a framework 404.
- The API host is `hackthedillo.29court.com`, a first-level subdomain. Cloudflare Universal SSL does not cover second-level subdomains, so the old `hackthedillo.dillo.29court.com` host is dead.

## The schema

```ts
export const schema = {
  tables: {
    DilloValue: {
      fields: {
        label: { type: "string", required: true },
        value: { type: "string", required: true },
      },
      permissions: {
        view: "anyone",   // the whole internet can read it
        edit: "nobody",   // nobody can write through the table API. nobody.
      },
    },
  },
};
```

Records live in create-dillo's JSON storage (`_armadillo_objects`).

One table. One row. Public reads. Writes are locked down to a single function:

```ts
export const backend = {
  functions: {
    // one-time setup. creates the owner and the first row.
    // refuses to run if a row already exists.
    seed: async () => { /* ... */ },

    // the ONLY way to change the value.
    // throws 403 unless you are signed in as bob@hackthedillo.dev.
    setValue: async (input: { value: string }, context) => {
      if (context.user?.email !== "bob@hackthedillo.dev") {
        throw new Error("403 FORBIDDEN: only the owner can change the value.");
      }
      /* ... */
    },
  },
};
```

## The rules

1. The value can only be changed by `setValue`, and `setValue` only works for `bob@hackthedillo.dev`.
2. Bob's password was generated at random and discarded. Nobody knows it. Nobody can sign in as Bob.
3. The table API itself allows no writes (`edit: "nobody"`).

So: the front door is welded shut. Find another way in, or prove there isn't one.

## What counts

Change the `value` field of the `DilloValue` row. The challenge page watches the database directly. The moment your write lands, the page flips to **THE DILLO HAS BEEN HACKED** and you are immortal.

Good luck. The dillo is waiting.

## This repo

- `index.html` + `assets/` — the exact live challenge page served at https://ideas.29court.com/hack-the-dillo/ (verified byte-identical with the production deploy).
- The API is a create-dillo deployment; its data lives server-side in JSON storage. This repo documents its public surface, not its internals.
