> **NOTE**: When this article was written, I was still using XState version 4. I might write about changes or differences between XState version 4 and 5. But fundamentally, the concepts used are the same

In this article, I want to share about finite state machines (FSM) and how to create a finite state machine using XState. I also share the final results of the project mentioned in this article at the following link.

## Setting Up the Project

I am using Next.js 13 with the app directory and TailwindCSS. A tutorial on creating a Next.js project with an app router can be found at this link.

```ts
What is your project named? logs-understanding-fsm-with-xstate
Would you like to use TypeScript? Yes
Would you like to use ESLint? Yes
Would you like to use Tailwind CSS? Yes
Would you like to use `src/` directory? Yes
Would you like to use App Router? (recommended) Yes
Would you like to customize the default import alias (@/*)? Yes
What import alias would you like configured? @/*
```

Another libraries needed will be introduced in the related subchapter.

## What is a Finite State Machine?

What is a finite state machine? The way I understand this term is by trying to dissect and understand each word in the term:

### Machine

Here, 'Machine' refers to 'a model of a system.' The model itself can be interpreted as an informative representation of something (basically, a representation of a system).

### State

In this context, 'State' refers to information. In particular, the information in question is “the behavior of a system”.

When these two words are combined, 'state machine' can be understood as a representation of a system's behavior. A state machine, of course, consists of a list of its behaviors. A state machine also describes the transition from one state to another. These transitions are triggered by inputs given to the state machine.

The last word, '_finite_,' implies that the number of states within the state machine is limited

## Modeling a Fan's Finite State Machine

Given that the weather where I live is hot while I am writing this article, we will use a fan as a case study. We will create a model of the fan using an FSM.

### Installing XState

Before we start modeling the FSM of the fan, we first need to install some libraries related to finite state machines

```bash
yarn add xstate@^4.38.1 @xstate/react@^3.2.2
```

### VSCode Extension

