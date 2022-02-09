## Hooks and Context

--

React ships with all the tools you would need to handle state management.  This can be achieved with Hooks and Context

--

## What are hooks?

--

Hooks provide a way to manage side effects within a React component

--

```tsx
import { useState } from "react";

export default function App() {
  const [count, setCount] = useState(0);
  return (
    <div className="container">
      <div>
        <h2>The Count is {count}</h2>
        <button onClick={() => setCount(count + 1)}>Increment</button>
        <button onClick={() => setCount(count - 1)}>Decrement</button>
      </div>
    </div>
  );
}
```

--

`useState` is great for local state, but how do we share it?

--

## Prop Drilling

--

```tsx [22-34|23|27|29-30|4-7|9-12]
import React from "react";
import "./styles.css";

type DeepNestedProps = { count: number };
function DeeplyNested({ count }: DeepNestedProps) {
  return <div>The Count is {count}</div>;
}

type ButtonProps = React.PropsWithChildren<{ onClick: () => void }>;
function Button({ onClick, children }: ButtonProps) {
  return <button onClick={() => onClick()}>{children}</button>;
}

function Nester({ children, ...props }: React.PropsWithChildren<any>) {
  return (
    <div>
      <DeeplyNested count={props.count} />
    </div>
  );
}

export default function App() {
  const [count, setCount] = React.useState(0);
  return (
    <div className="App">
      <div>
        <Nester count={count} />
        <br />
        <Button onClick={() => setCount(count + 1)}>Increment</Button>
        <Button onClick={() => setCount(count - 1)}>Decrement</Button>
      </div>
    </div>
  );
}

```

--

Prop drilling gets very hard to manage when your application grows beyond just a few components

--

State gets tightly coupled to a component as well, and we want components to just worry about view logic

--

## Context to the rescue

--

We can "lift" shared state up into a context so all children can get access to data without prop drilling

--

## Modeling our app with Hooks and Context

--

