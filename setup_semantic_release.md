# Setup Semantic Release & GoReleaser Playbook

This guide outlines the procedure for configuring a Go project with Semantic Release, GoReleaser, and a Homebrew Tap.

**Agent Protocol:**
1.  **Assume Nothing:** verify tools and paths.
2.  **CRUD Safety:** Explicitly ask for user approval before creating files, running git commands, or using `gh` to create repos.
3.  **User Delegation:** Clearly identify steps that the user *must* perform manually (e.g., creating secrets).

---

## 1. Prerequisites (User Actions Required)

**STOP.** Before proceeding with automation, the user must perform the following actions manually on GitHub. The agent cannot do this securely.

1.  **Generate a Personal Access Token (PAT):**
    *   Go to GitHub -> Settings -> Developer settings -> Personal access tokens -> Tokens (classic).
    *   Generate a new token (e.g., named `goreleaser-token`).
    *   **Scopes:** `repo` (full control of private repositories) or `public_repo` (if only using public repos), and `workflow`.
2.  **Add Secret to Repository:**
    *   Go to the *Code Repository* on GitHub -> Settings -> Secrets and variables -> Actions.
    *   Click "New repository secret".
    *   **Name:** `GH_TOKEN`
    *   **Value:** Paste the PAT generated above.

*Agent Instruction: Ask the user to confirm these steps are complete before proceeding.*

---

## 2. Discovery (Read-Only)

Gather context about the environment. Run these commands without asking for permission.

### 2.1. Verify GitHub CLI Auth
Ensure the agent can interact with GitHub.
```bash
gh auth status
```

### 2.2. Identify Repository Details
Find the current owner and repo name.
```bash
git remote -v
```
*Variable Extraction:*
*   `<OWNER>`: The GitHub user/org name.
*   `<REPO>`: The code repository name.

### 2.3. Check Project Status
Ensure this is a valid Go project and check for existing config.
```bash
ls -F go.mod .goreleaser.yaml .releaserc.json .github/workflows/release.yml
```

---

## 3. Definition (Preparation)

Based on Discovery, define the following variables.

*   **Owner:** `<OWNER>`
*   **Code Repo:** `<REPO>`
*   **Tap Repo:** `homebrew-<REPO>` (Standard convention, or user specified)
*   **Secret Name:** `GH_TOKEN` (Must match what the user created in Step 1)

---

## 4. Execution (State-Changing)

**CRITICAL:** You must obtain explicit user approval before running *any* command in this section.

### 4.1. Initialize Homebrew Tap Repository
If the tap repo does not exist, create it.
*Action:* Create a public repository to host formulae.
```bash
gh repo create <OWNER>/homebrew-<REPO> --public --description "Homebrew tap for <REPO>" --add-readme
```

### 4.2. Create GoReleaser Config
Create `.goreleaser.yaml`. Ensure you use `directory: Formula` (v2 standard) and map the token correctly.

*Action:* Write configuration file.
**File Content (`.goreleaser.yaml`):**
```yaml
version: 2

before:
  hooks:
    - go mod tidy

builds:
  - env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w -X main.version={{.Version}} -X main.commit={{.Commit}} -X main.date={{.Date}}

archives:
  - format: tar.gz
    name_template: >-
      {{ .ProjectName }}_ 
      {{- title .Os }}_ 
      {{- if eq .Arch "amd64" }}x86_64
      {{- else if eq .Arch "386" }}i386
      {{- else }}{{ .Arch }}{{ end }}
      {{- if .Arm }}v{{ .Arm }}{{ end }}
    format_overrides:
    - goos: windows
      format: zip

checksum:
  name_template: 'checksums.txt'

snapshot:
  version_template: "{{ .Tag }}-next"

changelog:
  sort: asc
  filters:
    exclude:
      - '^docs:'
      - '^test:'

brews:
  - name: <REPO>
    repository:
      owner: <OWNER>
      name: homebrew-<REPO>
      token: "{{ .Env.GITHUB_TOKEN }}"
    
    commit_author:
      name: goreleaserbot
      email: bot@goreleaser.com

    directory: Formula
    homepage: "https://github.com/<OWNER>/<REPO>"
    description: "Description of <REPO>."
    license: "MIT"
```

### 4.3. Create Semantic Release Config
Create `.releaserc.json` to configure the release plugins.

*Action:* Write configuration file.
**File Content (`.releaserc.json`):**
```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/github",
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ]
  ]
}
```

### 4.4. Create GitHub Action Workflow
Create `.github/workflows/release.yml` to tie it all together.

*Action:* Write workflow file.
**File Content (`.github/workflows/release.yml`):**
```yaml
name: release

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: stable

      - name: Semantic Release
        id: semantic
        uses: cycjimmy/semantic-release-action@v4
        with:
          extra_plugins: |
            @semantic-release/changelog
            @semantic-release/git
            @semantic-release/github
        env:
          GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}

      - name: Run GoReleaser
        if: steps.semantic.outputs.new_release_published == 'true'
        uses: goreleaser/goreleaser-action@v6
        with:
          distribution: goreleaser
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}
```

### 4.5. Commit and Push
Save the changes to the repository.

*Action:* Run git commands.
```bash
git add .goreleaser.yaml .releaserc.json .github/workflows/release.yml
git commit -m "chore: setup semantic-release and goreleaser"
git push origin main
```

---

## 5. Verification (Read-Only)

1.  **Check Workflow Run:**
    *   Instruct the user to check the "Actions" tab in their repository.
    *   Note: The `chore` commit usually won't trigger a release unless it's a breaking change or configured otherwise.
2.  **Trigger Release:**
    *   Instruct the user to push a feature or fix to test the pipeline.
    ```bash
git commit --allow-empty -m "fix: validate release pipeline"
git push origin main
```

---

## 6. Cleanup

*   No temporary files are created in this process, so no cleanup is required.
