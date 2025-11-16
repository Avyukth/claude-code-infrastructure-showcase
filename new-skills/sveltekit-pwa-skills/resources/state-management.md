# State Management - SvelteKit PWAs

**Advanced state management patterns for PWA components**


## State Management Patterns

### Store Factory Pattern

```javascript
// lib/stores/store-factory.js
import { writable, derived, get } from 'svelte/store';

export function createStore(initialState, actions) {
  const { subscribe, set, update } = writable(initialState);
  
  const methods = {};
  
  // Bind actions to store
  for (const [name, action] of Object.entries(actions)) {
    methods[name] = (...args) => {
      update(state => action(state, ...args));
    };
  }
  
  return {
    subscribe,
    set,
    update,
    ...methods,
    // Helper methods
    reset: () => set(initialState),
    get: () => get({ subscribe })
  };
}

// Usage example
export const todoStore = createStore(
  { items: [], filter: 'all' },
  {
    addItem: (state, item) => ({
      ...state,
      items: [...state.items, { ...item, id: Date.now() }]
    }),
    
    removeItem: (state, id) => ({
      ...state,
      items: state.items.filter(item => item.id !== id)
    }),
    
    setFilter: (state, filter) => ({
      ...state,
      filter
    })
  }
);

// Derived store
export const filteredTodos = derived(
  todoStore,
  $todos => {
    switch ($todos.filter) {
      case 'completed':
        return $todos.items.filter(item => item.completed);
      case 'active':
        return $todos.items.filter(item => !item.completed);
      default:
        return $todos.items;
    }
  }
);
```

### Context-Based State

```svelte
<!-- lib/components/providers/ThemeProvider.svelte -->
<script>
  import { setContext, onMount } from 'svelte';
  import { writable } from 'svelte/store';
  
  const THEME_KEY = 'theme';
  
  // Create theme store
  const theme = writable('light');
  
  // Load saved theme
  onMount(() => {
    const saved = localStorage.getItem('theme');
    if (saved) {
      theme.set(saved);
    } else if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
      theme.set('dark');
    }
    
    // Watch for system changes
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    const handler = (e) => {
      if (!localStorage.getItem('theme')) {
        theme.set(e.matches ? 'dark' : 'light');
      }
    };
    
    mediaQuery.addEventListener('change', handler);
    
    return () => mediaQuery.removeEventListener('change', handler);
  });
  
  // Apply theme to document
  $: {
    if (typeof document !== 'undefined') {
      document.documentElement.setAttribute('data-theme', $theme);
      localStorage.setItem('theme', $theme);
    }
  }
  
  // Provide theme context
  setContext(THEME_KEY, {
    theme,
    toggleTheme: () => {
      theme.update(t => t === 'light' ? 'dark' : 'light');
    },
    setTheme: (newTheme) => {
      theme.set(newTheme);
    }
  });
</script>

<div class="theme-provider">
  <slot />
</div>

<!-- Usage in child component -->
<script>
  import { getContext } from 'svelte';
  
  const { theme, toggleTheme } = getContext('theme');
</script>

<button on:click={toggleTheme}>
  Current theme: {$theme}
</button>
```

### Reactive State Management

```svelte
<!-- lib/components/ReactiveForm.svelte -->
<script>
  import { createEventDispatcher } from 'svelte';
  
  const dispatch = createEventDispatcher();
  
  // Form state
  let values = {};
  let errors = {};
  let touched = {};
  let isSubmitting = false;
  
  // Reactive validation
  $: isValid = Object.keys(errors).length === 0 && 
               Object.keys(touched).length > 0;
  
  // Reactive form status
  $: formStatus = {
    pristine: Object.keys(touched).length === 0,
    dirty: Object.keys(touched).length > 0,
    valid: isValid,
    submitting: isSubmitting
  };
  
  // Validation rules
  const validators = {
    required: (value) => !value ? 'Required' : null,
    email: (value) => {
      const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      return !regex.test(value) ? 'Invalid email' : null;
    },
    minLength: (min) => (value) => 
      value.length < min ? `Minimum ${min} characters` : null
  };
  
  // Field registration
  export function field(name, rules = []) {
    return {
      name,
      value: values[name] || '',
      error: errors[name],
      touched: touched[name],
      on: {
        input: (e) => {
          values[name] = e.target.value;
          validateField(name, e.target.value, rules);
        },
        blur: () => {
          touched[name] = true;
          validateField(name, values[name], rules);
        }
      }
    };
  }
  
  function validateField(name, value, rules) {
    const fieldErrors = rules
      .map(rule => rule(value))
      .filter(Boolean);
    
    if (fieldErrors.length > 0) {
      errors[name] = fieldErrors[0];
    } else {
      delete errors[name];
    }
    
    errors = { ...errors }; // Trigger reactivity
  }
  
  async function handleSubmit() {
    if (!isValid || isSubmitting) return;
    
    isSubmitting = true;
    
    try {
      await dispatch('submit', values);
      // Reset form
      values = {};
      errors = {};
      touched = {};
    } catch (error) {
      errors.form = error.message;
    } finally {
      isSubmitting = false;
    }
  }
</script>

<form on:submit|preventDefault={handleSubmit}>
  <slot {field} {formStatus} {errors} />
  
  <button 
    type="submit" 
    disabled={!isValid || isSubmitting}
  >
    {isSubmitting ? 'Submitting...' : 'Submit'}
  </button>
</form>
```

