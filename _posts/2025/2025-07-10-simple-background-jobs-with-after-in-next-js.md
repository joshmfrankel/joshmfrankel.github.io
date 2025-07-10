---
layout: post
title: Simple Background Jobs with After in Next.js
categories:
  - today-i-learned
tags:
  - nextjs
  - javascript
---

Today I learned about the `after` function for scheduling side effects which avoid blocking execution. Think of it as a simple background job scheduler. These are much lighter than a database or Redis backed queueing infrastructure. That being said, I'm still learning what the benefits vs. costs might be for this Next.js version 15 update.

<!--excerpt-->

> after allows you to schedule work to be executed after a response (or prerender) is finished. This is useful for tasks and other side effects that should not block the response, such as logging and analytics.
>
> - [Next.js documentation](https://nextjs.org/docs/app/api-reference/functions/after)

`after` runs after all other logic including when there is failure case. This means you can rely on it for
things like tracking.

```js
import { generateMatches } from "@/services/generate-matches-service";

export async function handleSubmit(
  prevState: any,
  formData: FormData
): Promise<SubmissionResponse> {
  try {
    const createdSearch = await prisma.search.create({ data: { ...formData } });

    // Runs after all other logic including the redirect below
    after(() => {
      generateMatches(createdSearch)
        .then((matches) => {
          console.log("Matches generated", matches);
        })
        .catch((error) => {
          console.error("Error generating matches", error);
        });
    });

    redirect(`/searches/${createdSearch.id}`);
  } catch (error) {
    console.log(error);
  }
}
```

Have you used `after()` yet? What has gone well when using it? What could be improved? Start the conversation below.
