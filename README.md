Splunk Archived Data Ingestion Tool

This Splunk app provides a streamlined way to selectively ingest archived data from an NFS share into Splunk. It combines a custom search command and a dashboard to allow on-demand "bring back" of specific files.


Overview

Your data is archived as raw files on an NFS share. This app enables you to:


Discover archived file paths using a Splunk search.
Select specific files from a dashboard.
Generate monitor stanzas in inputs.conf for the selected files within this app.
Ingest those files via a Universal Forwarder (UF) monitoring the NFS share.

How It Works

File Discovery: Your existing Splunk data (e.g., from sourcetype=deepest_dirs) provides a list of archived file paths.
Dashboard Selection: You use the app's dashboard to filter these paths and select the specific files you want to ingest. You also define the target sourcetype and index.
inputs.conf Generation: Clicking "Submit" on the dashboard triggers the | bringbackdata custom command. This command writes [monitor://<file_path>] stanzas for your selected files directly into YOUR_APP_NAME/local/inputs.conf.
Deployment & Ingestion: If using a Deployment Server, it pushes the updated app (with the new inputs.conf) to your Universal Forwarder. The UF then needs its configuration reloaded (or restarted) to begin monitoring and ingesting the selected files from the NFS share. Events will have the full file path as their source.

Setup and Installation

Prerequisites

Splunk Enterprise (Search Head) and Universal Forwarder (UF).
NFS share mounted on the UF, with read access for the Splunk user.
Python 3.x environment for Splunk (e.g., /opt/splunk/bin/python3.9).
splunklib installed within this app's lib directory.

Installation Steps

Install App: Download the app package (e.g., .tar.gz) from GitHub.

On Search Head: Install via Splunk Web (Apps > Manage Apps > Install app from file).
On Deployment Server (if used): Extract into $SPLUNK_HOME/etc/deployment-apps/ and assign to your UF clients.
On Universal Forwarder (if no DS): Extract into $SPLUNK_HOME/etc/apps/.
Restart Splunk if prompted or after manual installation.
Install splunklib (if not already packaged): Ensure splunklib is installed within this app's Python environment.

bash
Copy Code
/opt/splunk/bin/python3.9 -m pip install splunklib -t $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/lib
(Replace YOUR_APP_NAME with the actual name of this app.)

Permissions: The Splunk user must have:

Write permissions to $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/local/ (on the machine running the dashboard/custom command).
Read permissions to the archived files on the NFS share (on the UF).

Usage Guide

Access Dashboard: Open the "Archived Data Ingestion Tool" dashboard.
Filter & Select: Use the inputs to find and select the full paths of the files you want to ingest.
Set Ingestion Details: Specify the Target App Name (this app's name), Sourcetype, and Index for the ingested data.
Submit: Click the Submit button to run the bringbackdata command.
Reload Configuration (CRITICAL): After the command runs successfully, you MUST reload the app's configuration on the Universal Forwarder for the new monitor stanzas to take effect.
Recommended: On your Splunk Search Head, go to Apps > Manage Apps, find this app, and click Reload.
Alternative: Restart the Universal Forwarder.
Verify: Search Splunk for your ingested data using the specified index and sourcetype.

Troubleshooting

ModuleNotFoundError: No module named 'splunklib': Re-install splunklib into the app's lib directory using the Splunk Python interpreter.
Failed to write inputs.conf... Permission denied: Adjust file permissions for the Splunk user on the app's local directory.
Files not ingesting: Did you reload the app's configuration on the UF after running the command? This is the most common cause. Also, verify UF has read access to NFS files.
