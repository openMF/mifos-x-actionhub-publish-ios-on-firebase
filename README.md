# KMP Build & Publish iOS App on Firebase

This GitHub Action automates the process of building an iOS application, generating release notes, and distributing the build to Firebase App Distribution. It includes version management and comprehensive caching for optimized build times.

## Features

- Automated iOS app building
- Version number generation
- Automated release notes generation
- Firebase App Distribution integration
- Gradle and Konan caching
- Java development environment setup
- Fastlane integration

## Prerequisites

- iOS project with Xcode configuration
- Firebase project setup
- GitHub repository with release history
- Fastlane setup in the repository
- Valid iOS certificates and provisioning profiles

## Inputs

| Input              | Description                               | Required |
|--------------------|-------------------------------------------|----------|
| `ios_package_name` | Name of the iOS project module            | Yes      |
| `firebase_creds`   | Firebase service account credentials JSON | Yes      |
| `github_token`     | GitHub token for API access               | Yes      |
| `target_branch`    | Target branch for deployment              | Yes      |

## Usage

```yaml
name: KMP iOS deploy to Firebase

on:
  workflow_dispatch:
    inputs:
      ios_package_name:
        description: 'Name of the iOS project module'
        required: true
        default: 'mifospay-ios'


permissions:
  contents: write

jobs:
  deploy_ios_app:
    name: Deploy iOS App
    runs-on: macos-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Deploy iOS App to Firebase
        uses: openMF/kmp-publish-ios-on-firebase-action@v1.0.0
        with:
          ios_package_name: ${{ inputs.ios_package_name }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          firebase_creds: ${{ secrets.FIREBASECREDS }}
          target_branch: 'dev'

```

## Workflow Details

1. **Environment Setup**
    - Configures Java 17 (Zulu distribution)
    - Sets up Gradle with caching
    - Configures Ruby and Fastlane
    - Sets up dependency caching for Gradle, Konan, and build outputs

2. **Version Management**
    - Generates version codes based on commits and tags
    - Reads version name from version.txt
    - Formula: `version-code = (commits + tags) << 1`

3. **Release Notes Generation**
    - Fetches latest release tag
    - Generates release notes using GitHub API
    - Creates two changelog files:
        - `changelogGithub`: Comprehensive release notes
        - `changelogBeta`: Changes since last commit

4. **Build Process**
    - Configures Firebase credentials
    - Builds iOS application using Fastlane
    - Generates IPA file

5. **Distribution**
    - Uploads build artifacts with high compression
    - Distributes to Firebase App Distribution

## Dependencies

- Java 17 (Zulu distribution)
- Gradle
- Ruby
- Bundler 2.2.27
- Fastlane plugins:
    - firebase_app_distribution
    - increment_build_number