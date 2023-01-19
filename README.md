# useAsyncTask

useAsyncTask makes it easy to manage asynchronous calls in React Hooks. You know, like for calling APIs? That thing _everyone needs to do_?

It's a lightweight, one-file solution with zero dependencies, and it's been battle-tested for years.

For more on why I wrote this, see [the blog post](https://medium.com/@nfarina/precise-async-tasks-for-react-hooks-8daf8bd70858).

## Installation

```bash
npm install use-async-task
```

## Usage

Here's an example of using it with `fetch()`:

```js
import React from "react";
import { useAsyncTask } from "use-async-task";

function BookList() {
  const getBooks = useAsyncTask({
    async func() {
      const result = await fetch("https://www.anapioficeandfire.com/api/books");
      return await result.json();
    },
  });

  return (
    <>
      <ul>
        {getBooks.result?.map((book) => (
          <li key={book.url}>{book.name}</li>
        ))}
      </ul>
      <button onClick={getBooks.run}>Submit</button>
    </>
  );
}
```

## API

You can pass more (optional) arguments to `useAsyncTask()`:

```js
const getBooks = useAsyncTask({
  async func() {
    // do work, return result if needed
  },
  // Runs the task function (with no arguments) when
  // the component mounts.
  runOnMount: true,
  // Leaves the `isRunning` flag set to `true` after the
  // task function completes. This is useful if you want
  // to leave your UI in a "working" state while making it
  // animate away, for instance if it's in a modal.
  leaveRunning: true,
  // Called when the task function starts.
  onStart: () => {},
  // Called when the task function completes.
  onComplete: (result) => {},
  // Called when the task function throws an error.
  onError: (error) => {},
  // Called when the task function completes or throws an error.
  onFinally: () => {},
});
```

The `useAsyncTask()` hook returns an object with the following properties:

```js
{
  // Runs the task function, with optional arguments. In
  // TypeScript, the arguments will be typed as the
  // arguments of the task function.
  run: (...args) => void,
  // Cancels the task function (if it's running). This is always
  // safe to call, even if the task function isn't running.
  cancel: () => void,
  // Whether the task function is currently running.
  isRunning: boolean,
  // The result of the task function, if it completed. In
  // TypeScript, this will be typed as the return type of
  // the task function.
  result: T,
  // The error thrown by the task function, if it threw an error.
  error: Error,
}
```

## Handling Errors

In TypeScript, you must provide a value for the `onError` argument:

```ts
const getBooks = useAsyncTask({
  async func() {
    // do work
  },
  onError(error: Error) {
    // make sure the user sees this error!
  }
});
```

You can say `onError: null` to silence the warning if you are checking the `getBooks.error` value in render instead. But we wanted to make it very difficult for (usually rare) errors to be silently ignored because a developer forgot to handle them.

## Canceling Tasks

If you want to cancel a task, you can call `cancel()` on the task function. This will cause any result of the task function to be discarded, and the `onComplete()` and `onFinally()` callbacks will not be called.

You can also discover from within your task function whether it has been canceled by checking the special `this.isCanceled()` function:

```js
const getBooks = useAsyncTask({
  async func() {
    const result = await fetch("https://www.anapioficeandfire.com/api/books");

    // Were we canceled?
    if (this.isCanceled()) return;

    return await result.json();
  }
});
```

**NOTE**: For this to work, you need to define your task function like this:

```js
const getBooks = useAsyncTask({
  async func() {
    if (this.isCanceled()) { /* ... */ }
  }
});
```

If you define your task function as an arrow function, `this` will not work:

```js
const getBooks = useAsyncTask({
  func: async () => {
    // this.isCanceled() will not work here!
  }
});
```

The unusual technique of using `this` to access the task function's state was the most elegant way I could come up with, due to limitations of TypeScript argument types, closures, and the possibility of multiple task invocations. If you have a better idea, please let me know!

## Multiple Invocations

It is safe to call `run()` multiple times even before the task is finished. There is no way in JavaScript to "kill" a running task, so the task will continue to run in the background (or until you check `isCanceled()`). But the `onComplete()` and `onFinally()` callbacks will only be called for the most recent invocation.

In the rare instance that you need to know which invocation of the task function is currently running, you can check the `this.invocation` property from within the task function. This is a number that increments every time the task function is invoked.