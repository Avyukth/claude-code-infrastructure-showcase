# Accessibility Patterns for SvelteKit PWAs

## Overview

Accessibility ensures your PWA is usable by everyone, including users with disabilities. This guide covers ARIA patterns, keyboard navigation, screen reader support, and inclusive design principles.

## Core Accessibility Principles

### POUR Guidelines

1. **Perceivable** - Information must be presentable in different ways
2. **Operable** - Interface components must be operable by keyboard
3. **Understandable** - Information and UI operation must be understandable
4. **Robust** - Content must work with assistive technologies

## Keyboard Navigation

### Focus Management

```svelte
<!-- lib/components/FocusTrap.svelte -->
<script>
  import { onMount, onDestroy } from 'svelte';
  
  export let active = false;
  
  let trapElement;
  let previousFocus;
  
  function getFocusableElements() {
    if (!trapElement) return [];
    
    const selector = `
      a[href]:not([disabled]),
      button:not([disabled]),
      textarea:not([disabled]),
      input:not([disabled]),
      select:not([disabled]),
      [tabindex]:not([tabindex="-1"])
    `;
    
    return Array.from(trapElement.querySelectorAll(selector));
  }
  
  function handleKeydown(e) {
    if (!active || e.key !== 'Tab') return;
    
    const focusable = getFocusableElements();
    const first = focusable[0];
    const last = focusable[focusable.length - 1];
    
    if (e.shiftKey) {
      // Backward tab
      if (document.activeElement === first) {
        e.preventDefault();
        last?.focus();
      }
    } else {
      // Forward tab
      if (document.activeElement === last) {
        e.preventDefault();
        first?.focus();
      }
    }
  }
  
  function handleEscape(e) {
    if (active && e.key === 'Escape') {
      active = false;
    }
  }
  
  $: if (active && trapElement) {
    previousFocus = document.activeElement;
    const focusable = getFocusableElements();
    focusable[0]?.focus();
  } else if (!active && previousFocus) {
    previousFocus.focus();
  }
  
  onMount(() => {
    document.addEventListener('keydown', handleKeydown);
    document.addEventListener('keydown', handleEscape);
  });
  
  onDestroy(() => {
    document.removeEventListener('keydown', handleKeydown);
    document.removeEventListener('keydown', handleEscape);
  });
</script>

<div 
  bind:this={trapElement}
  class="focus-trap"
  class:active
>
  <slot />
</div>
```

### Skip Navigation

```svelte
<!-- +layout.svelte -->
<script>
  import { page } from '$app/stores';
  
  function skipToMain() {
    const main = document.getElementById('main-content');
    main?.focus();
    main?.scrollIntoView();
  }
</script>

<a 
  href="#main-content" 
  class="skip-link"
  on:click|preventDefault={skipToMain}
>
  Skip to main content
</a>

<nav aria-label="Main navigation">
  <!-- Navigation items -->
</nav>

<main id="main-content" tabindex="-1">
  <slot />
</main>

<style>
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: var(--primary-color);
    color: white;
    padding: 8px;
    text-decoration: none;
    z-index: 100;
  }
  
  .skip-link:focus {
    top: 0;
  }
</style>
```

### Roving Tabindex Pattern

