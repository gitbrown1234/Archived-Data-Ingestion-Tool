# Splunk Archived Data Ingestion Solution

This solution provides a streamlined, dashboard-driven approach to selectively ingest archived data from an NFS share into Splunk. It leverages two distinct Splunk apps:

1.  **`TA-archived-data-ingestor`**: (Installed on Deployment Server) Contains the user-facing dashboard and the custom command that generates `inputs.conf`.
2.  **`scrape_files`**: (Installed on Deployment Server, pushed to Universal Forwarder) Contains the `find_deepest_dirs.sh` script for file discovery and is the target for the dynamically generated `inputs.conf`.

## Overview

This solution is designed to manage and selectively re-ingest data that has been archived to an NFS share, particularly useful for **low-value logs that are typically not ingested into Splunk but should not be deleted**. Often, such data is written to NFS via "ingest actions" or other archiving mechanisms. When a specific need arises to examine a subset of this archived data (e.g., for a compliance audit, incident investigation, or rare troubleshooting), manually re-ingesting it can be cumbersome.

This tool simplifies that process by allowing you to:

*   **Discover Files:** The `scrape_files` app on a Universal Forwarder (UF) uses `find_deepest_dirs.sh` to regularly scan the NFS share and index available file paths.
*   **Select & Configure:** From the `TA-archived-data-ingestor` dashboard (accessed on the Deployment Server), you select specific archived files and define their target `sourcetype` and `index`.
*   **Dynamic `inputs.conf` Generation:** A custom command, running on the Deployment Server, writes `monitor` stanzas for your selections directly into the `scrape_files` app's `local/inputs.conf` *on that same Deployment Server*.
*   **Deployment & Ingestion:** The DS pushes the updated `scrape_files` app to the UF. After a configuration reload, the UF begins monitoring and ingesting only the selected files.

## How It Works (Workflow)

1.  **File Discovery (`scrape_files` app on UF):** The `find_deepest_dirs.sh` script (configured as a scripted input within the `scrape_files` app on the UF) runs periodically, scanning your NFS share for archived files and indexing their full paths (e.g., to `index=main sourcetype=deepest_dirs`).
2.  **Dashboard Interaction (`TA-archived-data-ingestor` app on DS):**
    *   You access the dashboard on the Deployment Server (or a Search Head configured to search the DS).
    *   You use filters (Year, Day, Sourcetype) to populate a checkbox list of discovered file paths.
    *   You select the files, specify the desired `sourcetype` and `index` for ingestion, and click **Submit**.
3.  **`inputs.conf` Generation (`bringbackdata` command on DS):**
    *   The `| bringbackdata` custom command executes on the Deployment Server.
    *   It generates `[monitor://<file_path>]` stanzas for your selections.
    *   **It writes this `inputs.conf` content to the `scrape_files` app's `local` directory located *on the Deployment Server itself*.**
4.  **Deployment & Ingestion (DS to UF):**
    *   The Deployment Server detects the change in `scrape_files/local/inputs.conf`.
    *   The DS pushes the updated `scrape_files` app to the Universal Forwarder.
    *   The UF's configuration **must be reloaded** (or the UF restarted) for the new `monitor` stanzas to take effect.
    *   The UF then starts monitoring and ingesting the selected files from the NFS share. Events will have the full file path as their `source`.

## Setup and Installation

### Prerequisites

*   Splunk Enterprise Deployment Server (DS) (which may also act as a Search Head).
*   Universal Forwarder (UF) with read access to the NFS share.
*   NFS share mounted on the UF.
*   Python 3.x environment on your Splunk instances.
*   `splunklib` installed within the `TA-archived-data-ingestor` app's `lib` directory on the DS.

### Installation Steps

You have already installed the following:

1.  **`TA-archived-data-ingestor` App:** Installed on your **Deployment Server** (in `$SPLUNK_HOME/etc/apps/` or `$SPLUNK_HOME/etc/deployment-apps/` if it's also pushed to other SHs). This app contains:
    *   The dashboard.
    *   `bin/bringbackdata.py` (the custom command).
    *   `lib/splunklib` (required for the custom command).
    *   Its `commands.conf` entry.

2.  **`scrape_files` App:** Installed on your **Deployment Server** (in `$SPLUNK_HOME/etc/deployment-apps/`) and pushed to your **Universal Forwarder(s)**. This app contains:
    *   `bin/find_deepest_dirs.sh` (the file discovery script).
    *   An `inputs.conf` (e.g., in `default/`) to configure `find_deepest_dirs.sh` as a scripted input.
    *   An empty `local/inputs.conf` (or just the `local` directory) where the `bringbackdata` command will write its output.

### Permissions (CRITICAL)

*   **On the Deployment Server:** The `splunk` user (running the `bringbackdata` command on the DS) **must have write permissions** to `$SPLUNK_HOME/etc/deployment-apps/scrape_files/local/`. This is essential for the custom command to write the `inputs.conf` file.
    ```bash
    sudo chown -R splunk:splunk $SPLUNK_HOME/etc/deployment-apps/scrape_files
    ```
*   **On the Universal Forwarder:** The `splunk` user must have **read permissions** to the archived files on the NFS share.

## Usage Guide

1.  **Ensure File Discovery is Running:** Verify the `find_deepest_dirs.sh` script (on the UF) is populating your Splunk index with archived file paths.
2.  **Access Dashboard:** Open the "Archived Data Ingestion Tool" dashboard on your Deployment Server.
3.  **Filter & Select:** Use the dashboard inputs to find and select the full paths of the files you want to ingest.
4.  **Set Ingestion Details:** Specify the `Target App Name` (which should be `scrape_files`), `Sourcetype`, and `Index` for the ingested data.
5.  **Submit:** Click the **Submit** button to run the `bringbackdata` command.
6.  **Review Output:** The dashboard will show the command's output, confirming `inputs.conf` generation and providing a crucial reminder.
7.  **Reload Configuration (CRITICAL):** After the command runs successfully, you **MUST reload the `scrape_files` app's configuration on the Universal Forwarder** for the new `monitor` stanzas to take effect.
    *   **Recommended:** On your Deployment Server, go to **Apps > Manage Apps**, find the `scrape_files` app, and click **Reload**. This will trigger the DS to push the updated app and reload it on the UF.
    *   **Alternative:** Restart the Universal Forwarder directly.
8.  **Verify:** Search Splunk for your ingested data using the specified `index` and `sourcetype`.

## Troubleshooting

*   **`Failed to write inputs.conf... Permission denied`**: This is a permission issue on the Deployment Server. Ensure the Splunk user has write access to `$SPLUNK_HOME/etc/deployment-apps/scrape_files/local/`.
*   **Files not ingesting**:
    *   Did you reload the `scrape_files` app's configuration on the UF after running the command?
    *   Verify the `inputs.conf` content on the UF (`$SPLUNK_HOME/etc/apps/scrape_files/local/inputs.conf`).
    *   Confirm the UF has read access to the NFS files.
    *   Ensure the file paths selected on the dashboard are correct and absolute.
*   **`ModuleNotFoundError: No module named 'splunklib'`**: Re-install `splunklib` into `TA-archived-data-ingestor/lib` on the Deployment Server.
