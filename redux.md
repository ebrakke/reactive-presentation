## Redux
Redux has long been the defacto state management solution for large react products

--

### Why is Redux popular?
Single source of truth <!-- .element: class="fragment" -->

Predictability <!-- .element: class="fragment" -->

Immutability <!-- .element: class="fragment" -->

Common terms to describe ideas <!-- .element: class="fragment" -->

Framework Agnostic <!-- .element: class="fragment" -->

Note:
* Single source of truth helps ensure all components get the most up to date data
* Predictability comes from the fact that actions are processed synchronously
* Actions, reducers, middleware, stores.  Anyone who uses redux will know what you're talking about
* Easy to get people onboarded with redux in any framework, and the ideas remain consistent
* Came about at a time where all of this had to be built into a framework, so angularJS was different than Ember

--

### Modeling our application with Redux
```tsx [|22-30|32-38|39|48-76|78-83|233-237|205|217,225|89|90-93|98-105|102-103|117-119|121-132|134|144-155|166|174-203|157-163]
import IconButton from "@mui/material/IconButton";
import Box from "@mui/material/Box";
import ArrowCircleLeft from "@mui/icons-material/ArrowCircleLeft";
import ArrowCircleRight from "@mui/icons-material/ArrowCircleRight";

import { configureStore, createSlice } from "@reduxjs/toolkit";
import {
  Provider,
  TypedUseSelectorHook,
  useDispatch,
  useSelector
} from "react-redux";

import Header from "./Header";
import ResultCard from "./ResultCard";
import Preferences from "./Preferences";
import { Dog, Size } from "./types";
import { useEffect } from "react";
import { getBreeds, getDogs } from "./api";

// Redux Config
type CommonState = { breeds: null | "Loading" | string[] };
const commonSlice = createSlice({
  name: "common",
  initialState: { breeds: null } as CommonState,
  reducers: {
    breedsLoading: (state) => ({ ...state, breeds: "Loading" }),
    breedsLoaded: (state, action) => ({ ...state, breeds: action.payload })
  }
});

type SelectionState = {
  dogs: null | "Loading" | Dog[];
  currentDog: number;
  filters: { breed: string | null; size: Size; ageRange: [number, number] };
  acceptedDogs: Dog[];
  rejectedDogs: Dog[];
};
const selectionSlice = createSlice({
  name: "selection",
  initialState: {
    dogs: null,
    currentDog: 0,
    filters: { breed: null, size: "Smol", ageRange: [0, 100] },
    acceptedDogs: [],
    rejectedDogs: []
  } as SelectionState,
  reducers: {
    dogsLoading: (state) => ({ ...state, dogs: "Loading" }),
    dogsLoaded: (state, action) => ({ ...state, dogs: action.payload }),
    setBreedFilter: (state, action) => ({
      ...state,
      filters: { ...state.filters, breed: action.payload }
    }),
    setSizeFilter: (state, action) => ({
      ...state,
      filters: { ...state.filters, size: action.payload }
    }),
    setAgeFilter: (state, action) => ({
      ...state,
      filters: { ...state.filters, ageRange: action.payload }
    }),
    setCurrentDog: (state, action) => ({
      ...state,
      currentDog: action.payload
    }),
    acceptDog: (state, action) => ({
      ...state,
      acceptedDogs: [...state.acceptedDogs, action.payload]
    }),
    rejectDog: (state, action) => ({
      ...state,
      rejectedDogs: [...state.rejectedDogs, action.payload]
    })
  }
});

const store = configureStore({
  reducer: {
    common: commonSlice.reducer,
    selection: selectionSlice.reducer
  }
});
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;

const WrappedPreferences = () => {
  const state = useAppSelector((state) => ({
    breeds: state.common.breeds,
    filters: state.selection.filters
  }));
  const dispatch = useAppDispatch();

  useEffect(() => {
    // Load breeds if they haven't been loaded yet
    if (!state.breeds) {
      dispatch(commonSlice.actions.breedsLoading());
      getBreeds().then((breeds) => {
        dispatch(commonSlice.actions.breedsLoaded(breeds));
        if (!state.filters.breed)
          dispatch(selectionSlice.actions.setBreedFilter(breeds[0]));
      });
    }
  }, [state.breeds, state.filters.breed]);

  const onBreedChange = (b: string) =>
    dispatch(selectionSlice.actions.setBreedFilter(b));
  const onAgeChange = (a: [number, number]) =>
    dispatch(selectionSlice.actions.setAgeFilter(a));
  const onSizeChange = (s: Size) =>
    dispatch(selectionSlice.actions.setSizeFilter(s));

  const breeds = state.breeds;

  if (breeds === "Loading" || breeds === null || !state.filters.breed) {
    return <div>Loading...</div>;
  }

  return (
    <Preferences
      breeds={state.breeds as string[]}
      currentBreed={state.filters.breed as string}
      currentAgeRange={state.filters.ageRange}
      currentSize={state.filters.size}
      onBreedChange={onBreedChange}
      onAgeChange={onAgeChange}
      onSizeChange={onSizeChange}
    />
  );
};

const WrappedResultCard = () => {
  const state = useAppSelector((state) => ({
    filters: state.selection.filters,
    dogs: state.selection.dogs,
    currentDog: state.selection.currentDog,
    acceptedDogs: state.selection.acceptedDogs,
    rejectedDogs: state.selection.rejectedDogs
  }));
  const dispatch = useAppDispatch();

  useEffect(() => {
    if (!state.filters.breed) return;
    dispatch(selectionSlice.actions.dogsLoading());
    getDogs(
      state.filters.breed,
      state.filters.size,
      state.filters.ageRange
    ).then((dogs) => {
      dispatch(selectionSlice.actions.dogsLoaded(dogs));
      dispatch(selectionSlice.actions.setCurrentDog(0));
    });
  }, [state.filters]);

  const onDogAccept = (d: Dog) => {
    dispatch(selectionSlice.actions.acceptDog(d));
  };

  const onDogReject = (d: Dog) => {
    dispatch(selectionSlice.actions.rejectDog(d));
  };

  const dogs = state.dogs;
  if (!dogs || dogs === "Loading") return <div>Loading...</div>;
  const filteredDogs = dogs.filter(
    (d) =>
      !state.acceptedDogs.find((ad) => ad.name === d.name) &&
      !state.rejectedDogs.find((rd) => rd.name === d.name)
  );

  if (!filteredDogs.length) return <div>No dogs found</div>;
  return (
    <>
      <Box mr={6}>
        <IconButton
          onClick={() =>
            dispatch(selectionSlice.actions.setCurrentDog(state.currentDog - 1))
          }
          disabled={state.currentDog === 0}
        >
          <ArrowCircleLeft sx={{ fontSize: 60 }} />
        </IconButton>
      </Box>
      <ResultCard
        dog={filteredDogs[state.currentDog]}
        onCheckClick={() => onDogAccept(filteredDogs[state.currentDog])}
        onDismissClick={() => onDogReject(filteredDogs[state.currentDog])}
      />
      <Box ml={6}>
        <IconButton
          onClick={() =>
            dispatch(selectionSlice.actions.setCurrentDog(state.currentDog + 1))
          }
          disabled={state.currentDog === filteredDogs.length - 1}
        >
          <ArrowCircleRight sx={{ fontSize: 60 }} />
        </IconButton>
      </Box>
    </>
  );
};

function App() {
  return (
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
            <WrappedResultCard />
          </Box>
        </Box>
      </Box>
    </Box>
  );
}

export default () => (
  <Provider store={store}>
    <App />
  </Provider>
);
```
Note:
* First, we need to model what our application state looks like. Common state can be used app wide, pages have page specific info.
* Need to track async stuff so we know when we can actually render the page
* Use redux toolkit to remove some boilerplate
* Need to coordinate within the component about which data to load

