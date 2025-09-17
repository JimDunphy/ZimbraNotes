`zmproxyconfgen` (which is the Java class `ProxyConfGen`) is the key. It acts as a "gatekeeper" to decide if a configuration rewrite is necessary. The `echo REWRITE proxy | nc localhost 7171` command is precisely how it tells the `zmconfigd` daemon to perform the actual work.

Based on the Java code in `ProxyConfGen.java` and the process described in the [AJCody wiki](https://wiki.zimbra.com/wiki/Ajcody-Proxy-Config-Txt), here is the explanation of **when it generates new files**.

---

### The Core Logic: Change Detection via Checksum

The fundamental job of `ProxyConfGen.java` is **not** to generate the files itself, but to **detect if the configuration has changed**. It does this by calculating a checksum (specifically, an SHA-1 digest) of all the relevant source data.

It follows this logic every time it's run (e.g., during `zmproxyctl start` or `restart`):

1.  **Calculate the "Current" Checksum:** The program gathers all the configuration values that could possibly affect the final Nginx files. This includes:
    * All proxy-related settings from LDAP (`zimbraMailProxyPort`, `zimbraVirtualHostName`, timeouts, enabled services, etc.).
    * The contents of every single `.template` file in `/opt/zimbra/conf/nginx/templates/`.
    * The server's software version (e.g., ZCS 10.0.14).

2.  **Read the "Stored" Checksum:** It then looks for a state file where it stored the checksum from the *last successful run*. This file is located at:
    * `/opt/zimbra/conf/nginx/proxy_digest`

3.  **Compare the Checksums:** It compares the newly calculated checksum with the one stored in `proxy_digest`.

---

###  When Regeneration is Triggered

Based on that comparison, new files are generated under the following conditions:

* **A Configuration Change is Detected:** This is the most common trigger. If the current checksum **does not match** the stored checksum, `ProxyConfGen` concludes that something has changed and a rewrite is needed. It then sends the `REWRITE proxy` command to `zmconfigd` and, upon success, updates the `proxy_digest` file with the new checksum.
    * This is why editing a template file *should* have worked. Modifying the file changes its content, which changes the overall checksum, triggering the rewrite.

* **The `proxy_digest` File is Missing:** If the state file `/opt/zimbra/conf/nginx/proxy_digest` doesn't exist, the program assumes it's the first run (or a corrupted state) and will always trigger a regeneration.

* **Target Files are Missing:** The code also checks if the essential generated files (like `/opt/zimbra/conf/nginx/includes/nginx.conf.web`) are missing. If they are, it forces a regeneration regardless of the checksum.

* **A "Force" Option is Used:** Commands like `zmproxyctl` often have a `-f` or `--force` flag. When used, this flag is passed to `zmproxyconfgen`, telling it to skip the checksum comparison and always trigger the `REWRITE` command.

In summary, `ProxyConfGen` acts as an intelligent change-detection mechanism. It calculates a fingerprint of the Nginx configuration sources and only triggers the expensive file-generation process (handled by `zmconfigd`) when that fingerprint actually changes.

### Example Usage
See which nginx template files would be touched.
<pre>
/opt/zimbra/libexec/zmproxyconfgen -n -v  --- [-n, --dryrun | -v, --verbose]
</pre>
### Notes
zmproxyctl calls libexec/configwrite proxy when there isn't a 2nd argument to zmproxyctl.
* in zm-mailbox.git - ./store/src/java/com/zimbra/cs/util/ProxyConfGen.java

