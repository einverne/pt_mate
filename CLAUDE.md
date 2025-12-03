# CLAUDE.md

## Project Overview

PT Mate is a Flutter-based private tracker torrent management application supporting multiple PT sites (M-Team, NexusPHP). It features torrent browsing, searching, favorites, and integration with qBittorrent and Transmission downloaders.

## Development Commands

### Flutter Commands

```bash
# Install dependencies
flutter pub get

# Run in debug mode
flutter run

# Run in release mode
flutter run --release

# Build APK (debug)
flutter build apk --debug

# Build APK (release)
flutter build apk --release

# Run tests
flutter test

# Run specific test
flutter test test/nexusphp_adapter_test.dart

# Code analysis
flutter analyze
```

### Site Configuration Management

```bash
# Generate sites manifest after adding new site configs
./generate_sites_manifest.sh

# This script must be run when:
# - Adding new site configuration files to assets/sites/
# - Modifying existing site configurations
# - The script generates assets/sites_manifest.json
```

### Backend Server (Go)

```bash
cd server

# Build server
./build.sh

# Run server (requires .env file)
go run .
```

## Architecture Overview

### Adapter Pattern for Multi-Site Support

The application uses an adapter pattern to support different PT site types through a unified interface:

- **`SiteAdapter`** (abstract): Defines the contract all site implementations must follow
- **`MTeamAdapter`**: M-Team official API implementation
- **`NexusPHPAdapter`**: NexusPHP framework API implementation
- **`NexusPHPWebAdapter`**: Web scraping implementation for sites without API support
- **`SiteAdapterFactory`**: Creates appropriate adapter based on site type

This architecture allows adding new site types without modifying existing code - implement `SiteAdapter` interface and register in the factory.

### Downloader Integration Architecture

Similar adapter pattern for downloader clients:

- **`DownloaderClient`** (abstract): Unified interface for all downloader types
- **`QBittorrentClient`**: qBittorrent Web API implementation
- **`TransmissionClient`**: Transmission RPC implementation
- **`DownloaderFactory`**: Creates appropriate client based on configuration

All downloader operations (add task, pause, resume, delete, get status) go through this abstraction layer.

### State Management

- **Provider pattern** for app-wide state management
- **`AppState`**: Central state holder for active site configuration and initialization status
- **`AggregateSearchProvider`**: Manages multi-site search state and results
- **Notifier pattern**: State changes automatically propagate to UI

### Storage Architecture

- **`StorageService`**: Singleton service managing all local data persistence
  - Uses `SharedPreferences` for general configuration
  - Uses `FlutterSecureStorage` for sensitive data (API keys, passwords)
  - Implements data migration system for version upgrades
- **`BackupService`**: Handles backup/restore with WebDAV support

### Site Configuration System

Site configurations are loaded from JSON files in `assets/sites/`:

1. **Site manifest** (`assets/sites_manifest.json`): Lists all available site configs
2. **Individual site configs** (`assets/sites/*.json`): Each site has its own JSON with:
   - Basic info (id, name, URLs, type)
   - Feature flags (what the site supports)
   - For NexusPHPWeb: `infoFinder` configurations defining CSS selectors for data extraction
3. **Default templates** (`assets/site_configs.json`): Fallback configurations

The `SiteConfigService` loads and merges these configurations at runtime.

### Image Loading with Authentication

- **`ImageHttpClient`**: Custom HTTP client extending Dio
- Automatically injects authentication cookies from site configuration
- Handles retry logic and error cases
- Used by `CachedNetworkImageWidget` for authenticated image loading

## Key Implementation Details

### Site Configuration Extraction (NexusPHPWeb)

The `infoFinder` system in NexusPHPWeb adapter uses two selector types:

1. **PTM Selector** (default): Custom selector supporting:
   - Cross-hierarchy selection
   - Attribute filtering with `^=`, `~=`, `==` operators
   - Special keywords: `next`, `prev`, `parent`, `nth-child`

2. **CSS Selector** (prefix with `@@`): Standard CSS selectors

Fields are extracted in two steps:
- `rows.selector`: Find container elements (can match multiple, e.g., torrent list items)
- `fields.*.selector`: Extract specific data from each row using attribute/text extraction

### Download URL Construction

Download URLs are constructed based on site configuration:
- Default pattern: `download.php?downhash={userId}.{jwt}`
- Customizable via `infoFinder.search.fields.downloadUrl.value`
- Supports variables: `{baseUrl}`, `{torrentId}`, `{passKey}`, `{userId}`, `{jwt}`
- Prefix with `##` to force local relay mode (download .torrent first, then push to downloader)

### Testing Strategy

Tests focus on adapter implementations:
- `nexusphp_adapter_test.dart`: Tests NexusPHP API adapter
- `nexusphp_web_adapter_test.dart`: Tests web scraping adapter
- Widget tests for UI components

## Adding New Site Support

See [SITE_CONFIGURATION_GUIDE.md](./SITE_CONFIGURATION_GUIDE.md) for detailed instructions. Quick overview:

1. Create JSON config in `assets/sites/newsite.json`
2. Set `siteType` to `M-Team`, `NexusPHP`, or `NexusPHPWeb`
3. For NexusPHPWeb: Configure `infoFinder` with selectors for data extraction
4. Run `./generate_sites_manifest.sh`
5. Test the site configuration

## Important Patterns

### Secure Storage

Never log or expose sensitive information:
- API keys, passkeys, passwords stored via `FlutterSecureStorage`
- Cookies handled internally by adapters
- Use debug logging service that filters sensitive data

### Error Handling

- All API calls should handle network errors gracefully
- Use try-catch blocks in adapter implementations
- Return meaningful error messages to UI layer
- Log errors for debugging but never log sensitive data

### Internationalization

The app uses `flutter_localizations` for i18n support. When adding new strings, ensure they're properly externalized for translation.

### Platform-Specific Code

Be aware of platform differences:
- Web builds cannot use local file system for torrent relay
- Desktop platforms have different secure storage implementations
- Check `kIsWeb` or `Platform.is*` before platform-specific operations
