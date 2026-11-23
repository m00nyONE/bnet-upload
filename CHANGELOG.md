# 2026.11.23
- curl now retries 5 times before failing - this should fix the flaky behavior of the action failing to download the CLI from GitHub

## 2025.10.27
- now uses https://github.com/sirinsidiator/ESOAddOnUploaderCLI as a repository to pull the CLI from
- switch default tag to "latest"
- the action now "notifies" you when it ran successfully
- add TODO.md

## 2025.10.24
- initial release