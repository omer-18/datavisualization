# DataViz Studio - Architecture Guide

## Overview
This is a React data visualization app that lets users drag data fields onto encoding channels (like X-axis, Y-axis, color) to create charts. It uses **Vega-Lite** under the hood to render the actual visualizations.

---

## 1. App.tsx - The Entry Point

### Component Hierarchy

Looking at `src/App.tsx`, the app has a two-layer structure:

```
App
└── AppContent
    ├── Header
    ├── FieldList
    ├── EncodingPanel
    └── ChartView
```

**What each component does:**
- **App**: Wraps everything in `<AppProvider>` - provides global state to all children
- **AppContent**: The main UI container that handles loading/error states
- **Header**: Shows app title and record count
- **FieldList**: Left sidebar - displays all available data fields
- **EncodingPanel**: Middle sidebar - drop zones for X, Y, color, size, etc.
- **ChartView**: Main area - renders the Vega-Lite chart
```

### Why This Structure?

**`App` component (lines 180-186):**
- Wraps everything in `<AppProvider>` - this makes the context available to ALL child components
- Think of it like a "global state container" that any component can access

**`AppContent` component (lines 6-178):**
- This is where the actual UI lives
- Uses `const { state } = useApp()` to read the global state
- Handles three states:
  1. **Loading** (lines 9-48): Shows a spinner while data loads
  2. **Error** (lines 51-88): Shows error message if data fails to load
  3. **Main UI** (lines 91-177): The 3-column grid layout

**The Grid Layout (lines 94-99):**
```typescript
gridTemplateColumns: '240px 280px 1fr'
```
- Column 1 (240px): `FieldList` - shows all available data fields
- Column 2 (280px): `EncodingPanel` - drop zones for X, Y, color, etc.
- Column 3 (1fr): `ChartView` - takes remaining space, shows the chart

---

## 2. Data Flow - From Context to Components

### The Context Pattern

React Context is like a "global variable" that any component can read. Here's how it works in this app:

#### Step 1: AppContext.tsx - The State Manager

**Initial State (lines 6-12):**
```typescript
const initialState: AppState = {
  data: [],           // The raw JSON data (from cars.json)
  fields: [],        // Detected fields with their types (quantitative, nominal, etc.)
  encodings: {},     // Which fields are assigned to which channels (x, y, color, etc.)
  isLoading: true,
  error: null,
};
```

**The Reducer (lines 14-58):**
- This is a **state machine** - it takes the current state + an action, returns new state
- Actions like `ASSIGN_FIELD`, `REMOVE_FIELD`, `CLEAR_ALL` modify the state
- Example: When you drag a field to X-axis, it dispatches `ASSIGN_FIELD` which updates `encodings.x`

**The Provider (lines 70-106):**
- `AppProvider` wraps the app and provides:
  - `state` - the current app state
  - `assignField()` - function to assign a field to a channel
  - `removeField()` - function to remove a field from a channel
  - `clearAll()` - clears all encodings
  - `toggleFieldType()` - switches between ordinal/nominal

**Data Loading (lines 73-83):**
- On mount, it loads `cars.json` and:
  1. Dispatches `SET_DATA` with the raw data
  2. Calls `detectAllFields()` to analyze each column
  3. Dispatches `SET_FIELDS` with the detected fields

#### Step 2: Components Read from Context

**FieldList.tsx (line 13):**
```typescript
const { state } = useApp();
```
- Reads `state.fields` to display all available fields
- Maps over them to render `<FieldPill>` components

**EncodingPanel.tsx (line 57):**
```typescript
const { clearAll, state } = useApp();
```
- Reads `state.encodings` to see which fields are assigned
- Passes this info to each `<EncodingShelf>` component

**ChartView.tsx (line 7):**
```typescript
const { state } = useApp();
```
- Reads `state.encodings` and `state.data`
- Passes them to `buildVegaSpec()` to generate the chart

**FieldPill.tsx (line 25):**
```typescript
const { toggleFieldType } = useApp();
```
- Uses `toggleFieldType` function when user clicks the O/N toggle button

### The Flow Diagram

```
AppContext (global state)
│
├── FieldList
│   └── Reads: state.fields
│   └── Renders: FieldPill for each field
│
├── EncodingPanel
│   └── Reads: state.encodings
│   └── Renders: EncodingShelf for each channel
│
└── ChartView
    └── Reads: state.encodings + state.data
    └── Builds: Vega spec and renders chart