--

<iframe data-src="https://codesandbox.io/embed/pet-adoption-redux-ljhfk?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="pet-adoption-redux"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

--

## What is good about this approach?

--

There's only one convention for storing and accessing data app wide (by convention)

Any changes to data will be picked up by components (reactive) <!-- .element: class="fragment" -->

All actions are synchronous, which works well with how react handles data <!-- .element: class="fragment" -->

Battle tested in many large applications <!-- .element: class="fragment" -->

Note:
* As long as we always load data from redux, data will be in sync
* There is a recipe for just about everything, and lots of support
* Lots of middleware to make redux whatever you want

--

## What could be improved?

--

Lots of places to modify if new data is needed (action and reducer)

Boilerplate (even with redux toolkit) <!-- .element: class="fragment" -->

Managing data dependencies <!-- .element: class="fragment" -->

Managing loading states (synchronous) <!-- .element: class="fragment" -->

Making decisions about where to store state <!-- .element: class="fragment"-->

Industry is moving away from redux <!-- .element: class="fragment" -->

Even Dan Abramov (the creator) wouldn't use redux when starting a new project <!-- .element: class="fragment" -->

Note:
* You might keep things in local state just because you don't know if it's worth the hassle of lifting it to redux
* Even with redux tool kit, we need to make sure that when want new data, we update in the correct spots
* Potentially many spots call on a single action, so if that action changes, updates are needed everywhere
* Unknown if data has already loaded, need to manually check it during data loading
* A developer needs to decide where is best to put the data in the store
* Stores usually get massive in large apps and become cumbersome to know where all data lives