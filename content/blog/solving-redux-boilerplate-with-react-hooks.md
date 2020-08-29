---
title       : "My recent bout with React/Redux boilerplate"
date        : 2020-08-29T20:17:01+05:30
description : "Creating a custom react-hook to ergonomically perform asnychronous operations in a React app without having to deal with tons of Redux boilderplate"
feature_image: "/images/blog/quick-sand-of-redux-1280.png"
image: "/images/blog/quick-sand-of-redux-1280.png"
author: Charanjit Singh
---

With rise of GraphQL, we often have to work with React Apollo. I personally
don't like apollo for reasons I'll refrain from discussing in this post. But I
absolutely loved its `useQuery` and `useMutation` react-hooks.

When building [SoS App](https://github.com/sosinc/sos-app/), we chose Redux for state management because:

1.  It gives a greater degree of control and predictability over global state
2.  Its [debugging tools](https://addons.mozilla.org/en-US/firefox/addon/reduxdevtools/) are very helpful

But Redux does not come without problems. My biggest beef with Redux is

### The Boilerplate Monster

Redux is too verbose. It causes problems, from making learning hard for
beginners, to significant loss of productivity. Even causing serious accumulated
fatigue over time. You don't get a good return for your investment.

Internet is full of different approaches to solve this.


### @redux/toolkit

[@redux/toolkit](https://redux-starter-kit.js.org/) provides utilities to fight off the boilerplate monster. We make
extensive use of many of them. But I wanted more. A friction-free way to perform
async actions, with as few as possible compromises with Redux principles.


### Custom react hooks

React hooks provide a great way of creating modular abstractions which can be
beautifully composed together. They give me the warm fuzzy feeling of using
Haskell Type Classes.

We chose to create our own version of `useQuery`, that

1.  interoperates with @redux/toolkit's [createAsyncThunk](https://redux-toolkit.js.org/api/createAsyncThunk)
2.  optimized ease of use with minimal compromises of redux principles


### A Compromise with "The Redux Way"&reg;

We chose to keep state of async operation (i.e loading, success, and failure)
locally inside the component. Reason being that most of the times, we don't care
about the lifecycle of async operation globally, just the results.

e.g when fetching list of employees, this is how we want to handle its state transitions:

1.  `Loading`: show the skeleton in `EmployeesList`
2.  `Error`: show error notification
3.  `Success`: store the employees in global state; and show success notification

Keeping this in mind, we came up with this react-hook:

```typescript

    const useAsyncThunk = (
      asyncThunk: (args?: any) => AsyncThunkAction<any, any, any>,
      options: Options = {},
    ) => {
      const [isFetching, setIsFetching] = useState<boolean>(false);
      const [flash] = useFlash();
      const dispatch = useDispatch();

      const runAsyncThunk = useCallback(async (args?: any) => {
        try {
          setIsFetching(true);
          return await unwrapResult((await dispatch(asyncThunk(args))) as any);
        } catch (err) {
          throw err;
        } finally {
          setIsFetching(false);
        }
      }, []);

      return [runAsyncThunk, isFetching] as [(arg?: any) => Promise<void>, boolean];
    };

```

This tiny little hook allow us to do this in components;

```typescript

    const [createEmployee, isCreatingEmployee] = useAsyncThunk(createEmployeeAction);

```

Ergonomic much!? While we are doing this, Redux is still getting all the right
actions. [createEmployeeAction](https://github.com/sosinc/sos-app/blob/c326ef651e1f5653a1e9bc279d550dadd2f5aa20/ui/src/duck/employees.ts#L35) here is just a simple [async-thunk](https://redux-starter-kit.js.org/api/createAsyncThunk) wrapping a [plain
TS function](https://github.com/sosinc/sos-app/blob/c326ef651e1f5653a1e9bc279d550dadd2f5aa20/ui/src/entities/Employee.ts#L27) which hits an API.

Incidentally, in SoS App we communicate status of async operations with toast
notifications. That meant we can get rid of even more boilerplate. With some
[small modifications](https://github.com/sosinc/sos-app/blob/c326ef651e1f5653a1e9bc279d550dadd2f5aa20/ui/src/lib/asyncHooks.ts#L17), we were able to make calls like this:

```ts

    const [createEmployee] = useAsyncThunk(createEmployeeAction, {
      errorTitle: 'Failed to crate Employee',
      successTitle: 'Employee created successfully',
    });

```

which would:

1.  Perform the async action "the Redux way" (i.e by emitting all the right
    actions)
2.  Show error/success toast messages on completion

This boosted our productivity considerably. We could just create an [async-thunk](https://redux-starter-kit.js.org/api/createAsyncThunk),
and using this hook we could go "just do it" with the async operation, and user
feedback was handled for free.

I am pretty okay with this compromise and gains we had from it. But I still feel
the Redux way is bloated. Redux focuses too much on puritanism which don't
really pay off. Until we find a better way, I hope this little tid-bit might of
use for someone else too!

[xstate](https://xstate.js.org/docs/) looks very promising. It's not really a Redux alternative, but something
that can make modular components much more robust.