For easier development, I use [the Stately extension in VSCode](https://marketplace.visualstudio.com/items?itemName=statelyai.stately-vscode). This extension greatly facilitates FSM modeling because it uses a GUI.

### Modeling the FSM of a Fan

To create an FSM with XState, we can use the `createMachine` function.

```ts
import { createMachine } from "xstate";
export const fanMachine = createMachine({
  id: "fan",
});
```

The `id` property is used to identify the machine we create.

To model how a fan works, we can start with the question: 'What are the states of a fan?'. Simple, `stop` and`spin`. `stop` means our fan is not spinning, while `spin` means it is spinning. Essentially, there are states when the fan is on and off. We can write the states of this fan into our machine definition as follows:

```ts
import { createMachine } from "xstate";

export const fanMachine = createMachine({
  id: "fan",

  states: {
    stop: {},
    spin: {},
  },

  initial: "off",
});
```

The `states` property lists all the states that our machine possesses, and the `initial` property determines the initial state when the machine is first run.

Now, our machine has states. But, we have not yet determined how our machine can transition from the `stop` state to `spin`.

---

### (Tips) TypeScript Support (you can skip this part if you’re not using TypeScript)

If you are also using TypeScript, XState provides a types generator (typegen) for our machine. According to this documentation, VSCode users only need to install an extension.

Then, in our state machine definition, we just need to add the property `tsTypes: {}`. When we save this file, typings from our state machine will automatically be created.

---

## State Transitions

In XState, the transition from one state to another is triggered by an event. Here, event is equivalent to the 'input' we referred to in the earlier section about the concept of FSM.

For our case, the transitions we need are from the `stop` state to the `spin` state and vice versa. These transitions are triggered by the user turning the fan on or off. In XState, we can write these transitions in the `on` property of a state:

```ts
export const fanMachine = createMachine({
  id: "fan",

  states: {
    stop: {
      on: {
        "USER.PRESS.ON": "spin",
      },
    },

    spin: {
      on: {
        "USER.PRESS.OFF": "stop",
      },
    },
  },

  initial: "stop",
});
```

Essentially, the transition in the `stop` state can be read as: 'When `USER.PRESS.ON`, transition to `spin`.

What’s interesting about XState is that we can use any string value as the name for the event. Here, I use a convention where action names are written in uppercase and each word is separated by a dot instead of a space.

To generate better typings, we can define any event recognized by our machine. For example:

```ts
type MachineEvent = { type: "USER.PRESS.ON" } | { type: "USER.PRESS.OFF" };

export const fanMachine = createMachine({
  id: "fan",
  tsTypes: {} as import("./fanMachine.fsm.typegen").Typegen0,
  schema: {
    events: {} as MachineEvent,
  },
  // ...
});
```

### Enriching the State Machine with Additional Information

We have successfully made our fan turn on and off. But, what about the fan's rotation speed? In XState, additional information (or simply, data) known to the machine is stored in the `context`.

In our case, the additional information needed is the speed of the fan.

We can add speed to the `context` property:

```ts
export const fanMachine = createMachine({
  id: "fan",
  context: {
    fanSpeed: 0,
  },
  // ...
});
```

We can also create type for the `context` and add it to the `schema`:

```ts
type MachineContext = {
  fanSpeed: number;
};

// ...

export const fanMachine = createMachine({
  id: "fan",
  schema: {
    events: {} as MachineEvent,
    context: {} as MachineContext,
  },
  // ...
});
```

Now, our machine has additional information, `fanSpeed`, stored in the `context`.

### Changing the Context Value with an Action

So far, we have added the fan's speed, `fanSpeed`, to the machine through the `context`. But, when the state transitions from `stop` to `spin`, our fan's `fanSpeed` is still 0!

To change the `context` value, we can utilize one of XState's features, namely “action”.

In XState, an action is a form of side-effect that can be triggered. When is an action triggered? An action can be triggered during state transitions, either on entering or exiting a `state`, or by an `event`. An action in XState is a pure function; it is generally synchronous. We can use an action to change the `context` value.

#### Adding `action` to an `event`

First, let's update the `event` we send during the transition from `stop` to `spin` and vice versa to trigger an action that will change the value of `fanSpeed`. Let's name this action `changeFanSpeed`. This action is added to the `actions` property within the `event`.

```ts
export const fanMachine = createMachine({
  id: "fan",
  // ...
  states: {
    stop: {
      on: {
        "USER.PRESS.ON": {
          target: "spin",
          // v let's add action here!
          actions: "changeFanSpeed",
        },
      },
    },

    spin: {
      on: {
        "USER.PRESS.OFF": {
          target: "stop",
          // v let's add action here!
          actions: "changeFanSpeed",
        },
      },
    },
  },
  // ...
});
```

#### Writing the Implementation of the `changeFanSpeed` action

The next step is to write the implementation of the `changeFanSpeed` action.

According to its documentation, the `createMachine` function accepts two arguments, the first being the machine configuration and the second the `options`. One of the properties of `options` is `actions`, where we write the implementation of `actions`. Almost every property in options — whether `guards`, `actions`, or `services` — that is a function, will receive two arguments in order: the `context` when the action is triggered and the event that triggers the action. We can write the `changeFanSpeed` action like this:

```ts
export const fanMachine = createMachine(
  {
    // ...
  },
  {
    actions: {
      changeFanSpeed: (_context, event) => {
        /* implementation goes here */
      },
    },
  }
);
```

However, to change the `context`, we need a built-in action from XState called the “assign action”. Simply put, the assign action is a function that receives a new value to be applied to the `context` and sets this value within the `context`. If the latest `context` value we want is the result of an `action`, we simply wrap that action using the assign function.

```ts
import { assign } from "xstate";

export const fanMachine = createMachine(
  {
    // ...
  },
  {
    actions: {
      changeFanSpeed: assign((_context, event) => {
        /* implementation goes here */
      }),
    },
  }
);
```

For example, if the `fanSpeed` value when the fan is first turned on is `1`. We can write the assign action `changeFanSpeed` like this:

```ts
export const fanMachine = createMachine(
  {
    id: "fan",
    // ...
    states: {
      stop: {
        on: {
          "USER.PRESS.ON": {
            target: "spin",
            actions: "changeFanSpeed",
          },
        },
      },

      spin: {
        on: {
          "USER.PRESS.OFF": {
            target: "stop",
            actions: "changeFanSpeed",
          },
        },
      },
    },
    // ...
  },
  {
    actions: {
      changeFanSpeed: assign((_context, event) => {
        if (event.type === "USER.PRESS.ON") {
          return {
            fanSpeed: 1,
          };
        }
        if (event.type === "USER.PRESS.OFF") {
          return {
            fanSpeed: 0,
          };
        }
        return {};
      }),
    },
  }
);
```

Now, we have our fan FSM!

Next, one of the equally exciting parts: integrating the FSM we have created into the UI!

If you are also using the VSCode extension, our FSM currently looks something like this:

(insert image)

## Integrating the State Machine into the UI

Before proceeding, we need to install some libraries first:

```bash
yarn add framer-motion@^10.16.12 react-icons@^4.12.0
```

### Integrating the FSM into a React Component

First, let's create a component named `Fan.tsx`. This component can be considered as the visual representation of the fan.

In XState, the machine we have defined, `fanMachine`, can be seen as the definition of a process. The process that runs based on our definition is referred to as a “service” or “actor”. Lately, the term “actor” seems to be used more often.

XState provides a hook called `useMachine` to create an actor (process) from our defined machine. This hook returns a tuple, containing information about the running actor in the form of an object, a function to send events to the actor, and a reference to the created actor. Additionally, this hook also binds the actor to the component's lifecycle. So when the component is unmounted, the actor will stop and will start again (from the initial state) when the component is mounted.

```ts
export function Fan() {
  const [fsmState, fsmSendEvent] = useMachine(fanMachine);

  const isOn = fsmState.matches("spin");
  const speed = fsmState.context.fanSpeed;

  return <div>{/* ... */}</div>;
}
```

`isOn` stores the result of the matches method. This method is used to ensure whether the current state of the actor matches the given argument. To access the `context`, we can use the `context` property from the information obtained from the tuple returned by the `useMachine` hook, `fsmState`.

### Creating the UI

Here is a simple UI that represents the fan:

```tsx
// utils to merge an array of `className`s
function cn(...classes: any[]) {
  return classes.filter(Boolean).join(" ");
}

export function Fan() {
  // ...
  return (
    <div className="flex flex-col items-stretch py-3 px-4 bg-gray-200 rounded-lg gap-5">
      <div>
        <FaDotCircle
          className={cn("text-md", isOn ? "text-green-400" : "text-red-400")}
        />
      </div>
      <FaFan className="text-8xl text-gray-500" />
      <div className="flex flex-row item-center justify-between w-full gap-[150px]">
        <button
          className={cn(
            "p-2 rounded-lg",
            !isOn ? "bg-red-300 text-white" : "bg-red-400 text-black"
          )}
        >
          Off
        </button>
        <button
          className={cn(
            "p-2 rounded-lg",
            isOn ? "bg-green-300 text-white" : "bg-green-400 text-black"
          )}
        >
          On
        </button>
      </div>
    </div>
  );
}
```

### Sending `events` to the `actor`

Our UI is now complete, and we want to be able to send events to the actor we have created. We can use `fsmSendEvent`, which we obtained earlier. (One of the things I like most is the IDE suggestions when writing the event we want to send).

```tsx
export function Fan() {
  // ...
  return (
    <div className="flex flex-col items-stretch py-3 px-4 bg-gray-200 rounded-lg gap-5">
      {/* ... */}
      <div className="flex flex-row item-center justify-between w-full gap-[150px]">
        <button
          // ...
          onClick={() => fsmSendEvent({ type: "USER.PRESS.OFF" })}
        >
          Off
        </button>
        <button
          // ...
          onClick={() => fsmSendEvent({ type: "USER.PRESS.ON" })}
        >
          On
        </button>
      </div>
    </div>
  );
}
```

Just by sending events, our fan machine is now integrated with our React component! But there's one more thing missing, which is animation. Don't worry, it won't take long ;)

### Adding Fan Rotation Animation

For the animation part, we use `framer-motion`. We want to rotate the fan blades based on whether it's on or off and at the predetermined speed. Considering this article is not about animation, I won't go into a lengthy explanation of how to use framer-motion in detail.

```tsx
export function Fan() {
  // animation values
  const calculatedSpeed = isOn ? 1000 - 100 * speed : 0;
  const time = useTime();
  const rotate = useTransform(
    time,
    [0, calculatedSpeed], // For every calculatedSpeed,
    [0, -360], // rotate 360 degrees to the left direction
    {
      clamp: false, // make it rotate forever
    }
  );

  return (
    <div className="flex flex-col items-stretch py-3 px-4 bg-gray-200 rounded-lg gap-5">
      <motion.div
        style={{ rotate }}
        className="flex justify-center items-center"
      >
        <FaFan className="text-8xl text-gray-500" />
      </motion.div>
      {/* ... */}
    </div>
  );
}
```

In the above lines of code, we use the rotate generated from the useTransform hook of framer motion.

## Demo

<div style="position: relative; padding-bottom: 74.3801652892562%; height: 0;"><iframe src="https://www.loom.com/embed/5d9faea27d5c4701b4962eb982152926?sid=73c304f3-404d-41e7-8e84-b3d1aad44fd9" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"></iframe></div>

## Summary

In this article, we have learned about the concept of finite state machines and their implementation using XState in Next.js.

I personally believe that XState is a useful library in complex situations.

Thank you for reading!