## Composition Patterns

### Compound Components

```svelte
<!-- lib/components/Tabs/index.js -->
export { default as Tabs } from './Tabs.svelte';
export { default as TabList } from './TabList.svelte';
export { default as Tab } from './Tab.svelte';
export { default as TabPanels } from './TabPanels.svelte';
export { default as TabPanel } from './TabPanel.svelte';

<!-- Tabs.svelte -->
<script>
  import { setContext } from 'svelte';
  import { writable } from 'svelte/store';
  
  export let defaultIndex = 0;
  
  const activeTab = writable(defaultIndex);
  
  setContext('tabs', {
    activeTab,
    selectTab: (index) => activeTab.set(index)
  });
</script>

<div class="tabs" {...$$restProps}>
  <slot />
</div>

<!-- Tab.svelte -->
<script>
  import { getContext } from 'svelte';
  
  export let index;
  
  const { activeTab, selectTab } = getContext('tabs');
  
  $: isActive = $activeTab === index;
</script>

<button
  role="tab"
  aria-selected={isActive}
  tabindex={isActive ? 0 : -1}
  on:click={() => selectTab(index)}
  class:active={isActive}
>
  <slot />
</button>

<!-- Usage -->
<Tabs defaultIndex={0}>
  <TabList>
    <Tab index={0}>Profile</Tab>
    <Tab index={1}>Settings</Tab>
    <Tab index={2}>Security</Tab>
  </TabList>
  
  <TabPanels>
    <TabPanel index={0}>
      <ProfileContent />
    </TabPanel>
    <TabPanel index={1}>
      <SettingsContent />
    </TabPanel>
    <TabPanel index={2}>
      <SecurityContent />
    </TabPanel>
  </TabPanels>
</Tabs>
```

### Render Props Pattern

```svelte
<!-- lib/components/DataFetcher.svelte -->
<script>
  import { onMount } from 'svelte';
  
  export let url;
  export let transform = (data) => data;
  
  let data = null;
  let error = null;
  let loading = true;
  
  async function fetchData() {
    loading = true;
    error = null;
    
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error('Failed to fetch');
      
      const json = await response.json();
      data = transform(json);
    } catch (e) {
      error = e;
    } finally {
      loading = false;
    }
  }
  
  onMount(fetchData);
  
  $: if (url) fetchData();
</script>

<slot {data} {error} {loading} {refetch: fetchData} />

<!-- Usage -->
<DataFetcher 
  url="/api/users"
  transform={(users) => users.filter(u => u.active)}
  let:data
  let:loading
  let:error
>
  {#if loading}
    <LoadingSpinner />
  {:else if error}
    <ErrorMessage {error} />
  {:else}
    <UserList users={data} />
  {/if}
</DataFetcher>
```

### Higher-Order Components

```javascript
// lib/components/withAuth.js
import { get } from 'svelte/store';
import { goto } from '$app/navigation';
import { user } from '$lib/stores/auth';

export function withAuth(Component, options = {}) {
  return class AuthenticatedComponent extends Component {
    constructor(props) {
      const currentUser = get(user);
      
      if (!currentUser && options.redirect !== false) {
        goto('/login');
        return;
      }
      
      if (options.roles && !options.roles.includes(currentUser?.role)) {