```

---

## 3. What Happens When You Drag a Field?

Let's trace the complete flow when you drag "Miles_per_Gallon" to the X-axis:

### Step 1: Drag Starts (FieldPill.tsx)

**Line 29-33 in FieldPill.tsx:**
```typescript
const handleDragStart = (e: React.DragEvent) => {
  e.dataTransfer.setData('application/json', JSON.stringify(field));
  e.dataTransfer.effectAllowed = 'copy';
  setIsDragging(true);
};
```

- When you start dragging, it serializes the `field` object (which includes `name`, `type`, `uniqueCount`) into JSON
- Stores it in the browser's drag-and-drop data transfer object
- Sets visual state (`isDragging`) to show the pill is being dragged

### Step 2: Drag Over Drop Zone (EncodingShelf.tsx)

**Lines 31-35 in EncodingShelf.tsx:**
```typescript
const handleDragOver = (e: React.DragEvent) => {
  e.preventDefault();  // Required to allow drop
  e.dataTransfer.dropEffect = 'copy';
  setIsOver(true);     // Visual feedback - highlights the drop zone
};
```

- When you hover over a drop zone (like "X Axis"), it:
  - Prevents default behavior (which would block dropping)
  - Sets the cursor to show it's a valid drop target
  - Updates `isOver` state to highlight the zone

### Step 3: Drop (EncodingShelf.tsx)

**Lines 41-50 in EncodingShelf.tsx:**
```typescript
const handleDrop = (e: React.DragEvent) => {
  e.preventDefault();
  setIsOver(false);

  const fieldData = e.dataTransfer.getData('application/json');
  if (fieldData) {
    const field = JSON.parse(fieldData) as DetectedField;
    assignField(channel, field);  // ← THIS IS THE KEY LINE
  }
};
```

- Retrieves the field data from the drag event
- Parses it back into a `DetectedField` object
- Calls `assignField(channel, field)` - this is from the context!

### Step 4: Context Updates State (AppContext.tsx)

**Lines 85-87 in AppContext.tsx:**
```typescript
const assignField = (channel: EncodingChannel, field: DetectedField) => {
  dispatch({ type: 'ASSIGN_FIELD', channel, field });
};
```

- Dispatches an action to the reducer

**Lines 20-24 in AppContext.tsx (the reducer):**
```typescript
case 'ASSIGN_FIELD':
  return {
    ...state,
    encodings: { ...state.encodings, [action.channel]: action.field },
  };
