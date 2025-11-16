# Accessibility Testing - SvelteKit PWAs

**Automated and manual accessibility testing strategies**


## Testing Accessibility

### Automated Testing

```javascript
// tests/a11y.test.js
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
  test('homepage should have no accessibility violations', async ({ page }) => {
    await page.goto('/');
    
    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();
    
    expect(accessibilityScanResults.violations).toEqual([]);
  });
  
  test('navigation is keyboard accessible', async ({ page }) => {
    await page.goto('/');
    
    // Tab through navigation
    await page.keyboard.press('Tab');
    const firstLink = await page.evaluate(() => 
      document.activeElement?.tagName
    );
    expect(firstLink).toBe('A');
    
    // Test Enter key navigation
    await page.keyboard.press('Enter');
    await expect(page).toHaveURL(/.*about/);
  });
  
  test('form shows error messages to screen readers', async ({ page }) => {
    await page.goto('/contact');
    
    // Submit empty form
    await page.click('button[type="submit"]');
    
    // Check for aria-invalid
    const invalidFields = await page.$$('[aria-invalid="true"]');
    expect(invalidFields.length).toBeGreaterThan(0);
    
    // Check for error messages with role="alert"
    const alerts = await page.$$('[role="alert"]');
    expect(alerts.length).toBeGreaterThan(0);
  });
});
```

### Manual Testing Checklist

- [ ] Navigate entire site using only keyboard
- [ ] All interactive elements reachable via Tab
- [ ] Focus indicators clearly visible
- [ ] Escape key closes modals/menus
- [ ] Screen reader announces all content
- [ ] Images have appropriate alt text
- [ ] Form labels properly associated
- [ ] Error messages announced
- [ ] Color not sole indicator of meaning
- [ ] Contrast ratios meet WCAG standards
- [ ] Text resizable to 200% without horizontal scroll
- [ ] No keyboard traps
- [ ] Skip links functional
- [ ] Landmarks properly labeled
- [ ] Heading hierarchy logical

## Best Practices

1. **Start with semantic HTML** - Use proper elements before adding ARIA
2. **Test with real assistive technology** - Use screen readers and keyboard navigation
3. **Don't remove focus indicators** - Style them instead of removing
4. **Provide text alternatives** - For all non-text content
5. **Ensure sufficient contrast** - Test with automated tools
6. **Make touch targets large enough** - Minimum 44x44px
7. **Support user preferences** - Respect reduced motion, high contrast
8. **Test early and often** - Integrate a11y testing in CI/CD

## References

- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [WebAIM Resources](https://webaim.org/resources/)
- [Inclusive Components](https://inclusive-components.design/)
