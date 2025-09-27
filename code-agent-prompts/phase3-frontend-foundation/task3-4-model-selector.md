# Task 3.4: Model Selector

## Objective
Create model selection interface for multi-model chat.

## Requirements
Implement:
- Model dropdown/selector component
- Model status indicators (available, loading, loaded)
- Model loading progress display
- Model switching within threads
- Default model preference from user settings
- Model-specific icons and descriptions
- Disabled state for unavailable models

## Model Selector Components
- ModelDropdown with available models list
- ModelStatus indicator component
- ModelLoadingProgress component
- ModelIcon component for provider branding
- ModelDescription with capabilities info

## Model Types and Status
```typescript
interface Model {
  id: string;
  name: string;
  provider: 'claude' | 'openai' | 'google' | 'ollama';
  type: 'remote' | 'local';
  description: string;
  status: ModelStatus;
  capabilities: string[];
}

interface ModelStatus {
  state: 'available' | 'loading' | 'loaded' | 'error' | 'unavailable';
  progress?: number;
  error?: string;
}
```

## Model Selection Flow
1. Display current selected model in thread
2. Click to open model selector dropdown
3. Show available models with status indicators
4. Allow selection of available models
5. Handle loading for local models
6. Switch model context within thread
7. Update UI to reflect model change

## Status Indicators
- **Available**: Green checkmark, ready to use
- **Loading**: Progress spinner with percentage
- **Loaded**: Blue indicator, ready for local models
- **Error**: Red warning with error message
- **Unavailable**: Grayed out, not selectable

## Model Loading Interface
- Progress bar for loading local models
- Time estimates for loading completion
- Cancel button for ongoing loads
- Retry option for failed loads
- Resource usage warnings

## Provider Branding
- Anthropic Claude: Purple accent color
- OpenAI: Green accent color
- Google AI: Blue/red accent colors
- Ollama: Orange accent color
- Provider logos/icons where appropriate

## State Management
```typescript
interface ModelStore {
  models: Model[];
  selectedModel: string | null;
  isLoading: boolean;
  loadingProgress: Record<string, number>;
  loadModels: () => Promise<void>;
  selectModel: (modelId: string) => void;
  loadModel: (modelId: string) => Promise<void>;
  getModelStatus: (modelId: string) => ModelStatus;
}
```

## User Experience Features
- Remember last used model per thread
- Default model from user preferences
- Quick switch between recent models
- Model capabilities tooltip
- Performance indicators (speed, quality)

## Accessibility
- Keyboard navigation support
- Screen reader compatibility
- High contrast mode support
- Focus indicators and ARIA labels
- Descriptive model information

## Deliverables
- Model selector dropdown component
- Status indicators for all model states
- Loading progress interface
- Provider branding and icons
- Model state management
- Accessibility compliance