```svelte
<!-- lib/components/RadioGroup.svelte -->
<script>
  export let options = [];
  export let value = null;
  export let name;
  
  let focusedIndex = 0;
  
  function handleKeydown(e) {
    const { key } = e;
    const lastIndex = options.length - 1;
    
    switch(key) {
      case 'ArrowDown':
      case 'ArrowRight':
        e.preventDefault();
        focusedIndex = focusedIndex === lastIndex ? 0 : focusedIndex + 1;
        options[focusedIndex]?.focus();
        value = options[focusedIndex].value;
        break;
        
      case 'ArrowUp':
      case 'ArrowLeft':
        e.preventDefault();
        focusedIndex = focusedIndex === 0 ? lastIndex : focusedIndex - 1;
        options[focusedIndex]?.focus();
        value = options[focusedIndex].value;
        break;
        
      case 'Home':
        e.preventDefault();
        focusedIndex = 0;
        options[0]?.focus();
        value = options[0].value;
        break;
        
      case 'End':
        e.preventDefault();
        focusedIndex = lastIndex;
        options[lastIndex]?.focus();
        value = options[lastIndex].value;
        break;
        
      case ' ':
      case 'Enter':
        e.preventDefault();
        value = options[focusedIndex].value;
        break;
    }
  }
</script>

<div 
  role="radiogroup"
  aria-label={name}
  on:keydown={handleKeydown}
>
  {#each options as option, i}
    <div
      role="radio"
      tabindex={i === focusedIndex ? 0 : -1}
      aria-checked={value === option.value}
      bind:this={options[i]}
      on:click={() => {
        value = option.value;
        focusedIndex = i;
      }}
      on:focus={() => focusedIndex = i}
    >
      {option.label}
    </div>
  {/each}
</div>
```

## ARIA Patterns

### Live Regions

```svelte
<!-- lib/components/LiveAnnouncer.svelte -->
<script>
  import { onMount } from 'svelte';
  
  let announcements = [];
  let politeMessage = '';
  let assertiveMessage = '';
  
  export function announce(message, priority = 'polite') {
    if (priority === 'assertive') {
      assertiveMessage = message;
      setTimeout(() => assertiveMessage = '', 1000);
    } else {
      politeMessage = message;
      setTimeout(() => politeMessage = '', 1000);
    }
  }
  
  // Export for global use
  onMount(() => {
    window.announcer = { announce };
  });
</script>

<!-- Screen reader only announcements -->
<div class="sr-only" aria-live="polite" aria-atomic="true">
  {politeMessage}
</div>

<div class="sr-only" aria-live="assertive" aria-atomic="true">
  {assertiveMessage}
</div>

<style>
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  }
</style>
```

### Modal Dialog Pattern

```svelte
<!-- lib/components/AccessibleModal.svelte -->
<script>
  import { createEventDispatcher } from 'svelte';
  import FocusTrap from './FocusTrap.svelte';
  
  export let open = false;
  export let title = '';
  
  const dispatch = createEventDispatcher();
  let dialogElement;
  
  $: if (open && dialogElement) {
    // Announce to screen readers
    dialogElement.focus();
  }
  
  function close() {
    open = false;
    dispatch('close');
  }
</script>

{#if open}
  <div
    class="modal-backdrop"
    on:click={close}
    aria-hidden="true"
  />
  
  <FocusTrap active={open}>
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      aria-describedby="modal-description"
      bind:this={dialogElement}
      tabindex="-1"
      class="modal"
    >
      <header>
        <h2 id="modal-title">{title}</h2>
        <button
          aria-label="Close dialog"
          on:click={close}
        >
          <span aria-hidden="true">×</span>
        </button>
      </header>
      
      <div id="modal-description">
        <slot />
      </div>
      
      <footer>
        <slot name="footer">
          <button on:click={close}>Close</button>
        </slot>
      </footer>
    </div>
  </FocusTrap>
{/if}
```

### Form Accessibility

```svelte
<!-- lib/components/AccessibleForm.svelte -->
<script>
  export let errors = {};
  
  let formElement;
  
  function handleSubmit(e) {
    // Validate form
    const formData = new FormData(e.target);
    
    // If errors, announce to screen reader
    if (Object.keys(errors).length > 0) {
      announceErrors();
      focusFirstError();
    }
  }
  
  function announceErrors() {
    const count = Object.keys(errors).length;
    const message = `${count} error${count > 1 ? 's' : ''} in form`;
    window.announcer?.announce(message, 'assertive');
  }
  
  function focusFirstError() {
    const firstErrorField = formElement.querySelector('[aria-invalid="true"]');
    firstErrorField?.focus();
  }
</script>

<form
  bind:this={formElement}
  on:submit|preventDefault={handleSubmit}
  novalidate
  aria-label="Contact form"
>
  <div class="field">
    <label for="name">
