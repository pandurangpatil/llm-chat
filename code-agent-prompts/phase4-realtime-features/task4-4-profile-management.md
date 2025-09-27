# Task 4.4: Profile Management

## Objective
Implement user profile and settings interface.

## Requirements
Build:
- Profile page/modal with user information
- Settings form for default model, temperature, system prompt
- API key management interface (add/update/remove)
- Encrypted key display with show/hide toggle
- Summarization interval configuration
- Save/cancel functionality
- Settings validation before save
- Success/error notifications

## Profile Components
- ProfileModal: Main settings container
- UserInfo: Display name and username
- ModelSettings: Default model and temperature
- SystemPrompt: Custom system prompt editor
- APIKeyManager: Manage provider API keys
- NotificationSettings: App notification preferences

## User Settings Schema
```typescript
interface UserSettings {
  defaultModel: string;
  defaultTemperature: number; // 0.0 - 1.0
  systemPrompt: string;
  summarizationInterval: number; // message count
  notifications: {
    modelLoading: boolean;
    messageFailed: boolean;
    systemUpdates: boolean;
  };
  ui: {
    theme: 'light' | 'dark' | 'system';
    fontSize: 'small' | 'medium' | 'large';
    compactMode: boolean;
  };
}
```

## Settings Form
- Form validation with real-time feedback
- Default value loading from user profile
- Unsaved changes warning
- Reset to defaults option
- Auto-save vs manual save modes
- Field-level validation and error display

## API Key Management
- Add new API keys for providers
- Update existing keys
- Remove/revoke API keys
- Encrypted display with show/hide toggle
- Key validation and testing
- Usage statistics per key

### API Key Interface
```typescript
interface APIKeyManager {
  keys: APIKey[];
  addKey: (provider: string, key: string) => Promise<void>;
  updateKey: (keyId: string, key: string) => Promise<void>;
  removeKey: (keyId: string) => Promise<void>;
  testKey: (keyId: string) => Promise<boolean>;
  toggleVisibility: (keyId: string) => void;
}

interface APIKey {
  id: string;
  provider: 'claude' | 'openai' | 'google';
  name: string;
  masked: string; // "sk-...abc123"
  isVisible: boolean;
  isValid: boolean;
  lastUsed?: number;
  createdAt: number;
}
```

## System Prompt Editor
- Multi-line text editor with syntax highlighting
- Placeholder suggestions and templates
- Character count and validation
- Preview functionality
- Save/revert changes
- Template library (optional)

## Model Settings
- Default model selection dropdown
- Temperature slider with visual feedback
- Token limit configuration
- Response time preferences
- Quality vs speed trade-offs

## Summarization Settings
- Message count threshold slider
- Auto-summarization toggle
- Summary length preferences
- Trigger conditions configuration

## Validation Rules
- Temperature: 0.0 to 1.0 range
- System prompt: max 2000 characters
- API keys: proper format validation
- Summarization interval: 5-50 messages
- Required field validation

## Notifications
- Success messages for saved settings
- Error messages for validation failures
- Warning for unsaved changes
- Confirmation for destructive actions
- Toast notifications for API key operations

## Security Features
- Masked API key display by default
- Secure key transmission and storage
- Key validation without exposure
- Audit log for key operations
- Session timeout for sensitive operations

## Accessibility
- Keyboard navigation throughout
- Screen reader compatibility
- Focus management in modals
- High contrast support
- Descriptive labels and help text

## State Management
```typescript
interface ProfileStore {
  settings: UserSettings;
  apiKeys: APIKey[];
  isLoading: boolean;
  hasUnsavedChanges: boolean;
  loadProfile: () => Promise<void>;
  updateSettings: (settings: Partial<UserSettings>) => Promise<void>;
  addAPIKey: (provider: string, key: string) => Promise<void>;
  removeAPIKey: (keyId: string) => Promise<void>;
  resetToDefaults: () => void;
}
```

## Deliverables
- Complete profile management interface
- Settings form with validation
- API key management system
- System prompt editor
- Notification system for user feedback
- Security measures for sensitive data