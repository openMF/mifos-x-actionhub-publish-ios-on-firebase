# KMP Publish iOS App on Firebase Action

This GitHub Action automates the process of building and publishing iOS applications to Firebase App Distribution. It handles the build process, signing, and deployment using Fastlane.

## Features

- Automated Firebase App Distribution deployment
- iOS IPA build and signing
- Build artifact archiving
- Gradle and Ruby dependency caching
- Firebase tester group management

## Prerequisites

Before using this action, ensure you have:

1. A Firebase project with App Distribution enabled
2. Firebase service account credentials
3. An iOS app set up in Firebase
4. Xcode project configured with proper signing

## Setup

### Fastlane Setup

Create a `Gemfile` in your project root:

```ruby
source "https://rubygems.org"

gem "fastlane"
gem "firebase_app_distribution"
```

Create a `fastlane/Fastfile` with the following content:

```ruby
default_platform(:ios)

platform :ios do
  desc "Deploy to Firebase App Distribution"
  lane :deploy_on_firebase do |options|
    build_ios_app(
      scheme: ENV["IOS_PACKAGE_NAME"],
      export_method: "ad-hoc"
    )
    
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID"],
      groups: options[:groups],
      service_credentials_file: options[:serviceCredsFile],
      release_notes: "New build from GitHub Actions"
    )
  end
end
```

## Usage

Add the following workflow to your GitHub Actions:

```yaml
name: Deploy to Firebase

on:
  push:
    branches: [ main ]
  # Or trigger on release
  release:
    types: [created]

jobs:
  deploy:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Firebase
        uses: openMF/kmp-publish-ios-on-firebase-action@v1.0.0
        with:
          ios_package_name: 'YourAppName'
          firebase_creds: ${{ secrets.FIREBASE_CREDS }}
          tester_groups: 'qa-team,beta-testers'
```

## Inputs

| Input              | Description                                         | Required |
|--------------------|-----------------------------------------------------|----------|
| `ios_package_name` | Name of your iOS app/scheme                         | Yes      |
| `firebase_creds`   | Base64 encoded Firebase service account credentials | Yes      |
| `tester_groups`    | Comma-separated list of Firebase tester groups      | Yes      |

## Setting up Secrets

1. Encode your Firebase credentials file to base64:
```bash
base64 -i path/to/firebase-credentials.json -o firebase-creds.txt
```

2. Add the following secret to your GitHub repository:
- `FIREBASE_CREDS`: Content of firebase-creds.txt

## Build Artifacts

The action uploads the built IPA file as an artifact with:
- Name: 'ios-app'
- Retention period: 1 day
- Maximum compression (level 9)

You can find the IPA file in your GitHub Actions run artifacts.

This helps reduce build times in subsequent runs.

## Firebase App Setup

1. Go to the Firebase Console
2. Navigate to App Distribution
3. Create a new iOS app or select an existing one
4. Set up tester groups under App Distribution
5. Create a service account with appropriate permissions
6. Download the service account JSON key file

## Troubleshooting

Common issues and solutions:

1. Build fails due to signing
   - Ensure your Xcode project has proper signing configuration
   - Verify the export method matches your provisioning profile

2. Firebase deployment fails
   - Check if the service account has sufficient permissions
   - Verify the Firebase app ID is correct
   - Ensure tester groups exist in Firebase console
