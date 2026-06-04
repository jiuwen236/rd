# MEMORY

## Git Changes Summary

This fork renames the project from "RustDesk" to "rd" with bundle ID "com.rd", and builds with the Flutter UI on macOS.

### Modified Files
- **Cargo.toml**: package name → `rd`, identifier → `com.rd`, `tray-icon` dep bumped to available version
- **build.py**: added `get_package_name()` / `get_bundle_identifier()` to read from Cargo.toml; `build_flutter_dmg()` uses dynamic app name; added `codesign --force --deep --sign -` after flutter build
- **flutter/lib/models/native_model.dart, platform_model.dart, web_model.dart**: `RustdeskImpl` → `RdImpl`
- **flutter/lib/web/bridge.dart**: `class RustdeskImpl` → `class RdImpl`
- **flutter/run.sh**: added `--class-name Rd` to `flutter_rust_bridge_codegen`
- **flutter/macos/Runner/Configs/AppInfo.xcconfig**: `PRODUCT_NAME = rd`, `PRODUCT_BUNDLE_IDENTIFIER = com.rd`
- **flutter/macos/Runner.xcodeproj/project.pbxproj**: all bundle IDs → `com.rd`
- **flutter/macos/Runner/Info.plist**: CFBundleURLName → `$(PRODUCT_BUNDLE_IDENTIFIER)`
- **flutter/ios/** (project.pbxproj, Info.plist, exportOptions.plist, GoogleService-Info.plist): bundle IDs → `com.rd`
- **res/**: all app icons (`.ico`, key `.png`, key `.svg`) replaced with `circle.png` / `circle.svg`
- **flutter/macos/Runner/AppIcon.icns**: regenerated from `res/circle.png`
- **flutter/assets/icon.svg**: replaced with `res/circle.svg`
- **flutter/android/app/src/main/res/mipmap-*/**: launcher and status icons replaced with `res/circle.png`
- **flutter/ios/Runner/Assets.xcassets/**: AppIcon and LaunchImage pngs replaced with `res/circle.png`
- **flutter/windows/runner/resources/app_icon.ico**: regenerated from `res/circle.png`

## macOS Deployment Guide

### 1. Install Prerequisites
```bash
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
export PATH=$HOME/.cargo/bin:$PATH
. "$HOME/.cargo/env"

# Full Xcode (from App Store), then point xcode-select to it
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer

# LLVM (needed by Dart ffigen for libclang)
brew install llvm

# vcpkg dependencies
brew install nasm yasm cmake gcc wget ninja pkg-config create-dmg
git clone https://github.com/microsoft/vcpkg
export VCPKG_ROOT=$PWD/vcpkg
$VCPKG_ROOT/bootstrap-vcpkg.sh
$VCPKG_ROOT/vcpkg install --x-install-root="$VCPKG_ROOT/installed"
```

### 2. Flutter SDK (3.24.3 via FVM)
```bash
dart pub global activate fvm   # or: brew install fvm
cd /Users/bytedance/Documents/pp/rd/flutter
fvm install 3.24.3
fvm use 3.24.3
export PATH="$PWD/.fvm/flutter_sdk/bin:$PATH"
flutter pub get
cd ..
```

### 3. Generate FFI Bridge
```bash
cargo install flutter_rust_bridge_codegen --version 1.80.1
flutter_rust_bridge_codegen \
  --rust-input ./src/flutter_ffi.rs \
  --dart-output ./flutter/lib/generated_bridge.dart \
  --class-name Rd
```

### 4. Build the App
```bash
export PATH=$HOME/.cargo/bin:$PATH
. "$HOME/.cargo/env"
cd /Users/bytedance/Documents/pp/rd
python3 build.py --flutter
```
build.py will: build the Rust dylib (release), run `flutter build macos --release`,
copy the `service` binary into the bundle, then ad-hoc sign with
`codesign --force --deep --sign -`.

### 5. Run
```bash
open flutter/build/macos/Build/Products/Release/rd.app
# or
flutter/build/macos/Build/Products/Release/rd.app/Contents/MacOS/rd
```

### Troubleshooting
- **`cargo: command not found`**: run `export PATH=$HOME/.cargo/bin:$PATH` first.
- **`Couldn't find lib/libclang.dylib`**: `brew install llvm`.
- **`xcrun: unable to find xcodebuild`**: install full Xcode + `xcode-select --switch`.
- **dyld Team ID mismatch on launch**: re-sign with
  `codesign --force --deep --sign - flutter/build/macos/Build/Products/Release/rd.app`.
- **app icon still shows old image on macOS**: clear build output and rebuild, then
  restart Finder/Dock or open a copied app bundle to bypass icon cache:
  `rm -rf flutter/build/macos && python3 build.py --flutter`
  `killall Finder; killall Dock`
