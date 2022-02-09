## React RxJS

Because React is inherently synchronous, streams (observables) don't work well out of the box

We need a way to get our state streams into our react components <!-- .element: class="fragment" -->

--
## Bind to the rescue

React RxJS lets you define your observable state outside of your components, and provides a hook to use for data inside of the components

```ts
const [useSocketData, socketData$] = bind(socketService.aggregations$)
const [useFilters, filters$] = bind(filters$$, {default: "Values"})

const MyApp = () => {
    const socketData = useSocketData()
    const filters = useFilters();
}
```

--

## BUT WAIT

This code is inherently synchronous, how can there be a value for `useSocketData()` until the socket actually emits data?

Two types of bind: with or without a default value <!-- .element: class="fragment" -->

Subscribe is an element that leverages suspense to handle when a stream has not emitted a value <!-- .element: class="fragment" -->

--

```ts
const [useSocketData, socketData$] = bind(socketService.aggregations$)
const Component = () => {
    const data = useSocketData();
    return <div>Using {data} Synchronously</div>;
}

const App = () => {
    return (
        <Subscribe fallback={<div>Loading...</div>}>
            <Component />
        </Subscribe>
    )
}
```

Subscribe leverages Suspense in the background, something that will make React behave asynchronously

--

### Building our app with React-RxJS

