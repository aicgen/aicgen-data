# Component-Based Architecture

## Rules

- Build the UI from small, reusable components with a single purpose and explicit props or inputs.
- Structure components in layers (atoms, molecules, organisms, templates, pages) and keep dependencies pointing from pages down to atoms.
- Keep presentational components pure and put data fetching and state in container components or hooks.
- Lift state only as far as needed; use a store for truly shared state.
- Compose rather than inherit, and keep components accessible and tested in isolation.

## Core Principle

Build the application as a composition of reusable, self-contained components. Common in frontend (React/Vue/Flutter) and mobile development.

## Component Types

### Atoms (Basic UI)

Smallest units. Buttons, inputs, icons. No business logic.

### Molecules (Composite)

Groups of atoms. Search bar (Input + Button), User Card (Avatar + Text).

### Organisms (Complex Sections)

Complex standalone UI sections. Navigation bar, Product Grid.

### Templates & Pages

Layout structures and full views connecting organisms with data.

## State Management

- **Local State**: UI state specific to a component (e.g., `isOpen`).
- **Lifted State**: Shared state moved up to a common ancestor.
- **Global Store**: Application-wide state (Redux/Zustand/Bloc) for data accessed by many unrelated components.

## Implementation Example (React)

```tsx
// Atom
const Button = ({ onClick, children }) => (
  <button onClick={onClick}>{children}</button>
);

// Molecule
const SearchBar = ({ onSearch }) => (
  <div className="search">
    <Input />
    <Button onClick={onSearch}>Search</Button>
  </div>
);

// Page (Organism composition)
const Dashboard = () => {
  return (
    <Layout>
      <Header />
      <SearchBar />
      <UserGrid />
    </Layout>
  );
};
```

## Best Practices

- **Single Responsibility**: A component should do one thing well.
- **Props Interface**: Define clear contracts for data input.
- **Composition over Inheritance**: Build complex UIs by combining simpler components.
