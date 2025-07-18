# update\_cert.sh

## Overview

`update_cert.sh` is a Bash script that pulls a SSL/TLS certificat from a remote source and deploys them on a Synology NAS. It automates fetching, archiving, converting, and installing the certificat for both system services and installed packages.

The script is designed to retrieve the certificat using `rsync`, but it can easily be adapted to work with local paths or other retrieval methods. The source server can use any ACME client (e.g. `acme.sh`, `Certbot`), as long as it outputs standard PEM-format files.

## Variable Naming Conventions

* `*_SRC`: Refer to the filenames of the original certificate files as provided by the remote source. These correspond to the filenames after they are copied into the local staging directory using `rsync`.
* `*_TGT`: Refer to the standardized filenames that Synology expects for its internal certificate handling. These include:

  * `cert.pem`: The domain certificate
  * `chain.pem`: The intermediate certificate chain
  * `fullchain.pem`: The full certificate chain including the domain certificate
  * `privkey.pem`: The private key (converted to PKCS#8 format if necessary)
  * `short-chain.pem`: A duplicate of the chain (same content as `chain.pem`)
  * `root.pem`: The root certificate, extracted from the DSM trust store by matching the last certificate in the provided `chain.pem`

## Requirements

This script was tested successfully on a **Synology DS923+ running DSM 7.2**.

The following tools must (and probably will) be available on the system:

* **Bash**
* **rsync** for pulling files from the certificate source
* **jq** for parsing Synology's certificate archive metadata (`INFO` file)
* **openssl** for private key inspection and optional conversion to PKCS#8
* **root access** to modify system certificate locations and restart services
* **SSH access** for optional use of the dry-run feature** 

If the source private key is in a non-PKCS#8 format (e.g. `BEGIN EC PRIVATE KEY`), it will be automatically converted to PKCS#8 (`BEGIN PRIVATE KEY`) as expected by Synology.

## Configuration

Adjust the following variables at the top of the script:

| Variable            | Description                                                                                                                            |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `DOMAIN`            | Your domain name (e.g. `example.com`)                                                                                                  |
| `REMOTE_USER`       | SSH user on the certificate host                                                                                                       |
| `REMOTE_HOST`       | Hostname or IP of the certificate source                                                                                               |
| `REMOTE_BASE_DIR`   | Remote directory containing the cert files                                                                                             |
| `LOCAL_BASE_DIR`    | Local staging directory for pulled files                                                                                               |
| `ARCHIVE_BASE_DIR`  | Synology archive directory (usually `/usr/syno/etc/certificate/_archive`)                                                              |
| `CERT_DESC`         | The certificate description label as assigned during manual certificate installation via the DSM web GUI. **This must match exactly**. |
| Logging (`LOGFILE`) | Path to the log file (default `/var/log/update_cert.log`)                                                                             |

## Installation on Synology

Install your certificat the first time manually. Give it a exact description and use it for `$CERT_DESC` in the config part of the script.

This script must be run with **root privileges**. Due to restrictions in the Synology DSM Task Scheduler (GUI), the script cannot be stored directly there. A **wrapper script** is needed.

To work around this, use a simple wrapper script placed in the Task Scheduler:

Example: If your main script is located at `/volume1/scripts/update_cert.sh`, create the following wrapper script:

```bash
#!/bin/bash
/volume1/scripts/update_cert.sh
```

Then in DSM:

1. Open **Control Panel → Task Scheduler**
2. Create a **Task → User-defined script**
3. Set user to `root`
4. Paste the wrapper script above
5. Set the schedule as needed

## Usage

1. Make the script executable:

   ```bash
   chmod +x /volume1/scripts/update-cert.sh
   ```
   You can set the Priviledge via DSMs File Manager aswell.
2. (Optional) Set `DRY_RUN="true"` at the top to simulate actions without making changes.
3. Run via the Task Scheduler or manually as root:

   ```bash
   sudo /volume1/scripts/update-cert.sh
   ```

## How It Works

1. **Sync**: Uses `rsync` to pull the certificate (`.cer`, `ca.cer`, `fullchain.cer`, `.key`) files from the remote source into `LOCAL_BASE_DIR`.
2. **Change Detection**: Checks `rsync` output for modifications.
3. **Archive Update**: If changes are detected:

   * Copies the files from `*_src` names (e.g. `example.com.cer`, `ca.cer`) to the Synology archive using the `*_tgt` names (e.g. `cert.pem`, `chain.pem`).
   * Converts the private key to PKCS#8 format if needed (`privkey.pem`).
4. **Deploy to Services**:

   * Reads Synology's JSON metadata to find which system services and packages use the given certificate.
   * Copies all required certificate files into each target service’s directory.
   * Adds `short-chain.pem` (same as `chain.pem`)
   * Extracts the root certificate (`root.pem`) by reading the last certificate in `chain.pem`, then finding a matching certificate in Synology's internal trust store (`/etc/ssl/certs`).
5. **Permissions**: Sets owner to `root:root` and file mode to `600` on all deployed `.pem` files.
6. **Restart**: Triggers Synology service reloads (`synow3tool`, Nginx, DSM services, FTP if present).
7. **Logging**: All steps and any errors are logged to `LOGFILE` with timestamps.

## Logging

* **Log file**: `/var/log/update_cert.log`
* **Error entries** are prefixed with `ERROR:` and also printed to STDERR.
