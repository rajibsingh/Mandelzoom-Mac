# REQUIREMENTS.md
## iOS Swift Port of Mandelbrot Set Explorer

### Project Overview
Create an iOS application in Swift that renders and explores the Mandelbrot set with interactive zooming capabilities. The app should provide touch-based navigation, GPU-accelerated rendering, bookmark management, and settings customization optimized for iPhone and iPad interfaces.

### Target Platform
- **iOS**: 15.0+ (to support modern Metal features and SwiftUI)
- **Devices**: iPhone and iPad (Universal app with adaptive UI)
- **Language**: Swift 5.7+
- **Architecture**: SwiftUI + UIKit hybrid approach
- **Frameworks**: SwiftUI, Metal, CoreData (for bookmarks), Foundation, UIKit

### Core Components

#### 1. Rendering Engine (`MandelbrotRenderer`)
**Purpose**: Mathematical computation and GPU/CPU rendering of the Mandelbrot set

**Key Requirements**:
- Swift implementation of complex number mathematics using `simd` framework
- Metal compute shader integration for GPU acceleration on A-series chips
- CPU fallback using concurrent queues (`DispatchQueue.concurrentPerform`)
- Support for high precision coordinates (Double precision minimum)
- Configurable iteration limits and escape radius
- Real-time rendering performance optimization
- Coordinate system: complex plane mapping to screen pixels

**Properties**:
- `bottomLeft: Complex<Double>` - Bottom-left complex coordinate
- `topRight: Complex<Double>` - Top-right complex coordinate  
- `metalDevice: MTLDevice?` - Metal GPU device
- `commandQueue: MTLCommandQueue?` - Metal command queue
- `computePipeline: MTLComputePipelineState?` - Compiled shader
- `useGPUAcceleration: Bool` - GPU vs CPU rendering toggle
- `maxIterations: Int` - Maximum iteration count
- `escapeRadius: Double` - Escape threshold

**Methods**:
- `render() -> UIImage`
- `render(width: Int, height: Int) -> UIImage`
- `renderAsync(completion: @escaping (UIImage, TimeInterval) -> Void)`

#### 2. Interactive View (`MandelbrotView`)
**Purpose**: Touch-based interaction and display of the fractal

**Key Requirements**:
- SwiftUI Canvas or UIKit custom view for optimal performance
- Multi-touch gesture support:
  - **Pinch-to-zoom**: Magnify/reduce view with gesture center as focus
  - **Pan**: Drag to translate view across complex plane
  - **Double-tap**: Quick zoom (2x) at tap location
  - **Two-finger double-tap**: Quick zoom out (0.5x)
  - **Long-press**: Context menu with bookmark/share options
- Real-time coordinate display during interaction
- Smooth animation transitions between zoom states
- Haptic feedback for gesture actions
- Support for both iPhone and iPad screen sizes
- Adaptive UI for different device orientations

**UI Components**:
- `ZStack` containing:
  - `Image` view for fractal display
  - Overlay info panel (toggleable)
  - Coordinate display labels
  - Render time indicator
  - Loading spinner during computation

#### 3. Bookmark System
**Purpose**: Save, organize, and restore interesting Mandelbrot locations

**Models**:
- `MandelbrotBookmark` (Core Data entity):
  - `id: UUID`
  - `title: String`
  - `description: String?`
  - `xMin, xMax, yMin, yMax: Double`
  - `dateCreated: Date`
  - `thumbnail: Data?` (optional small image)
  - Computed properties: `centerX`, `centerY`, `width`, `height`, `magnification`

**Controllers**:
- `BookmarkManager` (ObservableObject):
  - Core Data persistence layer
  - CRUD operations for bookmarks
  - Export/import functionality (JSON format)
  - CloudKit sync support (future enhancement)

**Views**:
- `AddBookmarkView`: SwiftUI sheet for creating new bookmarks
- `BookmarkListView`: NavigationView with list of saved bookmarks
- `BookmarkDetailView`: Detailed view with edit/delete options
- `BookmarkExportView`: Share sheet for exporting selected bookmarks

#### 4. Settings System
**Purpose**: User preferences and app configuration