```tsx [14-23|25-36|40-41|43-50|52|67,72|78-83|68-71|120,122]
import IconButton from "@mui/material/IconButton";
import { includes } from "lodash";
import Box from "@mui/material/Box";
import ArrowCircleLeft from "@mui/icons-material/ArrowCircleLeft";
import ArrowCircleRight from "@mui/icons-material/ArrowCircleRight";

import Header from "./Header";
import ResultCard from "./ResultCard";
import Preferences from "./Preferences";
import { Dog, Size } from "./types";
import React, { createContext, useContext, useEffect, useState } from "react";
import { getBreeds, getDogs } from "./api";

// Filters Setup
type Filters = { breed: string | null; size: Size; ageRange: [number, number] };
type FilterState = {
  filters: Filters;
  setFilters: (f: Filters) => void;
};
const FilterContext = createContext<FilterState>({
  filters: { breed: null, size: "Smol", ageRange: [0, 100] },
  setFilters: (f: Filters) => {}
});
const useFiltersContext = () => useContext(FilterContext);

const FiltersProvider: React.FC = ({ children }) => {
  const [filters, setFilters] = useState<Filters>({
    breed: null,
    size: "Smol",
    ageRange: [0, 100]
  });
  return (
    <FilterContext.Provider value={{ filters, setFilters }}>
      {children}
    </FilterContext.Provider>
  );
};

const WrappedPreferences = () => {
  const [breeds, setBreeds] = useState<null | string[]>(null);
  const { filters, setFilters } = useFiltersContext();

  useEffect(() => {
    if (!breeds) {
      getBreeds().then((breeds) => {
        setBreeds(breeds);
        if (!filters.breed) setFilters({ ...filters, breed: breeds[0] });
      });
    }
  }, [breeds]);

  if (!breeds || !filters.breed) return <div>Loading...</div>;

  return (
    <Preferences
      breeds={breeds}
      currentAgeRange={filters.ageRange}
      currentBreed={filters.breed}
      currentSize={filters.size}
      onAgeChange={(ageRange) => setFilters({ ...filters, ageRange })}
      onSizeChange={(size) => setFilters({ ...filters, size })}
      onBreedChange={(breed) => setFilters({ ...filters, breed })}
    />
  );
};

const WrappedResults = () => {
  const [dogs, setDogs] = useState<Dog[] | null>(null);
  const [rejectedDogs, setRejectedDogs] = useState<string[]>([]);
  const [acceptedDogs, setAcceptedDogs] = useState<string[]>([]);
  const [currentDog, setCurrentDog] = useState(0);
  const { filters } = useFiltersContext();

  useEffect(() => {
    setDogs(null);
  }, [filters]);

  useEffect(() => {
    if (!filters.breed) return; // Can't load anything without breed
    getDogs(filters.breed, filters.size, filters.ageRange).then((dogs) => {
      setDogs(dogs);
    });
  }, [filters.breed, filters.ageRange, filters.size]);

  if (!dogs) return <div>Loading...</div>;
  if (dogs.length === 0) return <div>No Dogs</div>;
  const filteredDogs = dogs.filter(
    (d) => !includes(rejectedDogs, d.name) && !includes(acceptedDogs, d.name)
  );

  const dog = filteredDogs[currentDog];

  return (
    <>
      <Box mr={6}>
        <IconButton
          onClick={() => setCurrentDog(currentDog - 1)}
          disabled={currentDog === 0}
        >
          <ArrowCircleLeft sx={{ fontSize: 60 }} />
        </IconButton>
      </Box>
      <ResultCard
        dog={dog}
        onCheckClick={() => setAcceptedDogs([...acceptedDogs, dog.name])}
        onDismissClick={() => setRejectedDogs([...rejectedDogs, dog.name])}
      />
      <Box ml={6}>
        <IconButton
          onClick={() => setCurrentDog(currentDog + 1)}
          disabled={currentDog === filteredDogs.length - 1}
        >
          <ArrowCircleRight sx={{ fontSize: 60 }} />
        </IconButton>
      </Box>
    </>
  );
};

export default function App() {
  return (
    <FiltersProvider>
      <Box display="flex" flexDirection="column">
        <Header />
        <Box
          mt={2}
          mb={4}
          mx={6}
          maxWidth={1024}
          minWidth={720}
          alignSelf="center"
        >
          <WrappedPreferences />
          <Box mt={4}>
            <Box
              mt={5}
              display="flex"
              justifyContent="center"
              alignItems="center"
            >
              <WrappedResults />
            </Box>
          </Box>
        </Box>
      </Box>
    </FiltersProvider>
  );
}
```

Note:
* First, setup filters context as this data is shared across two components
* Deal with async state in our context
* Define a context, leave breed null.  We could also decide to lead the breeds in here
* Initial loading logic in `useEffect` for preferences. Again, filters relies on breeds, but it's hard to know exactly where to put the loading logic
* Have to handle async state in rendering logic
* When using filters context in results view, need to rely on the fact that the breed has been loaded and set
* Need to wrap whole app in FiltersProvider to both children get access to it

--

<iframe data-src="https://codesandbox.io/embed/pet-adoption-hooks-context-bzq10?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="pet-adoption-hooks-context"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

--

## What is good about this approach?

--

No need for extra libraries, everything is included in React 

Keeping state local tends to be easier to start <!-- .element: class="fragment" -->

Flexible, there is no "one way" to use Providers, you can use it to suit your needs <!-- .element: class="fragment" -->

Note:
* Unlike redux which has very specific ways to dispatch actions, you can call whatever functions you want with Providers
* I like to think about providers as a DI system for React
* react-redux uses a Provider to work

--

## What could be improved?

--

Data dependencies are still hard to track

Performance issues <!-- .element: class="fragment" -->

Context Overload <!-- .element: class="fragment" -->

Scaling issues <!-- .element: class="fragment" -->

Un-opinionated <!-- .element: class="fragment" -->

Note:
* Contexts will re-render children whenever a value changes, even if the child doesn't care about it
* Can't memoize a component that relies on context values
* All users of context must be children of the provider
* It's better to keep contexts small, but also have too many can be cumbersome as well
* Un-opinionated, can lead to unclear APIs.  Different team members might implement Contexts in different ways