```

- Creates a new state object (React requires immutability)
- Spreads existing encodings, then adds/updates the specific channel
- Example: `encodings.x = field` (the Miles_per_Gallon field)

### Step 5: React Re-renders Everything

Because the context state changed, all components that read from it re-render:

1. **EncodingShelf** (line 29): `state.encodings[channel]` now has a value, so it shows the field name instead of "Drop field here"

2. **ChartView** (line 10): 
   ```typescript
   const spec = buildVegaSpec(state.encodings, state.data);
   ```
   - Recalculates the Vega spec with the new encoding
   - The `useEffect` (line 12) detects `spec` changed and re-embeds the chart

3. **FieldList**: Still shows all fields (unchanged)

### Visual Result

- The X-axis shelf now shows "Miles_per_Gallon" with a badge
- The chart updates to show a visualization (if Y-axis is also set)

---

## 4. Adding a New Chart Type

To add a new chart type (e.g., "area" chart), you need to modify **one main file**:

### File: `src/utils/vegaSpecBuilder.ts`

The `inferMark()` function (lines 4-36) decides which chart type to use based on the field types:

```typescript
function inferMark(encodings: EncodingState): string {
  const { x, y } = encodings;

  if (x && y) {
    if (x.type === 'quantitative' && y.type === 'quantitative') {
      return 'point';  // Scatter plot
    }
    if ((x.type === 'nominal' || x.type === 'ordinal') && y.type === 'quantitative') {
      return 'bar';   // Bar chart
    }
    // ... more conditions
  }
  
  return 'point';  // Default
}
```

### How to Add "Area" Chart

**Step 1:** Add a condition in `inferMark()`:

```typescript
// Add this after line 22 (after the line chart conditions)
if (x.type === 'temporal' && y.type === 'quantitative') {
  return 'area';  // New chart type!
}
```

**Step 2:** Update the type annotation (line 88):

```typescript
mark: { type: mark as 'point' | 'bar' | 'line' | 'rect' | 'area', tooltip: true },
```

That's it! Vega-Lite supports many mark types out of the box. The chart will automatically render as an area chart when:
- X-axis has a temporal field (dates)
- Y-axis has a quantitative field (numbers)

### Other Chart Types You Could Add

Vega-Lite supports: `'area' | 'bar' | 'line' | 'point' | 'rect' | 'circle' | 'square' | 'tick' | 'geoshape' | 'rule' | 'text' | 'boxplot' | 'errorband' | 'errorbar'`

Just add conditions in `inferMark()` and update the type annotation.

### Optional: Add UI Controls

If you want users to manually select chart types (instead of auto-inferring), you'd need to:

1. **Add to state** (`src/types/index.ts`):
   ```typescript
   export interface AppState {
     // ... existing fields
     chartType?: 'point' | 'bar' | 'line' | 'area';  // New field
   }
   ```

2. **Add action** (`src/types/index.ts`):
   ```typescript
   export type AppAction =
     // ... existing actions
     | { type: 'SET_CHART_TYPE'; payload: 'point' | 'bar' | 'line' | 'area' };
   ```

3. **Add reducer case** (`src/context/AppContext.tsx`):
   ```typescript
   case 'SET_CHART_TYPE':
     return { ...state, chartType: action.payload };
   ```

4. **Add UI control** (maybe in `EncodingPanel.tsx` or `ChartView.tsx`):
   - A dropdown or buttons to select chart type
   - Call `dispatch({ type: 'SET_CHART_TYPE', payload: 'area' })` on selection

5. **Use in builder** (`src/utils/vegaSpecBuilder.ts`):
   ```typescript
   export function buildVegaSpec(
     encodings: EncodingState,
     data: Record<string, unknown>[],
     chartType?: string  // New parameter
   ): TopLevelSpec | null {
     // ...
     const mark = chartType || inferMark(encodings);  // Use manual selection or infer
   }
   ```

---

## Key React Patterns Used

1. **Context API**: Global state management without prop drilling
2. **useReducer**: Complex state updates with actions (like Redux, but built-in)
3. **Custom Hooks**: `useApp()` provides a clean API to access context
4. **Controlled Components**: All state flows through context, components are "dumb"
5. **Effect Hooks**: `useEffect` in ChartView watches for spec changes and re-renders
6. **HTML5 Drag & Drop**: Native browser API for drag-and-drop (not a library!)

---

## File Responsibilities Summary

- **App.tsx**: Layout and routing (well, just layout - no routing here)
- **AppContext.tsx**: State management, data loading, business logic
- **FieldList.tsx**: Displays available fields
- **FieldPill.tsx**: Individual draggable field component
- **EncodingPanel.tsx**: Container for encoding shelves
- **EncodingShelf.tsx**: Individual drop zone for one channel (x, y, color, etc.)
- **ChartView.tsx**: Renders the Vega-Lite chart
- **vegaSpecBuilder.ts**: Converts encodings + data → Vega-Lite JSON spec
- **fieldDetection.ts**: Analyzes raw data to determine field types
- **types/index.ts**: TypeScript type definitions

---

## Questions?

This architecture follows common React patterns:
- **Single source of truth**: All state in AppContext
- **Unidirectional data flow**: Context → Components (no components modifying context directly, only through actions)
- **Separation of concerns**: UI components vs. business logic vs. data transformation

If you want to understand any specific part deeper, let me know!
