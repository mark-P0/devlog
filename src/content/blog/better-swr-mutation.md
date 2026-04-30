---
title: "A Better Way to Handle SWR Mutations"
description: "Exploring co-location of mutations with its corresponding data fetches for a more disciplined and full-featured pattern"
pubDate: "April 30 2026"
# heroImage: ""
---

<!-- ## The case -->

<!-- ## What's the problem? -->

In a production codebase I contribute to, there is a frequent pattern where SWRs are revalidated separately from a remote mutation.

Consider the following:

```tsx
function MutateButton() {
  const swr1 = useDataSwr1();

  async function mutate() {
    await remoteMutation();

    // revalidates only, and manually
    await swr1.mutate();
  }

  return button({
    onClick: mutate,
  });
}

function DataComponent1() {
  const { data } = useDataSwr1();

  return render(data);
}
```

It works, but the `swr.mutate()` is used purely for revalidation. That is not inherently wrong. In fact, it is even the [officially endorsed](https://swr.vercel.app/docs/mutation#revalidation) way to revalidate fetches (or update the cache).

However, this approach does not use the SWR mutation interface to its _full potential_. There is a reason why it is not called `swr.revalidate()`. The docs even says it upfront:

> SWR provides the `mutate` ... APIs for **mutating remote data** ....
>
> — [SWR](https://swr.vercel.app/docs/mutation)

## What should be done?

With the pattern above, remote mutations are awaited independently, and then you will have to remember to revalidate any associated fetches with it. This is an _implicit coupling_ with human memory, which is **fallible**. Hopefully the stale data issue will be caught in a visual test, but by nature that is not guaranteed.

Most mutations are semantically related to a particular resource. That resource is likely fetched and shown on the webpage at the time of the mutation. It is then a better pattern to tie **that mutation directly to the fetch of that resource**.

```tsx
function BetterMutateButton() {
  const swr1 = useDataSwr1();

  function mutate() {
    // automatic revalidation whether remote mutation succeeds or fails
    swr1.mutate(
      async () => {
        await remoteMutation();
      },
      {
        optimisticData: () => {}, // can implement optimistic updates easily
        revalidate: true, // revalidate after remote mutation, might not be needed if optimistic update is setup and trusted
        populateCache: false, // should probably be set if remote mutation does not return updated data
      },
    );
  }

  return button({
    onClick: mutate,
  });
}
```

Instead of treating revalidation as a _side effect_ of a mutation, it should be treated as a **main requirement**. _What data will this mutation change?_. The mutation then must happen as a _function_ of that data fetch. This way we explicitly _declare_ the relationship between the two. The data fetch **owns** the mutation. Revalidation now will also happen automatically without regard to what happens with the remote mutation. No need to remember what should be revalidated.

<!-- instead of asking "what do i need to revalidate", we ask "what is being changed by this mutation" -->

This structure also gives us convenient access to the modern **optimistic updates** pattern. With it, immediate feedback can be shown on the webpage without necessarily waiting for the remote mutation to succeed. In case it fails, the optimistic update can just be rolled back, which is [already handled automatically by SWR](https://swr.vercel.app/docs/mutation#rollback-on-errors).

## What about mutations that must revalidate several SWRs?

In practice, this is very much a valid use case. There will be times where the updated resource is a dependency of another resource, and that other resource is fetched separately.

I have explored several approaches, but settled on the following:

```tsx
function MultipleRevalidationMutateButton() {
  const swr1 = useDataSwr1();
  const swr2 = useDataSwr2();
  const swr3 = useDataSwr3();

  function mutate() {
    // lift the mutation up, but do not await it (yet)
    const mutationPromise = remoteMutation();

    swr1.mutate(
      async () => {
        // await promise inside `swr.mutate` for automatic rollback of optimistic updates
        // will still have automatic revalidation
        await mutationPromise;
      },
      {
        // can still have these options
        optimisticData: () => {},
        revalidate: true,
        populateCache: false,
      },
    );

    swr2.mutate(
      async () => {
        // shared reference to promise can be awaited in several locations
        await mutationPromise;
      },
      {
        // can have separate definitions for these
        optimisticData: () => {},
        revalidate: true,
        populateCache: false,
      },
    );

    swr3.mutate(
      async () => {
        await mutationPromise;
      },
      // may not even need optimistic update for other SWRs
    );
  }

  return button({
    onClick: mutate,
  });
}
```

The key idea is here is to execute the remote mutation only **once**, then _share_ a reference to that execution among the data fetches that it affects.

Just like [state](https://react.dev/learn/sharing-state-between-components), we lift the mutation up, but not completely. We create a promise for the mutation and then await it within the relevant SWRs. This way the coupling between the SWRs is made **explicit**. Each can still have their own optimistic updates. Should the remote mutation fail, the SWRs can still rollback automatically because it will catch the rejected promise as it is awaited inside `swr.mutate()`.

> It is fine to reuse custom SWR hooks, even in places that do not primarily use `swr.data`. [The library handles de-duplication automatically](https://swr.vercel.app/docs/advanced/performance#deduplication), ensuring such multiple usage does not result in multiple async calls (e.g. network requests). Some may even argue that no `useSWR()` should be embedded _directly_ in any component...

<!-- > this is also where the separate SWR revalidation makes sense -->

## What's the catch?

As you may have noticed, this pattern is **incredibly verbose**. A simple 2-liner has bloated threefold. But often this is the trade-off between _doing something fast over doing it right_. The pattern does for us more than just simply automatic revalidation. It provides us a better mindset over the relationship between data fetches and mutations, and more control over the data cache. This may even give way to more advanced use cases like pre-feching client-side or [server-side](https://swr.vercel.app/docs/with-nextjs).

Unlocking the full features of SWR also demands from us a **deeper understanding** of it. We must familiarize ourselves with how revalidations work, when is the cache populated, or with optimistic updates, when rollbacks happen. In my opinion this is a good thing, as we should strive to master our tools anyway. Without this discipline, it will be easy to introduce accidental misconfigurations, as the pattern gives us more places to get things wrong.

For example:

```tsx
function mutate() {
  swr.mutate(
    async () => {
      await remoteMutation();

      // will implicitly return `undefined` here
    },
    {
      populateCache: false, // `swr.data` will become `undefined` after the mutation unless this is disabled
      // revalidate: true // this is already the default, so `swr` will revalidate after the mutation, but `swr.data` will still be `undefined` for a time
    },
  );
}
```

`swr.mutate(data)` will directly assign `data` as the value of `swr`'s key in the cache. `swr.mutate(asyncFn)` will assign the awaited return value of `asyncFn` instead. If `asyncFn` is not written to return anything, it will instead return `undefined`, and thus that will be assigned as the cache value. This gives the impression that `swr` is somehow **cleared or reset randomly**. To avoid this, the option `populateCache: false` can be set, so that `undefined` is not assigned to the cache.

If using a type checker, it will not even complain about this because it is type-safe; `swr.data` starts out as `undefined` while fetching.

## Why I still do this

Right now I call this pattern **co-located mutations**. The mutation is done at (or under) the same place that fetches the data it mutates. Because they are semantically related, they must evolve together as much as possible.

This is also related to another emerging personal preference of mine to _organize things by feature_ instead of what they are. For example, rather than creating directories for `components/` and `hooks/`, I would name directories by feature like `users/` and `posts/`. All components and hooks related to users will be placed in the former, and posts in the latter. In Next.js, you might even want to [co-locate this in the actual routes](https://nextjs.org/docs/app/getting-started/project-structure#colocation).

Lastly I will reiterate the following point:

> Revalidation is not a side effect of the mutation, it is part of the mutation itself.

We should not treat the data fetch as an afterthought. It must be front and center, because our [UI depends on the data](https://kn8.lt/blog/ui-is-a-function-of-data/).

<!-- This ties back to the idea that [the UI does not own the data, the server does](https://tkdodo.eu/blog/react-query-as-a-state-manager#a-data-synchronization-tool). -->

## Bonus: Independent mutations with `useSWRMutation()`

Sometimes a mutation truly is not (yet) part of a particular fetch. For example, when logging in, the request may mutate several database records, but there is no fetch to revalidate yet.

For such use cases I like to use `useSWRMutation()`. With this we lift the mutation up to its own _entity_. This gains us several things:

- **A managed lifecycle**. With `const mutation = useSWRMutation()`, we have access to `mutation.isSubmitting` and `mutation.error`. These are _indispensable_ to building a reactive UI around that mutation. No more manual `useState()` and try-catches.
- **Pre-population**. `useSWRMutation()` can place the mutation results in the cache, either via key association, or manually via `swr.mutate()` (yes, they can be used together!)
  - In the aforementioned login process, usually a separate `useSWR()` is defined for a user fetch. If the login mutation sends back the same shape as the user fetch, it can be used to pre-populate that SWR, providing a much snappier UX.

We can take our [revalidate multiple SWRs](#what-about-mutations-that-must-revalidate-several-swrs) example further with this:

```tsx
function MultipleRevalidateIndependentMutateButton() {
  const swr1 = useDataSwr1();
  const swr2 = useDataSwr2();
  const swr3 = useDataSwr3();

  const mutation = useSWRMutation("/mutation/key", async () => {
    // still shared promise
    const mutationPromise = remoteMutation();

    // `swr.mutate()` itself also returns as a promise!
    const swr1Promise = swr1.mutate(
      async () => {
        await mutationPromise;
      },
      {
        revalidate: true,
        populateCache: false,
      },
    );
    const swr2Promise = swr2.mutate(
      async () => {
        await mutationPromise;
      },
      {
        revalidate: true,
        populateCache: false,
      },
    );
    const swr3Promise = swr3.mutate(
      async () => {
        await mutationPromise;
      },
      {
        revalidate: true,
        populateCache: false,
      },
    );

    // await all the `swr.mutate()` calls here
    // since they all just await the remote mutation promise as well, the `useSWRMutation()` will still resolve with the remote mutation
    await Promise.all([swr1Promise, swr2Promise, swr3Promise]);
  });

  // use lifecycle properties for reactive UI
  return button({
    disabled: mutation.isSubmitting,
    onClick: () => mutation.trigger(),
    render: mutation.error ? "An error occurred" : "Click me!",
  });
}
```

> If really wanted, `useSWRMutation()` can be tied to a particular `useSWR()` fetch by sharing its key, possibly wrapping in [`unstable_serialize`](https://swr.vercel.app/docs/with-nextjs#complex-keys) for complex keys. However **I would discourage this**, as this would require maintaining a separate key definition, and just associating with `swr.mutate()` (as discussed above) is relatively more convenient.

---

<!-- ### mutations without data fetch -->

<!-- for pure mutations, `useSWRMutation` -->
