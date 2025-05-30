# godot4-ios-export
Github Action to export a Godot Engine 4.x game to iOS.  
If you are facing problems with the action or this README feels incomplete, pull requests are welcome or open an issue.

# Table of contents
- [Requirements](#requirements)
- [Parameters](#parameters)
- [How to use](#how-to-use)
- [Example](#example)
- [License](#license)

# Requirements
 - Godot Engine project with a iOS config in your `exports_presets.cfg` file

# Parameters
| key | required | default | description |
| ----|----------|---------|-------------|
| godot-version | true | . | Godot Engine version. Supported are 4.x versions. Check versions [here](https://github.com/godotengine/godot-builds/releases) |
| godot-channel | false | stable | Godot Engine release channel (stable, beta, rc1, rc2, rc3...). Defaults to 'stable' Check release channels [here](https://github.com/godotengine/godot-builds/releases) |
| working-directory | false | . | Path to .project file |


# How to use
This is the minimal example on how to use the action.
This creates a XCode project in the destination directory defined in your `exports_presets.cfg` file.  
```yml
- name: Export iOS
  uses: dulvui/godot4-ios-export@v1
  with:
    godot-version: 4.3
```

# Example
The exported project can then be built and uploaded to the Apple App Store's Testflight.  

```yml
# SPDX-FileCopyrightText: 2023 Simon Dalvai <info@simondalvai.org>

# SPDX-License-Identifier: CC0-1.0

name: iOS upload

on:
  push:
    # paths:
    #   - ".github/workflows/upload-ios.yml"
    #   - "exportOptions.plist"
    #   - "export_presets.ios.example"

env:
  GODOT_VERSION: 4.3
  PROJECT_NAME: FutsalManager
  WORKING_DIRECTORY: game
  BUILD_CERTIFICATE_BASE64: ${{ secrets.IOS_BUILD_CERTIFICATE_BASE64 }}
  P12_PASSWORD: ${{ secrets.IOS_P12_PASSWORD }}
  BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.IOS_PROVISION_PROFILE_BASE64 }}
  KEYCHAIN_PASSWORD: ${{ secrets.IOS_KEYCHAIN_PASSWORD }}

jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: macos-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v3

      # https://docs.github.com/en/actions/deployment/deploying-xcode-applications/installing-an-apple-certificate-on-macos-runners-for-xcode-development
      - name: Install the Apple certificate and provisioning profile
        run: |
          # create variables
          CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

          # import certificate and provisioning profile from secrets
          echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
          echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

          # create temporary keychain
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH

          # import certificate to keychain
          security import $CERTIFICATE_PATH -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH

          # apply provisioning profile
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles

      - name: Create export_presets.cfg
        run: cp game/export_presets.ios.example game/export_presets.cfg

      - name: Extract Provisioning profile UUID and create GODOT_IOS_PROVISIONING_PROFILE_UUID_RELEASE env variable
        run: |
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          echo "GODOT_IOS_PROVISIONING_PROFILE_UUID_RELEASE=$(grep -a -A 1 'UUID' $PP_PATH | grep string | \
                sed -e "s|<string>||" -e "s|</string>||" | tr -d '\t')" >> $GITHUB_ENV

      - name: Export XCode project
        uses: dulvui/godot4-ios-export@v1
        env:
          CODE_SIGN_IDENTITY: "Apple Distribution"
        with:
          working-directory: $WORKING_DIRECTORY
          godot-version: $GODOT_VERSION

      - name: Publish the App on TestFlight
        if: success()
        run: |
          xcrun altool \
            --upload-app \
            -t ios \
            -f *.ipa \
            -u "${{ secrets.IOS_APPLE_ID_USERNAME }}" \
            -p "${{ secrets.IOS_APPLE_ID_PASSWORD }}" \
            --verbose
```

# License
This software is licensed under the [MIT license](LICENSE).
