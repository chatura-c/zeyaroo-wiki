# Configuration Guide

This directory contains centralized application configuration.

## Files

- **[app.ts](app.ts)** - Main configuration module that handles all environment variables
- **[firebase.ts](firebase.ts)** - Firebase initialization (uses app.ts for config)

## How It Works

### Automatic Environment Detection

The config automatically detects whether you're in development or production:

- **Development** (`npm run dev`): Uses `http://localhost:8080/api/v1`
- **Production** (`npm run build`): Uses `https://api.bytkloud.com/api/v1`

### Override with Environment Variables

You can override the default API URL by setting `VITE_API_URL` in your `.env` file:

```bash
# Use a custom API URL
VITE_API_URL=https://staging.api.bytkloud.com/api/v1
```

### CI/CD Usage

For CI/CD environments where setting env vars in files is difficult, you can:

1. **Use the defaults** - Just build with `npm run build`, it will automatically use production URLs
2. **Set environment variables** - Set `VITE_API_URL` as an environment variable in your CI/CD pipeline
3. **Use .env.production** - Vite automatically loads this file for production builds

## Usage in Code

Import configuration values from `app.ts`:

```typescript
import { apiConfig, appConfig, firebaseConfig, env } from '../config/app';

// API base URL
console.log(apiConfig.baseUrl);

// App name
console.log(appConfig.name);

// Check environment
if (env.isDevelopment) {
  // Development-only code
}
```

## Environment Variables

All environment variables must be prefixed with `VITE_` to be exposed to the frontend.

### API Configuration
- `VITE_API_URL` - Override default API URL
- `VITE_API_TIMEOUT` - API request timeout in milliseconds (default: 30000)

### App Branding
- `VITE_APP_NAME` - Application name
- `VITE_APP_TAGLINE` - Application tagline
- `VITE_APP_DOMAIN` - Application domain
- `VITE_APP_LOGO_URL` - URL to logo image

### Firebase
- `VITE_FIREBASE_API_KEY`
- `VITE_FIREBASE_AUTH_DOMAIN`
- `VITE_FIREBASE_PROJECT_ID`
- `VITE_FIREBASE_STORAGE_BUCKET`
- `VITE_FIREBASE_MESSAGING_SENDER_ID`
- `VITE_FIREBASE_APP_ID`
- `VITE_FIREBASE_USE_EMULATOR` - Set to "true" to use Firebase emulator in development

## CI/CD Example

### GitHub Actions

```yaml
- name: Build Frontend
  env:
    # Option 1: Use defaults (recommended)
    # No env vars needed, will use production defaults

    # Option 2: Override if needed
    VITE_API_URL: https://api.bytkloud.com/api/v1
  run: |
    cd frontend
    npm ci
    npm run build
```

### Docker Build

```dockerfile
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

# Build uses production defaults automatically
RUN npm run build

# Or override if needed:
# RUN VITE_API_URL=https://api.bytkloud.com/api/v1 npm run build
```

## Debug Configuration

In development mode, the config will log its values to the console when the app starts. Check the browser console to verify your configuration.
