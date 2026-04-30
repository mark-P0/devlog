---
title: "A Better Way to Handle SWR Mutations"
description: ""
pubDate: "April 30 2026"
# heroImage: ""
---

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

It works, but the SWR mutate is only used revalidation. That is not a problem per se. In fact, it is even the [officially endorsed](https://swr.vercel.app/docs/mutation#revalidation) way to revalidate fetches (or update the cache).

However in my opinion, this approach does not use the SWR mutation interface to its **full potential**. There is a reason why it is not called `swr.revalidate()`. The docs even says it upfront:

> SWR provides the `mutate` ... APIs for **mutating remote data** ....

## What should be done?

With the pattern above, remote mutations are awaited independently, and then you will have to remember to revalidate any associated fetches with it. This relies on human memory, which is fallible. Chances are the stale data issue will be catched by a visual test, but by nature even that is not guaranteed.

Most of the time, mutations are semantically related to a particular resource. That resource is likely fetched and shown on the webpage at the time of the mutation. It is then a better pattern to tie that mutation directly to the fetch of that resource.

```tsx
function BetterMutateButton() {
  const swr1 = useDataSwr1();

  function mutate() {
    // automatic revalidation in case remote mutation fails
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

Instead of thinking of revalidation as a _side effect_ of a mutation, we must think of it as a **main requirement**. _What data will this mutation change?_. The mutation then must happen as a "function" of that data fetch (SWR). This way we explicitly "declare" the relationship between the two. Revalidation now will also happen automatically after the remote mutation succeeds. No need to remember what should be revalidated.

This structure also gives us convenient access to the modern **optimistic updates** pattern. With it, immediate feedback can be shown on the webpage without necessarily waiting for the remote mutation to succeed. In case it fails, the optimistic update can just be rolled back, which is [already handled automatically by SWR](https://swr.vercel.app/docs/mutation#rollback-on-errors).

## What about mutations that must revalidate several SWRs?

Despite what I said above, this is still a valid use case. There might be times where the updated resource is a dependency of another resource, and that other resource is fetched separately.

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
      // may not even need optimistic opdate for other SWRs
    );
  }

  return button({
    onClick: mutate,
  });
}
```

Just like [state](https://react.dev/learn/sharing-state-between-components), we lift the mutation up, but not completely. We create a promise for the mutation and share it among the SWRs. This way the coupling between the SWRs is clearly defined. Each can still have their own optimistic updates.
