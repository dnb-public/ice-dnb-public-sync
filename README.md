# ice-dnb-public-sync: Secure Public Code Export Pipeline

> `v1.0.0` | `2026-06-12` | Integrated Cloud Engineering (ICE)
>
> **Status:** Production | **Owner:** `group:ghe-6758-dnbcloudengineering`

---

## 1. Project Overview & Trust Boundaries

The `ice-dnb-public-sync` pipeline is a secure replication and publishing mechanism designed to sync a production-ready, open-source-compliant codebase snapshot from our private internal workspace (`dnb-main`) to our public target organization (`dnb-public`).

To protect internal assets and avoid accidental leaks of keys, private history, or proprietary modules, the system establishes a strict unidirectional trust boundary:

```mermaid
graph TB
    subgraph dnb-main [Internal Zone - dnb-main]
        developer[Developer] -- "1. Push Tag" --> src_repo[(Private repo-name)]
        src_repo -- "2. Trigger" --> snyk_gate{3. Snyk Gate}
        snyk_gate -- "Pass" --> pipeline[Actions Runner]
        snyk_gate -- "Fail" --> block[Block & Notify]
        app_secret[App Private Key] -. "Secure Inject" .-> pipeline
    end

    subgraph github_auth [GitHub Auth Server]
        pipeline -- "4. Auth with Key" --> gh_app[GitHub App]
        gh_app -- "5. Issue Token (IAT)" --> pipeline
    end

    subgraph dnb-public [Public Zone - dnb-public]
        pipeline -- "6. Push Snapshot" --> target_repo[(Public repo-name)]
    end

    style dnb-main fill:#f5f5f7,stroke:#333,stroke-width:2px;
    style dnb-public fill:#f5fff5,stroke:#2e7d32,stroke-width:2px;
    style snyk_gate fill:#fff9c4,stroke:#fbc02d,stroke-width:1px;
```

*   **Internal Zone (`dnb-main`):** Private source repository containing full commit history, internal development/engineering documentation, and secrets. All Snyk gating and security checks run here.
*   **Public Zone (`dnb-public`):** Public organization hosting the public repository. It is a passive, clean, flattened storage endpoint with no historical development logs.

---

## 2. Pipeline Data Flow

The codebase undergoes several sanitization stages before reaching the public storage:

```mermaid
%% Data Flow Diagram (DFD)
graph TB
    E1([Developer])
    E2([GitHub Auth])

    P1[[1. Checkout Tag]]
    P2[[2. Snyk Scan Gate]]
    P3[[3. Scrub & Flatten]]
    P4[[4. Verify & Push]]

    D1[("Private History")]
    D2[("Runner Disk")]
    D3[("Public Storage")]

    E1 -- "Push Tag" --> P1
    D1 -- "Repo Tree" --> P1
    P1 -- "Checked Out" --> D2
    D2 -- "Source Code" --> P2
    P2 -- "Approved" --> P3
    E2 -- "Token" --> P4
    D2 -- "Filtered Files" --> P3
    P3 -- "Flat Commit" --> D2
    D2 -- "Git Stream" --> P4
    P4 -- "Commit Storage" --> D3

    style E1 fill:#bbdefb,stroke:#1565c0
    style E2 fill:#bbdefb,stroke:#1565c0
    style D1 fill:#ffe0b2,stroke:#ef6c00
    style D2 fill:#ffe0b2,stroke:#ef6c00
    style D3 fill:#ffe0b2,stroke:#ef6c00
```

1.  **Checkout & Gate:** The tag is checked out at a depth of 1 (no history).
2.  **Sanitization/Scrubbing:**
    *   Wipes all dotfiles and dotfolders in the root (`.*` files except `.` and `..`).
    *   Wipes all documentation directories (`docs/` and `doc/`) to prevent design leakages.
3.  **Flat Orphan Workspace:**
    *   Initializes a clean, unparented repository (`git init -b main`).
    *   Commits all scrubbed files with message: `"Public Release <tag_name> (Flattened Snapshot)"`.
4.  **Auto-Provision & Secure Push:**
    *   Leverages the GitHub App token to check if the remote repository exists, automatically creating it under `dnb-public` if missing.
    *   Force-pushes the flattened main branch and the release tag.
5.  **Post-Sync Canary Verification:**
    *   Clones the public target fresh and asserts compliance:
        *   **Assertion 1:** No forbidden dotfiles/dotfolders in root.
        *   **Assertion 2:** No forbidden documentation directories (`docs` or `doc`).
        *   **Assertion 3:** No gitignored files present (runs `git check-ignore` against the original `.gitignore` temporarily).
        *   **Assertion 4:** No remote branches other than `main` exist.

---

## 3. Developer Playbook & Verifications

To replicate code to the public organization, follow this two-step process:

### Step 1: Push Your Release Tag
1. Create a tag on your release commit:
   ```bash
   git tag v1.0.0
   ```
2. Push the tag to your internal repository:
   ```bash
   git push origin v1.0.0
   ```

### Step 2: Compliance Verification Checklist
Clone the public repository and perform these checks:
*   **Check A: Flattened History**
    ```bash
    git log --oneline
    ```
    *Expected output:* Exactly **one commit** with message `"Public Release v1.0.0 (Flattened Snapshot)"`.
*   **Check B: Scrubbed Files**
    *   Ensure root contains no hidden files (`.github`, `.env`, `.serena`, etc.).
    *   Ensure there are no `docs/` or `doc/` directories.

---

## 4. Local Automation (`justfile` Recipes)

A standardized `justfile` task runner is configured at the root.

To see available commands:
```bash
just --list
```

*   `just lint` - Audits local workflows and configurations.
*   `just check-links` - Validates integrity of relative markdown links.
*   `just stats` - Outputs stats on file configurations.

---

## 5. Technology Stack & Dependencies

*   **Orchestration:** GitHub Actions
*   **Integration Auth:** GitHub App (`DNB Public Release Bot`)
*   **Execution Shell:** Bash/POSIX
*   **Security Scanning:** Snyk Code & SCA