```tsx [29|33-36|38-53|65-69|72-79|84-98|101-108|128-130|171-173|181-183]
import {
  combineLatest,
  concat,
  debounceTime,
  defer,
  from,
  map,
  merge,
  of,
  scan,
  startWith,
  switchMap,
  take
} from "rxjs";
import IconButton from "@mui/material/IconButton";
import { bind, Subscribe, SUSPENSE } from "@react-rxjs/core";
import { createSignal, mergeWithKey } from "@react-rxjs/utils";
import Box from "@mui/material/Box";
import ArrowCircleLeft from "@mui/icons-material/ArrowCircleLeft";
import ArrowCircleRight from "@mui/icons-material/ArrowCircleRight";

import Header from "./Header";
import ResultCard from "./ResultCard";
import Preferences from "./Preferences";
import { Dog, Size } from "./types";
import { getBreeds, getDogs } from "./api";

// Breeds
const [useBreeds, breeds$] = bind(defer(() => from(getBreeds())));

// Filters

type Filters = { breed: string; size: Size; ageRange: [number, number] };
const [_ageFilter$, setAgeFilter] = createSignal<[number, number]>();
const [_sizeFilter$, setSizeFilter] = createSignal<Size>();
const [_breedFilter$, setBreedFilter] = createSignal<string>();

const _filterUpdates$ = mergeWithKey({
  age: _ageFilter$,
  size: _sizeFilter$,
  breed: _breedFilter$
}).pipe(
  scan((acc, c) => {
    switch (c.type) {
      case "age":
        return { ...acc, ageRange: c.payload };
      case "breed":
        return { ...acc, breed: c.payload };
      case "size":
        return { ...acc, size: c.payload };
    }
  }, {} as Filters)
);

const initialFilters$ = breeds$
  .pipe(
    map((b) => ({
      breed: b[0],
      size: "Smol" as Size,
      ageRange: [0, 100] as [number, number]
    }))
  )
  .pipe(take(1));

const [useFilters, filters$] = bind(
  concat(initialFilters$, _filterUpdates$).pipe(
    scan((acc, f) => ({ ...acc, ...f }))
  )
);

// Dogs
const [, dogs$] = bind(
  filters$.pipe(
    debounceTime(500),
    switchMap((f) =>
      concat(of(SUSPENSE), from(getDogs(f.breed, f.size, f.ageRange)))
    )
  )
);

// Smart Components

// Preferences
const WrappedPreferences = () => {
  const breeds = useBreeds();
  const filters = useFilters();
  return (
    <Preferences
      breeds={breeds}
      currentAgeRange={filters.ageRange}
      currentBreed={filters.breed}
      currentSize={filters.size}
      onAgeChange={setAgeFilter}
      onSizeChange={setSizeFilter}
      onBreedChange={setBreedFilter}
    />
  );
};

// Dog Selection
const [updateAcceptedDogs$, updateAcceptedDogs] = createSignal<Dog>();
const [updateRejectedDogs$, updateRejectedDogs] = createSignal<Dog>();
const accepted$ = updateAcceptedDogs$.pipe(
  scan((acc, c) => [...acc, c.name], [] as string[])
);
const rejected$ = updateRejectedDogs$.pipe(
  scan((acc, c) => [...acc, c.name], [] as string[])
);

const [updateCurrentDogIndex$, updateCurrentDogIndex] = createSignal<number>();
const [useCurrentDogIndex] = bind(updateCurrentDogIndex$, 0);

const dogNamesToFilter$ = merge(accepted$, rejected$).pipe(
  scan((acc, c) => new Set([...Array.from(acc), ...c]), new Set<string>()),
  startWith(new Set<string>())
);

const [useFilteredDogs] = bind(
  combineLatest([dogs$, dogNamesToFilter$]).pipe(
    map(([dogs, toFilter]) => {
      if (dogs === SUSPENSE) return dogs;
      return dogs.filter((d) => !toFilter.has(d.name));
    })
  )
);

const WrappedDisplay = () => {
  const dogs = useFilteredDogs();
  const currentDogIndex = useCurrentDogIndex();
  const dog = dogs[currentDogIndex];

  return (
    <>
      <Box mr={6}>
        <IconButton
          onClick={() => updateCurrentDogIndex(currentDogIndex - 1)}
          disabled={currentDogIndex === 0}
        >
          <ArrowCircleLeft sx={{ fontSize: 60 }} />
        </IconButton>
      </Box>
      <ResultCard
        dog={dog}
        onCheckClick={() => updateAcceptedDogs(dog)}
        onDismissClick={() => updateRejectedDogs(dog)}
      />
      <Box ml={6}>
        <IconButton
          onClick={() => updateCurrentDogIndex(currentDogIndex + 1)}
          disabled={currentDogIndex === dogs.length - 1}
        >
          <ArrowCircleRight sx={{ fontSize: 60 }} />
        </IconButton>
      </Box>
    </>
  );
};

export default function App() {
  return (
    <Box display="flex" flexDirection="column">
      <Header />
      <Box
        mt={2}
        mb={4}
        mx={6}
        maxWidth={1024}
        minWidth={500}
        alignSelf="center"
      >
        <Subscribe fallback={<div>Loading...</div>}>
          <WrappedPreferences />
        </Subscribe>
        <Box mt={4}>
          <Box
            mt={5}
            display="flex"
            justifyContent="center"
            alignItems="center"
          >
            <Subscribe fallback={<div>Loading...</div>}>
              <WrappedDisplay />
            </Subscribe>
          </Box>
        </Box>
      </Box>
    </Box>
  );
}
```

Note:
* Start from the bottom up
* Define filters with no async state
* createSignal allows for updates
* scan allows for maintaining state just like redux reducers
* initialization with startWith or concat for filters
* Derive dogs from filters
* use values synchronously
* I can use this for component specific state as well
* Easy to lift if I need to share it
* Use subscribe with fallbacks

--

<iframe data-src="https://codesandbox.io/embed/pet-adoption-rxjs-st2bl?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="pet-adoption-rxjs"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

--

## What is good about this approach

State and views aren't mingled <!-- .element: class="fragment -->

Everything is synchronous in react <!-- .element: class="fragment -->

Every state can be made globall available if needed <!-- .element: class="fragment -->

Components only update when the data they care about changes <!-- .element: class="fragment -->

--

## Drawbacks

Steep learning curve! <!-- .element: class="fragment -->

Thinking Reactively <!-- .element: class="fragment -->