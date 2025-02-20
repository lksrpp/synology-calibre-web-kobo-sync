# Setting up Kobo Sync with Calibre-Web on Synology NAS

This guide explains how to enable Kobo e-reader synchronization with a self-hosted Calibre-Web instance running in a Docker container on a Synology NAS by manually modifying the nginx reverse proxy settings to increase the buffer size.

## References

- [Calibre-Web Issue #1891](https://github.com/janeczku/calibre-web/issues/1891#issuecomment-801886803)
- [Synoforum Post](https://www.synoforum.com/threads/synology-reverse-proxy-under-the-hood.5271/page-5#post-71440)


## Problem

- I couldn't get my calibre web instance, accessible via Synology's built-in reverse proxy and a Let's Encrypt SSL certificate, to synchronize reliably with my Kobo e-reader
- Several sources promoted a [GitHub issue](https://github.com/janeczku/calibre-web/issues/1891#issuecomment-801886803), indicating an increased buffer size would fix the problem
- Synology's built-in reverse proxy has limited configuration options and doesn't support this change natively
- I didn't want to switch to a new reverse proxy setup
- A [forum post](https://www.synoforum.com/threads/synology-reverse-proxy-under-the-hood.5271/page-5#post-71440) helped me understand the necessary steps to modify the existing setup in a way that would allow proper syncing


## Prerequisites

1. Synology NAS with DSM 7.x installed

2. Calibre-Web running in a Docker container, with Kobo sync enabled
	1. Go to admin setting and enable `Kobo sync`.
	2. Set `Server External Port` to 80.
	3. Go to your profile page, enable `Kobo sync` and copy the API endpoint.
	4. Update the API endpoint so that the path includes `https://` and not `http://`

3. A Kobo e-reader with an updated config file
	1. Include the updated API endpoint URL from above in the  Kobo eReader.conf file 

4. Built-in Synology Reverse Proxy configured to access your Calibre Web Docker container (e.g., https://calibre.domain.td), incl. certificates and so on.

5. Administrative access to DSM

6. SSH access enabled on your Synology NAS


## Overview

We'll modify the reverse proxy configuration to include the necessary buffer settings for Kobo sync. The process involves creating a new configuration file via SSH on your synology while keeping everything else working as expected.


## Step 1: Prepare DSM Environment

1. Open DSM in your browser using your local IP address: `https://[local-IP]:5001`

2. Keep this browser window open throughout the process

3. Create a text file on your computer for temporary storage of configurations


## Step 2: Create Temporary Reverse Proxy

1. In DSM Control Panel, navigate to: Login Portal > Advanced > Reverse Proxy

2. Document your current Calibre-Web reverse proxy settings

3. Create a new temporary reverse proxy entry:
	- Name it for example "calibre_temp"
	- Use identical settings as your current configuration
	- Change the source port to a different number (e.g., if using 443, use 8443) to avoid conflicts when saving

4. Apply the settings


## Step 3: Configure Custom Nginx Settings

1. SSH into your Synology NAS
    
2. View the current reverse proxy configuration:
   ```bash
   cat /etc/nginx/sites-enabled/server.ReverseProxy.conf
   ```

3. Find and copy the server code block from your calibre_temp reverse proxy settings
	- You can find the correct block based on the source port you defined earlier (e.g., 8443). The beginning will look something like `server { listen 8443 ssl;`
	- Copy the complete block `server {` ... `}` 

4. Modify the server code block and include the buffer settings for Kobo sync
	- Change the ports in the file back to the correct calibre-web port 443
	- Add the three lines that will update the proxy_buffer settings as shown below
	- Keep everything else as is (e.g., certificate details, timeouts)
	- Copy the content to the clipboard
```nginx
   server {
       listen 443 ssl;
       listen [::]:443 ssl;

       # Your existing reverse proxy settings here
       # [...]

       location / {
           # Added buffer settings for Kobo sync
           proxy_buffer_size 32k;
           proxy_buffers 4 32k;
           proxy_busy_buffers_size 64k;

           # Your existing proxy settings here
           # [...]
       }
   }
```

5. Create a new configuration file:
   ```bash
   sudo touch /etc/nginx/conf.d/http.calibre_web.conf
   ```

6. Start editing the new configuration file:
   ```bash
   sudo vi /etc/nginx/conf.d/http.calibre_web.conf
   ```

7. Paste the modified sever code from above (press `i` to enter insert mode, than paste)

8. Save and exit:
	- Press `ESC`
	- Type `:wq`
	- Press `Enter`


## Step 4: Test and Apply Configuration

1. Test the Nginx configuration:
   ```bash
   sudo nginx -t
   ```

2. If you see conflicts, modify your existing reverse proxy rule via GUI:
	- You will probably see a conflict here, because you now have two configurations that listen on Port 443 with the same domain
	- To solve this, go to the reverse proxy settings on your Synology web interface, and change the port of your original Calibre Web rule (e.g., 443 --> 444)
	- Rename the entry to "Calibre Web Backup" or similar. This entry is not really necessary, you could delete it. The idea here is to have a backup rule handy in case you need to reset anything.

3. Test your configuration again. It's important to only proceed if there aren't any conflicts left.

4. Reload Nginx to apply the new configuration:
   ```bash
   sudo nginx -s reload
   ```


## Step 5: Finalize Setup

1. Test accessing Calibre-Web through your normal domain

2. Verify that Kobo sync is working properly

3. If everything works:
	1. Delete the temporary reverse proxy (port 8443)
	2. Keep the backup configuration (port 444) if you want


## Troubleshooting

1. **Nginx Test Fails**
   - Check for syntax errors in the configuration
   - Verify no duplicate server blocks are listening on the same port
   - Ensure all referenced SSL certificates exist

2. **Kobo Sync Not Working**
   - Verify the API endpoint is correctly formatted in the Kobo eReader.conf
   - Check Calibre-Web logs for connection attempts
   - Ensure the buffer settings are correctly applied

3. **Connection Refused**
   - Verify the Docker container is running
   - Check if the correct ports are exposed
   - Ensure your firewall allows the connection