**Settings Options**:
- **Rendering**:
  - Max iterations (50-10000 range with slider)
  - GPU acceleration toggle (with device capability detection)
  - Image quality/resolution multiplier
- **Interface**:
  - Show coordinate info panel (toggle)
  - Show render time (toggle)
  - Haptic feedback (toggle)
  - Color scheme preference (system/light/dark)
- **Navigation**:
  - Zoom sensitivity multiplier
  - Pan sensitivity multiplier
  - Animation duration preference
- **Storage**:
  - Save location for exported images
  - Bookmark sync preferences

**Implementation**:
- `UserDefaults` for simple preferences
- `@AppStorage` property wrappers in SwiftUI views
- Settings bundle for system Settings app integration
- `NotificationCenter` for real-time preference updates

#### 5. Metal Compute Shader (`MandelbrotShader.metal`)
**Purpose**: High-performance GPU computation on iOS devices

**Requirements**:
- Port existing Metal shader to iOS Metal Shading Language
- Optimize for A-series GPU architectures (A12+)
- Support for different tile sizes based on device capabilities
- Early bailout optimizations:
  - Main cardioid detection
  - Period-2 bulb detection  
  - Periodic cycle detection
- Enhanced color mapping with smooth gradients
- Memory-efficient buffer management
- Support for different precision levels based on zoom depth

**Shader Parameters**:
```metal
struct MandelbrotParams {
    float2 bottomLeft;
    float2 topRight;
    uint width;
    uint height;
    uint maxIterations;
    float escapeRadius;
    float colorScale;
}
```

### User Interface Requirements

#### Navigation & Layout
- **iPhone**:
  - Single view controller with modal sheets for settings/bookmarks
  - Tab bar for main sections: Explore, Bookmarks, Settings
  - Compact toolbar with essential controls
- **iPad**:
  - Split view controller with sidebar for bookmarks/settings
  - Larger toolbar with expanded controls
  - Support for multiple windows (iPadOS 13+)

#### Touch Interactions
- **Primary Gestures**:
  - Pinch: Zoom in/out (minimum 1.1x, maximum limited by double precision)
  - Pan: Translate view smoothly
  - Double-tap: Quick zoom in (2x at point)
  - Two-finger double-tap: Quick zoom out (0.5x)
- **Secondary Gestures**:
  - Long-press: Context menu (bookmark, share, reset)
  - Three-finger tap: Reset to initial view
  - Edge swipe: Quick access to bookmarks (iPad only)

#### Information Display
- **Info Panel** (toggleable overlay):
  - Current coordinate ranges (X: min→max, Y: min→max)
  - Magnification level (1x to 1e15x+)
  - Real-time touch coordinates
  - Current render settings
- **Status Indicators**:
  - Render progress for long computations
  - Render time display (bottom corner)
  - GPU/CPU mode indicator
  - Network status for CloudKit sync

### Performance Requirements

#### Rendering Performance
- **Target Frame Rates**:
  - 60 FPS for UI interactions and animations
  - 30 FPS minimum during active rendering
  - Background rendering queue for complex calculations
- **Memory Management**:
  - Efficient image buffer reuse
  - Automatic quality reduction for memory pressure
  - Progressive rendering for deep zooms
- **Device-Specific Optimizations**:
  - A12+ devices: Full GPU acceleration with high iteration counts
  - A10-A11 devices: Balanced GPU/CPU with moderate settings
  - Older devices: CPU-focused with reduced default settings

#### Battery Optimization
- **Power Management**:
  - Reduce rendering quality when battery < 20%
  - Pause background rendering when app enters background
  - Thermal throttling awareness
- **Efficient Rendering**:
  - Adaptive tile sizes based on device performance
  - Skip redundant calculations for identical views
  - Use lower precision for initial preview renders

### Data Management

#### Bookmark Persistence
- **Core Data Stack**:
  - Local SQLite database with Core Data
  - Automatic migration support for schema updates
  - Background context for imports/exports
- **Export/Import**:
  - JSON format compatible with macOS version
  - Share extension support for easy sharing
  - Backup/restore functionality via Files app
- **Future CloudKit Integration**:
  - Sync bookmarks across user devices
  - Conflict resolution for concurrent edits
  - Privacy-focused (user data stays in personal iCloud)

