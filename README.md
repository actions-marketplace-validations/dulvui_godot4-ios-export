# godot4-ios-export
Github Action to export a Godot Engine 4.x game to iOS.  
If you are facing problems with the action or this README feels incomplete, pull requests are welcome or open an issue.

# Table of contents
- [godot4-ios-export](#godot4-ios-export)
- [Table of contents](#table-of-contents)
- [Requirements](#requirements)
- [Parameters](#parameters)
- [How to use](#how-to-use)
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
```yml
- name: Export iOS
  uses: dulvui/godot4-ios-export@v1
  with:
    godot-version: 4.3
```

This creates a XCode project in the destination directory defined in your `exports_presets.cfg` file.  
The exported project can then be built and uploaded to the Apple App Store's Testflight
```yml
- name: Extract Provisioning profile UUID and create PP_UUID env variable
  run: echo "PP_UUID=$(grep -a -A 1 'UUID' $PPROFILE_PATH | grep string | sed -e "s|<string>||" -e "s|</string>||" | tr -d '\t')" >> $GITHUB_ENV

- name: Resolve package dependencies
  run: xcodebuild -resolvePackageDependencies

- name: Build the xarchive
  run: |
    set -eo pipefail
    xcodebuild  clean archive \
      -scheme $PROJECT_NAME \
      -configuration "Release" \
      -sdk iphoneos \
      -archivePath "$PWD/build/$PROJECT_NAME.xcarchive" \
      -destination "generic/platform=iOS,name=Any iOS Device" \
      OTHER_CODE_SIGN_FLAGS="--keychain $RUNNER_TEMP/app-signing.keychain-db" \
      CODE_SIGN_STYLE=Manual \
      PROVISIONING_PROFILE=$PP_UUID \
      CODE_SIGN_IDENTITY="Apple Distribution"

- name: Export .ipa
  run: |
    set -eo pipefail
    xcodebuild -archivePath "$PWD/build/$PROJECT_NAME.xcarchive" \
      -exportOptionsPlist exportOptions.plist \
      -exportPath $PWD/build \
      -allowProvisioningUpdates \
      -exportArchive

- name: Publish the App on TestFlight
  if: success()
  run: |
    xcrun altool \
      --upload-app \
      -t ios \
      -f $PWD/build/*.ipa \
      -u "${{ secrets.APPLE_ID_USERNAME }}" \
      -p "${{ secrets.APPLE_ID_PASSWORD }}" \
      --verbose

```

# License
This software is licensed under the [MIT license](LICENSE).
