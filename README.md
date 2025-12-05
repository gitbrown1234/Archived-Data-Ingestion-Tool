# Archived-Data-Ingestion-Tool
Splunk Archived Data Ingestion Tool

This Splunk app provides a powerful and flexible way to selectively ingest archived data from an NFS share. It combines a custom Splunk search command and a user-friendly dashboard to allow you to "bring back" specific files on demand.


Table of Contents

Overview
Core Components
How It Works (Workflow)
Setup and Installation
Prerequisites
App Installation
Universal Forwarder (UF) Setup
NFS Share Configuration
Permissions
Custom Command:
bringbackdata
Dashboard: Archived Data Ingestion Tool
Usage Guide
Troubleshooting
Future Enhancements


1. Overview

The "Splunk Archived Data Ingestion Tool" streamlines the process of re-ingesting specific archived files from an NFS share. Instead of manually configuring inputs.conf for each file, this app provides an automated, on-demand solution.


The process is simple:


Archive Data: Your data is written as raw files to an NFS share.
Discover Files: A pre-existing Splunk search helps discover the paths of these archived files.
Select & Ingest: A dashboard allows you to browse and select specific archived files.
Dynamic Configuration: A custom command within this app dynamically generates an inputs.conf stanza for each selected file. This inputs.conf is saved within the app itself.
Deployment (Optional via DS): If you use a Deployment Server, it will push this updated app (containing the new inputs.conf) to your Universal Forwarder.
Ingestion: The Universal Forwarder, upon reloading its configuration, begins monitoring and ingesting only the selected files.

2. Core Components

This Splunk App: Contains the bringbackdata.py custom command, its configuration, and the dashboard. It also serves as the container for the dynamically generated inputs.conf.
bringbackdata.py Custom Command: A Python script that takes a list of file paths, desired sourcetype, and index, then writes corresponding monitor stanzas to an inputs.conf file within this app's local directory.
Archived Data Ingestion Dashboard: A user interface to select files and trigger the custom command.
Splunk Universal Forwarder (UF): Installed on a server that has read access to the NFS share. It will monitor the selected files for ingestion.
NFS Share: The network file system location where your raw archived data is stored.

3. How It Works (Workflow)

Archiving: Your existing data archiving process writes raw data files to a designated NFS share (e.g., /opt/splunk/archived-data/year=YYYY/month=MM/day=DD/sourcetype=X/).
Initial Discovery: A separate Splunk input (e.g., a monitor input on the NFS share itself, or a script that lists files) populates an index (e.g., index=main) with records containing the full paths of available archived files. This is what your sourcetype=deepest_dirs search is likely doing.
Dashboard Interaction:
A user accesses the "Archived Data Ingestion Tool" dashboard.
They specify criteria (Year, Day, Sourcetype) to filter the list of available archived files.
The Select Files to Bring Back checkbox populates with the full paths of matching files.
The user selects the specific files they wish to ingest.
They also specify the desired Sourcetype for Ingested Data and Index for Ingested Data.
Crucially, they click the Submit button.
Custom Command Execution:
Clicking "Submit" triggers the | bringbackdata custom command.
The command receives the selected file paths, target sourcetype, and target index.
The Python script (bringbackdata.py) generates monitor stanzas for each selected file, specifying the chosen sourcetype and index.
It writes these stanzas to etc/apps/YOUR_APP_NAME/local/inputs.conf (where YOUR_APP_NAME is the name of this app). The directory structure for this inputs.conf is automatically created if it doesn't exist.
Configuration Reload:
For the Universal Forwarder to start monitoring the newly specified files, the app's configuration on the UF must be reloaded (or the UF restarted).
If a Deployment Server is used, it will detect the change in the app's inputs.conf and push the updated app to the UF. The UF will then need its configuration reloaded.
Universal Forwarder Ingestion:
Once reloaded, the UF starts monitoring the newly specified files on the NFS share.
Data from these files is then ingested into the specified Splunk index with the chosen sourcetype. The source field for each event will automatically be the full path of the ingested file.

4. Setup and Installation

Prerequisites

A running Splunk Enterprise environment (Search Head, Indexer).
A Universal Forwarder (UF) installed on a machine with read access to the NFS share.
An NFS share configured and mounted on the UF machine.
Python 3.x environment on your Splunk instance (typically /opt/splunk/bin/python3.9).
splunklib installed for the custom command's Python environment.