### Accessibility & Localization

#### Accessibility Support
- **VoiceOver Support**:
  - Descriptive labels for all UI elements
  - Coordinate announcements during navigation
  - Gesture alternatives for vision-impaired users
- **Dynamic Type**: 
  - Support for larger text sizes
  - Scalable UI elements
- **High Contrast**:
  - Alternative color schemes for better visibility
  - Enhanced border visibility modes
- **Motor Accessibility**:
  - Adjustable gesture sensitivity
  - Alternative input methods support

#### Localization
- **Supported Languages** (Phase 1):
  - English (base)
  - Spanish
  - French
  - German
  - Japanese
  - Chinese (Simplified/Traditional)
- **Localizable Content**:
  - All UI strings and labels
  - Number/coordinate formatting
  - Date/time formatting for bookmarks
  - Help/tutorial content

### Testing Requirements

#### Unit Testing
- **Mathematical Accuracy**:
  - Mandelbrot iteration calculations
  - Complex number arithmetic precision
  - Coordinate transformations
- **Data Layer**:
  - Bookmark CRUD operations
  - Core Data model validation
  - Import/export functionality
- **Performance Testing**:
  - Rendering speed benchmarks
  - Memory usage validation
  - Battery impact measurement

#### UI Testing
- **Gesture Recognition**:
  - Multi-touch interaction accuracy
  - Animation smoothness validation
  - Edge case handling (simultaneous gestures)
- **Accessibility Testing**:
  - VoiceOver navigation flows
  - Dynamic type compatibility
  - High contrast mode validation

### Development Phases

#### Phase 1: Core Functionality (MVP)
1. Basic SwiftUI interface with fractal display
2. Metal shader implementation and GPU rendering  
3. Touch gesture support (pan, pinch, tap)
4. Simple bookmark system with local storage
5. Basic settings (iterations, GPU toggle)

#### Phase 2: Enhanced Features
1. Advanced UI with info panels and animations
2. Comprehensive bookmark management
3. Export/import functionality
4. Performance optimizations and battery management
5. iPad-specific UI enhancements

#### Phase 3: Polish & Distribution
1. Accessibility implementation
2. Localization for target languages
3. App Store optimization
4. CloudKit integration for bookmark sync
5. Advanced color schemes and customization

### Technical Specifications

#### Minimum System Requirements
- **iOS**: 15.0+
- **Storage**: 50MB app size, 500MB for extensive bookmark collections
- **RAM**: 2GB minimum (iPhone 7+), 3GB recommended
- **GPU**: A10+ for optimal performance, A8+ supported with limitations

#### Dependencies
- **Apple Frameworks**:
  - SwiftUI (primary UI)
  - Metal (GPU compute)
  - MetalKit (Metal integration helpers)
  - Core Data (bookmark persistence)
  - simd (vector mathematics)
  - UniformTypeIdentifiers (file handling)
- **Third-party Libraries**: None (pure Apple ecosystem approach)

#### File Structure
```
MandelbrotiOS/
├── App/
│   ├── MandelbrotiOSApp.swift
│   ├── ContentView.swift
│   └── AppDelegate.swift
├── Views/
│   ├── MandelbrotView.swift
│   ├── BookmarkViews/
│   ├── SettingsViews/
│   └── Components/
├── Models/
│   ├── MandelbrotBookmark.swift
│   ├── Settings.swift
│   └── CoreDataModels.xcdatamodeld
├── Services/
│   ├── MandelbrotRenderer.swift
│   ├── BookmarkManager.swift
│   └── SettingsManager.swift
├── Shaders/
│   └── MandelbrotShader.metal
├── Resources/
│   ├── Localizations/
│   └── Assets.xcassets
└── Tests/
    ├── Unit/
    └── UI/
```

### Success Criteria
1. **Performance**: Smooth 60fps interactions on target devices
2. **Usability**: Intuitive touch navigation matching platform conventions
3. **Quality**: Mathematical accuracy matching desktop version precision  
4. **Compatibility**: Universal support across iPhone/iPad form factors
5. **Store Readiness**: App Store guidelines compliance and optimization