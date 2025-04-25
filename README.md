# KMP Publish iOS App on Firebase Action

This GitHub Action automates the process of building and publishing iOS applications to Firebase App Distribution. It handles the build process, signing, and deployment using Fastlane.

## Features

- Automated Firebase App Distribution deployment
- iOS IPA build and signing
- Build artifact archiving
- Gradle and Ruby dependency caching
- Firebase tester group management
- SSH key authentication via base64 secrets

## Prerequisites

Before using this action, ensure you have:

1. A Firebase project with App Distribution enabled
2. Firebase service account credentials
3. An iOS app set up in Firebase
4. Xcode project configured with proper signing
5. A private GitHub repo with certificates and provisioning profiles for Fastlane Match
6. SSH key added as a deploy key to the certificates repo

## Setup

### SSH Key Setup
1. Generate SSH Key Locally<br>
   In your terminal, run:
```yaml
ssh-keygen -t ed25519 -C "your_email@example.com"
```
It will ask for a file path. Press enter a custom path like:
```yaml
~/.ssh/match_ci_key
```
You can skip setting a passphrase when prompted (just hit enter twice).

This generates two files:

- `~/.ssh/match_ci_key` → Private key

- `~/.ssh/match_ci_key.pub` → Public key

2. Add the Private Key to the SSH Agent (optional but helpful)
   This step ensures the key is used during local development.
```yaml
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/match_ci_key
```
3. Add the Public Key to Your Certificates Repo (GitHub)
- Go to your certificates repo on GitHub (e.g., openMF/ios-provisioning-profile).
- Go to Settings → Deploy Keys.
- Click “Add deploy key”.
- Set title as `CI Match SSH Key` and paste the content of:

```yaml
cat ~/.ssh/match_ci_key.pub
```
- Check **Allow write access**.
- Click Add key.

4. Convert the Private Key to Base64
   This is how we pass it to GitHub Actions securely.
```yaml
base64 -i ~/.ssh/match_ci_key | pbcopy
```
This command copies the base64-encoded private key to your clipboard (macOS).

5. Save the Private Key as a GitHub Secret
- Go to the repo with your Fastfile (the project repo, not the certs repo).
- Navigate to Settings → Secrets and variables → Actions → New repository secret.
- Add the following:
   - Name: MATCH_SSH_PRIVATE_KEY
   - Value: Paste the base64-encoded private key

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
  
    firebase_app_id = options[:firebase_app_id] || ENV["FIREBASE_APP_ID"]
    match_type = options[:match_type] || ENV["MATCH_TYPE"]
    app_identifier = options[:app_identifier] || ENV["APP_IDENTIFIER"]
    git_url = options[:git_url] || ENV["GIT_URL"]
    git_branch = options[:git_branch] || ENV["GIT_BRANCH"]
    match_password = options[:match_password] || ENV["MATCH_PASSWORD"]
    git_private_key = options[:git_private_key] || ENV["GIT_PRIVATE_KEY"] || "./secrets/match_ci_key"
    groups = options[:groups] || ENV["GROUPS"]
    serviceCredsFile = options[:serviceCredsFile] || ENV["SERVICE_CREDS_FILE"]
    
    setup_ci(
      provider: "circleci"
    )
    
    latest_release = firebase_app_distribution_get_latest_release(
      app: firebase_app_id,
      service_credentials_file: options[:serviceCredsFile] || firebase_config[:serviceCredsFile]
    )

    if latest_release
      increment_build_number(
        xcodeproj: iosApp/iosApp.xcodeproj,
        build_number: latest_release[:buildVersion].to_i + 1
      )
    else
      UI.important("⚠️ No existing Firebase release found. Skipping build number increment.")
    end

    match(
      type: match_type,
      app_identifier: app_identifier,,
      readonly: true,
      git_url: git_url,
      git_branch: git_branch,
      git_private_key: git_private_key
    )
    
    build_ios_app(
        scheme: "iosApp",
        project: "iosApp/iosApp.xcodeproj",
        output_name: "DeployIosApp.ipa",
        output_directory: "iosApp/build",
    )
    
    firebase_app_distribution(
      app: firebase_app_id,
      service_credentials_file: serviceCredsFile,
      release_notes: r"New build from GitHub Actions",
      groups: groups
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
           app_identifier: 'org.mifos.kmp.template'
           git_url: 'git@github.com:openMF/ios-provisioning-profile.git'
           git_branch: 'master'
           match_type: 'adhoc'
           provisioning_profile_name: 'match AdHoc org.mifos.kmp.template'
           match_password: ${{ secrets.MATCH_PASSWORD }}
           match_ssh_private_key: ${{ secrets.MATCH_SSH_PRIVATE_KEY }}
           ios_package_name: 'YourAppName'
           firebase_creds: ${{ secrets.FIREBASE_CREDS }}
           firebase_app_id: '1:xxxx:ios:xxxxxxxx'
           tester_groups: 'qa-team,beta-testers'
```

## Inputs and Secrets

| Input              | Description                                         | Required |
|--------------------|-----------------------------------------------------|----------|
| `ios_package_name` | Name of your iOS app/scheme                         | Yes      |
| `firebase_creds`   | Base64 encoded Firebase service account credentials | Yes      |
| `tester_groups`    | Comma-separated list of Firebase tester groups      | Yes      |
| `MATCH_PASSWORD`    | Used to encrypt/decrypt match certs      | Yes      |
| `MATCH_SSH_PRIVATE_KEY`    | base64 encoded private SSH key used by Fastlane Match      | Yes      |

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