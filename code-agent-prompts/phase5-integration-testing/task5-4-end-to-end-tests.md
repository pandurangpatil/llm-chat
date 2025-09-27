# Task 5.4: End-to-End Tests

## Objective
Build E2E test suite using Playwright.

## Requirements
Test:
- Complete user journey from login to chat
- Thread creation and messaging flow
- Model switching and streaming
- Settings update and persistence
- Error recovery scenarios
- Mobile and desktop responsive behavior
- Cross-browser compatibility

## Testing Framework Setup
- Playwright for E2E automation
- Test configuration for multiple browsers
- Page object model for maintainability
- Test data management and cleanup
- Screenshot and video recording
- Parallel test execution

## User Journey Tests
```typescript
describe('Complete User Journey', () => {
  test('user can login, create thread, and chat', async ({ page }) => {
    // Login flow
    await page.goto('/login');
    await page.fill('[data-testid="username"]', testUser.username);
    await page.fill('[data-testid="password"]', testUser.password);
    await page.click('[data-testid="login-button"]');

    // Verify dashboard loads
    await expect(page).toHaveURL('/chat');

    // Create new thread
    await page.click('[data-testid="new-thread-button"]');
    await page.fill('[data-testid="message-input"]', 'Hello, how are you?');
    await page.click('[data-testid="send-button"]');

    // Verify message appears and response streams
    await expect(page.locator('[data-testid="user-message"]')).toBeVisible();
    await expect(page.locator('[data-testid="ai-response"]')).toBeVisible();
  });
});
```

## Authentication Flow Tests
- Login with valid credentials
- Login error handling
- Session persistence across page refresh
- Logout functionality
- Protected route redirection
- Session timeout handling

## Thread Management Tests
- Thread creation workflow
- Thread listing and pagination
- Thread selection and switching
- Thread deletion with confirmation
- Thread title editing
- Empty state handling

## Chat Functionality Tests
- Message sending and receiving
- Real-time streaming display
- Message history loading
- Model switching within thread
- Copy message functionality
- Retry failed messages

## Model Loading Tests
- Local model loading progress
- Model status indicators
- Loading cancellation
- Error handling for failed loads
- Background loading behavior
- Resource usage warnings

## Settings and Profile Tests
- Profile settings update
- API key management
- Default model configuration
- System prompt customization
- Settings persistence
- Validation error handling

## Error Recovery Tests
- Network disconnection scenarios
- Server error responses
- Timeout handling
- Retry mechanisms
- Graceful degradation
- User notification of errors

## Cross-Browser Testing
- Chrome/Chromium compatibility
- Firefox compatibility
- Safari compatibility (if applicable)
- Edge compatibility
- Feature detection and polyfills

## Responsive Design Tests
```typescript
describe('Mobile Responsiveness', () => {
  test('chat interface works on mobile', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await loginUser(page);

    // Test mobile navigation
    await page.click('[data-testid="mobile-menu-toggle"]');
    await expect(page.locator('[data-testid="thread-sidebar"]')).toBeVisible();

    // Test mobile chat interface
    await page.fill('[data-testid="message-input"]', 'Mobile test message');
    await page.click('[data-testid="send-button"]');
    await expect(page.locator('[data-testid="user-message"]')).toBeVisible();
  });
});
```

## Performance Testing
- Page load performance
- Chat message rendering speed
- Model loading time verification
- Memory usage monitoring
- Network request optimization
- Bundle size impact

## Page Object Model
```typescript
class ChatPage {
  constructor(private page: Page) {}

  async sendMessage(message: string) {
    await this.page.fill('[data-testid="message-input"]', message);
    await this.page.click('[data-testid="send-button"]');
  }

  async selectModel(modelId: string) {
    await this.page.click('[data-testid="model-selector"]');
    await this.page.click(`[data-testid="model-${modelId}"]`);
  }

  async waitForResponse() {
    await this.page.waitForSelector('[data-testid="ai-response"]');
  }
}
```

## Test Data Management
- User account creation and cleanup
- Thread and message test data
- API key test configuration
- Database state management
- Test isolation strategies

## Visual Regression Testing
- Screenshot comparison tests
- Component visual consistency
- Layout regression detection
- Theme and styling verification
- Icon and image rendering

## Accessibility Testing
- Keyboard navigation workflows
- Screen reader compatibility
- Focus management verification
- Color contrast validation
- ARIA attribute presence

## API Integration Testing
- Real API endpoint validation
- Error response handling
- Rate limiting behavior
- Authentication flow testing
- Data persistence verification

## Test Organization
```
e2e/
├── tests/
│   ├── auth.spec.ts
│   ├── chat.spec.ts
│   ├── threads.spec.ts
│   ├── models.spec.ts
│   ├── profile.spec.ts
│   └── mobile.spec.ts
├── pages/
│   ├── LoginPage.ts
│   ├── ChatPage.ts
│   ├── ThreadPage.ts
│   └── ProfilePage.ts
├── fixtures/
│   ├── users.json
│   └── test-data.json
└── utils/
    ├── test-helpers.ts
    └── cleanup.ts
```

## CI/CD Integration
- Automated E2E test execution
- Test result reporting
- Screenshot artifact collection
- Failed test video recording
- Performance metric tracking

## Deliverables
- Complete E2E test suite with Playwright
- Page object model implementation
- Cross-browser compatibility tests
- Mobile responsiveness validation
- Performance and accessibility testing
- CI integration for automated execution