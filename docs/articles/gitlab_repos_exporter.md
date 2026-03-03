### How to Export GitLab Projects Using `gitlab-rails console` and a Bash Script

This guide shows how to export GitLab projects using the internal `gitlab-rails console` and how to automate exports for multiple projects with a Bash script. This approach is useful when standard Rake tasks (for example `gitlab:import_export:export`) fail or behave unexpectedly.

The result of each export is a `.tar.gz` archive that typically includes the repository and import/export metadata (issues, merge requests, pipelines, etc.), suitable for importing into another self-managed GitLab instance.

## When this method is useful

Use `gitlab-rails console` if:

* Rake tasks fail for certain projects (for example, complex or nested group structures).
* You want to verify that the user and project can be resolved correctly before exporting.
* You need a repeatable, scriptable workflow for many projects.

Exports run asynchronously in the background (via Sidekiq), so the archive may not appear immediately.

## Prerequisites

* Self-managed GitLab (SSH access to the GitLab host).
* Sufficient permissions (typically admin-level access).
* Ability to run `sudo` commands on the GitLab server.

## Step-by-step export using `gitlab-rails console`

### 1) Start the Rails console

SSH to the GitLab server and run:

```bash
sudo gitlab-rails console
```

You should see a prompt like `irb(main):001:0>`.

### 2) Resolve the user and project

In the console, locate the user and the project by their identifiers:

```ruby
user = User.find_by_username('your_user')
project = Project.find_by_full_path('your_project/infrastructure/terraform')
```

If either of these is `nil`, double-check:

* The GitLab username (as shown on the profile page).
* The exact project full path (as in the project URL).

### 3) Trigger the export

Start the export job:

```ruby
ProjectExportWorker.new.perform(user.id, project.id)
puts "Export started for #{project.full_path}"
```

A message like `Scoped order is ignored, it's forced to be batch order.` can appear and is often normal for batch/Sidekiq processing.

## Where the exported archive is stored

GitLab typically writes the generated archive under:

```
/var/opt/gitlab/gitlab-rails/uploads/-/system/import_export_upload/export_file/
```

Exit the console (`exit`) and search for `.tar.gz` files created in the last 24 hours:

```bash
find /var/opt/gitlab/gitlab-rails/uploads/ -name "*.tar.gz" -mtime -1
```

Example output:

```
/var/opt/gitlab/gitlab-rails/uploads/-/system/import_export_upload/export_file/<number>/<timestamp>_your_project_infrastructure_terraform_export.tar.gz
```

Archives are often placed in numbered subdirectories. If the archive does not appear right away, wait and repeat the search.

## Automating exports with a Bash script

The script below:

* Defines a GitLab username, output directory, and list of project paths.
* Triggers an export through `gitlab-rails console` for each project.
* Polls the export directory for a recently created `.tar.gz` archive.
* Moves the archive into your output directory and writes a per-project log.

### `export.sh`

```bash
#!/bin/bash

# === Configuration ===
GITLAB_USER="your_user"
OUTPUT_DIR="archives"
REPO_LIST=("your_project/infrastructure/terraform" "your_project/infrastructure/tfmodules")

# === Functions ===
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
}

export_project() {
    local repo_path="$1"
    local archive_path="${OUTPUT_DIR}/${repo_path}-export.tar.gz"
    local temp_dir="/var/opt/gitlab/gitlab-rails/uploads/-/system/import_export_upload/export_file"
    local log_file="${OUTPUT_DIR}/${repo_path}-export.log"

    mkdir -p "$(dirname "$archive_path")"
    if [[ -f "$archive_path" ]]; then
        log "INFO: Archive $archive_path already exists, skipping."
        return 0
    fi

    log "Starting export for project $repo_path..."
    echo "user = User.find_by_username('$GITLAB_USER'); project = Project.find_by_full_path('$repo_path'); ProjectExportWorker.new.perform(user.id, project.id)" | sudo gitlab-rails console > "$log_file" 2>&1

    local max_attempts=360
    local attempt=0
    while [[ $attempt -lt $max_attempts ]]; do
        local temp_archive=$(sudo find "$temp_dir" -name "*.tar.gz" -mtime -1)
        if [[ -n "$temp_archive" && -f "$temp_archive" ]]; then
            sudo mv "$temp_archive" "$archive_path"
            log "OK: Export for project $repo_path saved to $archive_path"
            return 0
        fi
        log "Waiting for export completion of $repo_path... (Attempt: $((attempt+1))/$max_attempts)"
        sleep 10
        ((attempt++))
    done

    log "ERROR: Exporting $repo_path timed out after 60 minutes. Details in $log_file"
    cat "$log_file"
    return 1
}

log "Starting project exports..."
mkdir -p "$OUTPUT_DIR"

for repo in "${REPO_LIST[@]}"; do
    log "Processing project: $repo"
    export_project "$repo"
done

log "All exports completed."
```

## Running the script

1. Save the file as `export.sh` on the GitLab server.
2. Make it executable.
3. Run it.

```bash
chmod +x export.sh
./export.sh
```

Update these variables before running:

* `GITLAB_USER` — the GitLab username used for export permissions.
* `OUTPUT_DIR` — where you want the resulting archives.
* `REPO_LIST` — the list of projects in `group/subgroup/project` format.

## Notes

* Because exports run asynchronously, the script polls for the archive file to appear.
* The script searches for any `.tar.gz` created recently under the export directory. If multiple exports run concurrently, consider tightening the `find` pattern to match a specific project’s filename to avoid moving the wrong archive.