App Installation

Download the App: Obtain the Splunk app package (e.g., .tar.gz or .tgz) from the GitHub repository.
Install on Search Head:
Log into your Splunk Search Head.
Go to Apps > Manage Apps.
Click Install app from file.
Choose the downloaded app package and click Upload.
Restart Splunk if prompted.
Install on Deployment Server (if applicable): If you use a Deployment Server to manage your Universal Forwarders, install this app on your Deployment Server in the $SPLUNK_HOME/etc/deployment-apps/ directory.
bash
Copy Code
# Example for installing a .tgz file
sudo tar -xzf /path/to/your_app.tgz -C $SPLUNK_HOME/etc/deployment-apps/
sudo chown -R splunk:splunk $SPLUNK_HOME/etc/deployment-apps/YOUR_APP_NAME
sudo $SPLUNK_HOME/bin/splunk reload deploy-server
Then, ensure your UF is a deployment client of this DS and is assigned this app.
Install on Universal Forwarder (if no DS): If you are not using a Deployment Server, you will need to manually install this app on your Universal Forwarder(s) by placing it in $SPLUNK_HOME/etc/apps/.
bash
Copy Code
sudo tar -xzf /path/to/your_app.tgz -C $SPLUNK_HOME/etc/apps/
sudo chown -R splunk:splunk $SPLUNK_HOME/etc/apps/YOUR_APP_NAME
sudo $SPLUNK_HOME/bin/splunk restart # Restart UF after manual install

Universal Forwarder (UF) Setup

NFS Access: Ensure the UF has read access to the NFS share where your archived data is located.
App Deployment: Verify that this Splunk app is successfully deployed to your UF (either via DS or manual installation).

NFS Share Configuration

Mount: Ensure the NFS share is mounted on the Universal Forwarder machine.
Accessibility: Verify that the Splunk user on the UF has read access to the archived files on the NFS share.

Permissions

This is critical for the solution to work. The Splunk user (the user account under which Splunk processes run) needs specific permissions:


On the machine running the custom command (typically Search Head):
The splunk user must have write permissions to $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/local/. This is where the bringbackdata.py script will write the inputs.conf file.
bash
Copy Code
sudo chown -R splunk:splunk $SPLUNK_HOME/etc/apps/YOUR_APP_NAME
On the Universal Forwarder:
The splunk user must have read permissions to the archived files on the NFS share.

5. Custom Command: bringbackdata

Location within the app: bin/bringbackdata.py


This Python-based custom command is the core logic for generating the inputs.conf.


Syntax:


splunk
Copy Code
| bringbackdata field1="<space_separated_file_paths>" ingest_sourcetype="<sourcetype>" ingest_index="<index>" inputs_conf_path="<full_path_to_inputs.conf>"

Options:


field1 (required): A space-separated string of the full paths to the archived files you wish to ingest.
ingest_sourcetype (optional, default: archived_data_ingest): The sourcetype to assign to the ingested events.
ingest_index (optional, default: archive): The Splunk index where the events will be stored.
inputs_conf_path (required): The full path to the inputs.conf file to be created/updated. This will typically be $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/local/inputs.conf. The dashboard automatically constructs this path based on your input.

Functionality:


Parses the field1 value into individual file paths.
Generates a [monitor://<file_path>] stanza for each file.
Sets sourcetype, index, disabled=false, crcSalt=<SOURCE>, and followTail=0 for each stanza.
Writes (overwrites) these stanzas to the specified inputs_conf_path.
Creates the necessary directory structure for inputs_conf_path if it doesn't exist.
Yields Splunk events indicating the files being processed and the success/failure of inputs.conf generation.
Important: The source field for ingested events will automatically be the full path of the monitored file.

6. Dashboard: Archived Data Ingestion Tool

Location within the app: default/data/ui/views/archived_data_ingestion_tool.xml (or similar)


This dashboard provides the user interface for interacting with the bringbackdata command.


Key Features:


Input Fields:
Year, Day, Sourcetype (for finding files): Used to filter a list of available archived files.
Target App Name for inputs.conf: This should be the name of this app (e.g., your_app_name). This input is used to construct the correct path to inputs.conf within the app.
Sourcetype for Ingested Data: Sets the sourcetype for the files once ingested.
Index for Ingested Data: Sets the Splunk index for the ingested files.
Select Files to Bring Back Checkbox: Populated by a search (index=main sourcetype=deepest_dirs ...) that should return full file paths in its _raw field.
Submit Button: All inputs are wrapped in a fieldset submitButton="true", meaning the bringbackdata command only runs when this button is clicked.
Conditional Panel (depends): The panel containing the bringbackdata search uses depends="$field1$ != &quot;Nothing&quot;". This prevents the command from running on initial dashboard load and only executes it when a file (or files) other than the default "Nothing" has been selected and submitted.
Command Output: Displays the events generated by the bringbackdata command, confirming the inputs.conf generation and providing a critical reminder.

7. Usage Guide

Access the Dashboard: Navigate to the "Archived Data Ingestion Tool" dashboard in Splunk.
Filter Archived Files: Use the Year, Day, and Sourcetype (for finding files) inputs to narrow down the list of archived files.
Select Files: In the Select Files to Bring Back checkbox, choose the specific archived files you want to ingest. Ensure these are full paths.
Specify Ingestion Details:
Enter the Target App Name for inputs.conf. This should be the exact name of the app you installed (e.g., my_archive_ingestor_app).
Enter the desired Sourcetype for Ingested Data and Index for Ingested Data.
Trigger Ingestion Configuration: Click the Submit button.
Review Output: The table panel will display the output from the bringbackdata command, confirming the inputs.conf generation and providing a critical reminder.
Reload Configuration (CRITICAL STEP):
For the Universal Forwarder to start monitoring the newly specified files, the app's configuration on the UF must be reloaded (or the UF restarted).
Method 1 (Recommended): On your Splunk Search Head, go to Apps > Manage Apps. Find this app (e.g., your_app_name) and click Reload under the "Actions" column. If you are using a Deployment Server, this will push the reload command to the UF.
Method 2 (Alternative): Restart the Universal Forwarder directly.
bash
Copy Code
sudo $SPLUNK_HOME/bin/splunk restart
Verify Ingestion: After reloading, the UF will start monitoring the selected files. You can then search Splunk for the ingested data using the specified index and sourcetype.

8. Troubleshooting

ModuleNotFoundError: No module named 'splunklib':
Ensure splunklib is installed in the correct location within the app: $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/lib/.
Verify installation using the exact Splunk Python interpreter:
bash
Copy Code
/opt/splunk/bin/python3.9 -m pip install splunklib -t $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/lib
AttributeError: Inapplicable configuration settings: retaincontext=True:
This indicates an issue with the custom command's Python code. Ensure retaincontext=True is removed from the @Configuration decorator in bringbackdata.py.
External search command 'bringbackdata' returned error code 1 (due to commands.conf):
Verify that commands.conf (located in YOUR_APP_NAME/default/commands.conf) has the correct stanza for bringbackdata as defined in the app's package.
Failed to write inputs.conf... Permission denied:
The Splunk user (on the Search Head) does not have write permissions to $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/local/.
Adjust permissions: sudo chown -R splunk:splunk $SPLUNK_HOME/etc/apps/YOUR_APP_NAME
Files not ingesting after running command:
Did you reload the app on the UF? This is the most common reason. See Usage Guide - Step 7.
Check the UF's splunkd.log for errors related to this app or monitor stanzas.
Verify the inputs.conf content on the UF (e.g., $SPLUNK_HOME/etc/apps/YOUR_APP_NAME/local/inputs.conf).
Ensure the UF has read access to the actual file paths on the NFS share.
Confirm the field1 values from your dashboard are indeed full, correct file paths.

9. Future Enhancements

Append vs. Overwrite: Implement logic in bringbackdata.py to append new stanzas to inputs.conf instead of always overwriting, while also handling potential duplicates.
Deletion/Disabling: Add functionality to the dashboard to disable or remove existing monitor stanzas from inputs.conf.
More Robust File Discovery: Enhance the initial file discovery search to handle more complex directory structures or metadata.
Error Handling: Improve error reporting in the custom command and dashboard for better user feedback.
Confirmation Dialog: Add a confirmation dialog to the dashboard before generating inputs.conf.
