# macOS Release Workflow Setup Guide

This guide provides step-by-step instructions for setting up a GitHub Actions workflow to build, notarize, and release a macOS application as a DMG file.

## Prerequisites

- macOS Xcode project
- Apple Developer Account
- GitHub repository

## Step 1: Prepare Apple Developer Certificates and Profiles

### 1.1 Export Developer ID Certificate

1. Open **Keychain Access** on your Mac
2. Find your "Developer ID Application" certificate
3. Right-click and select **Export**
4. Save as `.p12` file with a password
5. Convert to base64:
   ```bash
   base64 -i DeveloperID.p12 | pbcopy
   ```
6. The base64 string is now in your clipboard

### 1.2 Create Provisioning Profile

1. Go to [Apple Developer Portal](https://developer.apple.com/account)
2. Navigate to **Certificates, Identifiers & Profiles**
3. Select **Profiles** → **+** (Create new)
4. Choose **Developer ID** profile type
5. Select your App ID
6. Select your Developer ID certificate
7. Download the `.provisionprofile` file
8. Convert to base64:
   ```bash
   base64 -i YourApp.provisionprofile | pbcopy
   ```

### 1.3 Generate App-Specific Password

1. Go to [appleid.apple.com](https://appleid.apple.com)
2. Sign in with your Apple ID
3. Navigate to **Security** → **App-Specific Passwords**
4. Click **Generate Password**
5. Give it a name (e.g., "GitHub Actions Notarization")
6. Copy the generated password

## Step 2: Create ExportOptions.plist

Create `ExportOptions.plist` in your Xcode project directory (e.g., `YourApp/YourApp/ExportOptions.plist`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>developer-id</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.yourcompany.yourapp</key>
        <string>YourApp Provisioning Profile Name</string>
    </dict>
</dict>
</plist>
```

Replace:
- `YOUR_TEAM_ID` with your Apple Team ID
- `com.yourcompany.yourapp` with your app's Bundle ID
- `YourApp Provisioning Profile Name` with your provisioning profile name

## Step 3: Configure GitHub Secrets

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add the following secrets:

| Secret Name | Value | Description |
|------------|-------|-------------|
| `DEVELOPER_ID_P12` | Base64 certificate | Base64-encoded .p12 certificate |
| `DEVELOPER_ID_P12_PASSWORD` | Certificate password | Password used when exporting .p12 |
| `[PROJECTNAME]_PROVISIONING_PROFILE` | Base64 profile | Base64-encoded .provisionprofile |
| `APPLE_ID` | your@email.com | Your Apple ID email |
| `APPLE_TEAM_ID` | XXXXXXXXXX | Your 10-character Team ID |
| `APP_SPECIFIC_PASSWORD` | xxxx-xxxx-xxxx-xxxx | App-specific password from Step 1.3 |

**Note:** Replace `[PROJECTNAME]` with your project name in uppercase (e.g., `MERMAIDO_PROVISIONING_PROFILE`).

## Step 4: Find Your Code Sign Identity

Run this command to find your exact code signing identity:

```bash
security find-identity -v -p codesigning
```

Look for a line like:
```
"Developer ID Application: Your Name (TEAM_ID)"
```

Copy the full identity string for use in the workflow.

## Step 5: Create Workflow File

Create `.github/workflows/macos-release.yml` in your repository:

```yaml
name: macOS Release

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: macos-26

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Set up Xcode
        run: sudo xcode-select -s /Applications/Xcode.app

      - name: Create build directory
        run: mkdir -p build

      - name: Import Developer ID Certificate
        run: |
          echo "${{ secrets.DEVELOPER_ID_P12 }}" | base64 --decode > build/DeveloperID.p12
          security create-keychain -p "" build.keychain
          security import build/DeveloperID.p12 -k build.keychain -P "${{ secrets.DEVELOPER_ID_P12_PASSWORD }}" -T /usr/bin/codesign
          security list-keychains -s build.keychain
          security unlock-keychain -p "" build.keychain
          security set-keychain-settings -t 3600 -l build.keychain

      - name: Configure keychain for codesign
        run: |
          security set-key-partition-list -S apple-tool:,apple: -k "" build.keychain

      - name: Install Provisioning Profile
        run: |
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "${{ secrets.PROJECTNAME_PROVISIONING_PROFILE }}" | base64 --decode > ~/Library/MobileDevice/Provisioning\ Profiles/ProjectName.provisionprofile

      - name: Build and Archive App
        run: |
          set -o pipefail
          xcodebuild -project "ProjectName/ProjectName.xcodeproj" \
            -scheme "ProjectName" \
            -configuration Release \
            -archivePath "$PWD/build/ProjectName.xcarchive" \
            -destination "platform=macOS,arch=arm64" \
            ARCHS=arm64 ONLY_ACTIVE_ARCH=YES \
            DEBUG_INFORMATION_FORMAT="dwarf" \
            ENABLE_HARDENED_RUNTIME=YES \
            CODE_SIGN_STYLE=Manual \
            DEVELOPMENT_TEAM="${{ secrets.APPLE_TEAM_ID }}" \
            CODE_SIGN_IDENTITY="Developer ID Application: Your Name (TEAM_ID)" \
            archive

      - name: Export App
        run: |
          xcodebuild -exportArchive \
            -archivePath "$PWD/build/ProjectName.xcarchive" \
            -exportOptionsPlist "$PWD/ProjectName/ProjectName/ExportOptions.plist" \
            -exportPath "$PWD/build"

      - name: Notarize app
        run: |
          ditto -c -k --keepParent "build/ProjectName.app" "build/ProjectName.zip"
          xcrun notarytool submit "build/ProjectName.zip" \
            --apple-id "${{ secrets.APPLE_ID }}" \
            --team-id "${{ secrets.APPLE_TEAM_ID }}" \
            --password "${{ secrets.APP_SPECIFIC_PASSWORD }}" \
            --wait
          xcrun stapler staple "build/ProjectName.app"

      - name: Create DMG
        run: |
          hdiutil create -volname "ProjectName" \
            -srcfolder "build/ProjectName.app" \
            -ov -format UDZO "build/ProjectName.dmg"

      - name: Upload DMG to Release
        uses: softprops/action-gh-release@v2
        with:
          files: build/ProjectName.dmg
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Customizations Required:

Replace the following placeholders:
- `PROJECTNAME_PROVISIONING_PROFILE` → Your secret name (e.g., `MERMAIDO_PROVISIONING_PROFILE`)
- `ProjectName` → Your project name (appears 12 times in the workflow)
- `ProjectName/ProjectName.xcodeproj` → Path to your Xcode project
- `"ProjectName"` (scheme) → Your Xcode scheme name
- `Developer ID Application: Your Name (TEAM_ID)` → Your code sign identity from Step 4
- `ProjectName/ProjectName/ExportOptions.plist` → Path to your ExportOptions.plist

## Step 6: Test the Workflow

### Test Manually

1. Go to your GitHub repository
2. Navigate to **Actions** tab
3. Select **macOS Release** workflow
4. Click **Run workflow** → **Run workflow**

### Test with Tag

```bash
git tag v1.0.0
git push origin v1.0.0
```

The workflow will automatically trigger and create a release with the DMG file attached.

## Troubleshooting

### Build Fails with Code Signing Error

- Verify your certificate and provisioning profile are correctly base64 encoded
- Check that `CODE_SIGN_IDENTITY` exactly matches your certificate name
- Ensure your provisioning profile matches your Bundle ID

### Notarization Fails

- Verify `APPLE_ID` is correct
- Ensure `APP_SPECIFIC_PASSWORD` is valid (not your regular password)
- Check that `APPLE_TEAM_ID` is your 10-character Team ID

### Export Fails

- Verify `ExportOptions.plist` exists at the specified path
- Check that Team ID and Bundle ID in `ExportOptions.plist` are correct
- Ensure provisioning profile name matches exactly

### Workflow Uses Wrong Xcode Version

- Update `runs-on: macos-26` to use a different macOS version if needed
- Check available runners at [GitHub Actions runner images](https://github.com/actions/runner-images)

## Additional Notes

- The workflow builds for Apple Silicon (arm64) only. To build universal binaries, modify the `ARCHS` parameter
- DMG files are automatically attached to GitHub Releases when tags are pushed
- The workflow runs on `macos-26` which may not be available yet; adjust to `macos-14` or `macos-15` as needed
- Keep your secrets secure and never commit them to your repository
