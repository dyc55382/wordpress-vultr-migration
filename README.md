# migrate wordpress to vultr: 4 methods compared, the zero-downtime trick, and the DNS and SSL steps done right

Moving a WordPress site to Vultr is a well-documented, mostly mechanical process. Vultr publishes official guides for four different migration routes, and they differ in three things: what the plugin costs, whether your site goes offline, and how much server work you do by hand. For a typical blog or business site, the short version is this: deploy Vultr's Marketplace WordPress app, move everything across with the free All-in-One WP Migration plugin, point your DNS at the new server, and install a Let's Encrypt certificate with certbot. Most people finish in an afternoon.

> **In a hurry?** For most sites: deploy the Marketplace WordPress app, migrate with the free All-in-One WP Migration plugin during a brief maintenance window, then switch DNS. If even a few minutes of downtime is unacceptable, skip straight to the hosts-file method below. 👉 [deploy a Vultr WordPress server](https://www.vultr.com/apps/wordpress/?ref=7268283)

## The four ways to migrate WordPress to Vultr

Vultr's own documentation covers four routes, and picking the right one upfront saves you from restarting halfway through. Here's how they compare:

| Method | Plugin cost | Site offline? | Technical level | Best suited for |
| --- | --- | --- | --- | --- |
| All-in-One WP Migration, standard | Free version handles most sites | Yes, short window | Low | Blogs, brochure sites, most small businesses |
| All-in-One WP Migration, hosts-file variant | Free version | No | Medium | Stores and busy sites that can't blink |
| Jetpack Backup | Paid subscription | No, during the move | Medium | Sites already on Jetpack, or anyone wanting point-in-time cloning |
| UpdraftPlus | Free version | Brief, during DNS switch | High | Admins comfortable with MySQL and FTP by hand |
| BackupBuddy | Paid | Brief, during DNS switch | High | Existing BackupBuddy owners |

Strictly speaking, Vultr documents four methods. The hosts-file route is a variant of the first one that eliminates downtime, but it changes enough steps that it deserves its own row in the table.

A few things worth knowing before you commit to one:

- All-in-One WP Migration is the method where Vultr explicitly says the free version is enough for most websites. It exports your entire site, themes and plugins included, into a single file.
- Jetpack Backup and BackupBuddy are premium products. Jetpack Backup requires buying a subscription plan before the clone feature unlocks; BackupBuddy is a paid plugin outright.
- UpdraftPlus works on the free version, but it's the most hands-on. You create the database manually, run an FTP server, and restore files over SSH.

Every method ends the same way, by the way. DNS points at the new server, a Let's Encrypt certificate goes on with certbot, and the old server stays shut down but not destroyed until you've confirmed the new site works. The differences are all in the middle.

## Step zero: back up, and find your Vultr credentials

Two things to sort out before touching either server.

Back up the old site first. Vultr's guide puts this as step one with a do-not-skip warning attached, and for good reason: a migration that goes sideways with no backup is a much worse afternoon. Take a complete server image if your current host allows it, or run a backup plugin such as Updraft or Jetpack on the WordPress side.

Then note down your Vultr login details. Once you deploy the Marketplace WordPress app, the username and password sit under Application Information on your server's information page in the Vultr console. You'll need them for a pop-up login that surprises people, which brings us to the walkthrough.

## The standard method: All-in-One WP Migration with a short maintenance window

This is the route Vultr recommends when your site can afford a brief offline period, and it's the one most people should use. The plugin does the heavy lifting; you mostly click buttons.

1. On the old site, install All-in-One WP Migration from Plugins > Add New, activate it, and go to All-in-One WP Migration > Export. Choose Export To > File. The export includes your themes and plugins, so there's nothing to reinstall later. Download the file to your computer when the prompt appears.
2. Shut down the old server. Shut down, don't destroy. Your site is now offline; this is your maintenance window.
3. Deploy a fresh Marketplace WordPress server and update your DNS to its new IP address.
4. Open the new site's admin area at yourdomain.com/wp-admin. Here's the pop-up: Vultr's WordPress app puts an extra login box in front of the standard WordPress sign-in. The credentials come from Application Information, and it's easy to mistake this box for something being broken. If you'd rather not keep that layer, Vultr's Marketplace documentation has a "Remove wp-admin Security" section that walks you through taking it off.
5. Run the WordPress installation wizard. Don't agonize over the site title, username, or email here, because the import overwrites all of it. Do remember the temporary password, since you need it to log in once.
6. Install the same All-in-One WP Migration plugin on the new site, then go to Import > Import From > File and upload the export file. Confirm with Proceed.
7. When the import finishes, click Save permalinks structure. A new tab opens. Log in with your old site's admin credentials, compare the permalink settings against the old site, adjust if needed, and save. This step matters: a mismatched permalink structure is what turns a successful import into a site full of 404s.
8. Back on the import tab, click Finish and log out.

SSL comes last. For a free Let's Encrypt certificate, SSH into the server and run certbot with your own domain and email:


certbot --nginx --redirect -d www.example.com -m admin@example.com --agree-tos --no-eff-email


Using a commercial certificate instead? Replace the server.crt and server.key files in /etc/nginx/ssl/ with your vendor's files.

## Zero downtime: the hosts-file method

This one works by splitting DNS in two. The public internet keeps resolving your domain to the old server, so visitors never notice anything, while your own computer resolves it to the new Vultr IP. That lets you build and test the new site in private.

Deploy the target server first and note its IP. Then edit the hosts file on your computer:

- **Windows:** press Win + S, open Notepad as administrator, and open C:\Windows\System32\drivers\etc\hosts
- **macOS or Linux:** open a terminal and edit /etc/hosts

Add a line at the bottom mapping your domain to the new server's IP, then verify it took effect: ping your domain from a terminal and check that the response shows the new IP. If it still shows the old one, your network stack or browser is caching. Close the browser, reset the network, or reboot and try again.

The SSL step has a twist here. A normal certbot run validates your domain over public DNS, which still points at the old server, so it would fail. Instead, run certbot with the certonly and --manual options. It pauses and asks you to add a TXT record to your domain's DNS. Add the record at your DNS provider, confirm it has propagated using dig or dnschecker.org, and only then press Enter to continue. Certbot reports where the new certificate files landed. Archive the existing Nginx certificate, link the new one where Nginx expects it, restart Nginx, and load your site over HTTPS on your machine. Thanks to the hosts entry, you should see the new site working with a valid certificate.

From here, import your content exactly as in the standard method. Once everything checks out, update your public DNS to the new IP. That's the moment your visitors move over, without ever having seen an error page.

## Jetpack, UpdraftPlus, or BackupBuddy

Vultr documents these three separately, each for a specific kind of user. All three are plugin-based moves, so here's just what makes each one different.

**Jetpack Backup** is the polished, paid option. You buy a Backup plan on the old site, then prep the new Vultr server over SSH: create a sudo user, and temporarily open up the web root permissions (chmod 777, then chown to www-data) so Jetpack can write files. You tighten those permissions again once the site is secured. The actual migration happens from your WordPress.com dashboard under My Sites > Switch Site > your site > Settings > General > Site tools > Clone. Enter the destination URL, choose SSH/SFTP credentials, and fill in the Vultr server's IP, port 22, and your sudo user. Point it at /var/www/html/, pick Clone current state, and start. Progress shows up in the Activity Log at cloud.jetpack.com, and you'll get a confirmation when the clone lands. Afterwards: update DNS, change the server_name directive in Nginx's wordpress_http.conf from the wildcard to your real domain, and run sudo certbot -d www.example.com --agree-tos. Test auto-renewal with certbot renew --dry-run; if that passes, certificates renew automatically every 90 days.

**UpdraftPlus** is the free, do-it-yourself route. On the Vultr server you create the WordPress database yourself in MySQL, install vsftpd for FTP, and open ports 21 and 20 in the firewall. Then point UpdraftPlus on the old site at the Vultr server as its remote storage (the FTP server field is simply your Vultr IP), test the connection, and run a full backup. On the server, you unzip the backup archives into a wp-content folder, gunzip the database file, and restore it into MySQL. One failure mode is worth knowing in advance: if the import dies with ERROR 1067 (42000), Invalid default value for 'user_registered', add these two lines at the top of the database file and import again:


SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
SET time_zone = "+00:00";


After that it's file shuffling: remove the default wp-content and wp-config.php from the web root, copy your restored wp-content in, and create a fresh wp-config.php from the sample file, pointing it at the database you just restored.

**BackupBuddy** makes sense mainly if you already own it. It's a paid plugin, and Vultr's guide for it targets a self-built Ubuntu 20.04 LAMP server, Apache rather than the Marketplace app's Nginx, so expect to assemble the stack yourself. The flow: set an email and a restore password in BackupBuddy, add your Vultr server as an SFTP destination, run a Complete Backup, and send both the backup file and importbuddy.php to that destination. On the new server, empty /var/www/html, move both files in, hand the directory to www-data, and open HTTP and HTTPS in the firewall. Then load your-server-IP/importbuddy.php in a browser, authenticate with the restore password, and walk through the restore. When it asks for the site URL, enter the server IP for now; you'll switch it to your domain later in the WordPress settings. Finish with ImportBuddy's cleanup, then point your domain at Vultr.

## FAQ

**How much does migrating to Vultr cost?**
The plugins are the easy part: All-in-One WP Migration and UpdraftPlus both work on free versions, while Jetpack Backup and BackupBuddy are paid products. The Vultr side is whatever cloud server plan you deploy. The Marketplace WordPress app runs on a standard Vultr instance, and pricing scales with the resources you pick. 👉 [check Vultr's current plans and pricing](https://bit.ly/VulTR)

**Will my site go offline?**
Only under the standard All-in-One WP Migration method, and only for the length of your maintenance window. The hosts-file method keeps the site fully available. The Jetpack, UpdraftPlus, and BackupBuddy routes switch DNS at the end, so any interruption is limited to DNS propagation.

**Do I have to reinstall my themes and plugins?**
No. All-in-One WP Migration packs themes and plugins into the same export file as your content. UpdraftPlus archives them as separate zip files (plugins, themes, uploads) that you restore onto the new server.

**What login do I use on the new server?**
Two layers. First there's the extra sign-in box in front of wp-admin, which uses the Application Information credentials from your server's info page. Then WordPress itself: the temporary account from the install wizard, which becomes your old admin account once the import completes.

**What happens to my old server?**
On the standard method you shut it down before the import, but leave it in place until you've tested the new site thoroughly. Vultr's guide only calls it safe to destroy the old server after testing is done, and that advice applies to every method here.

## Before you flip DNS: a final checklist

The last stretch is the same no matter which route you took, so run through this before going live:

- Export file downloaded and stored somewhere safe (standard and hosts-file methods), or backup archives confirmed sitting on the new server (UpdraftPlus and BackupBuddy routes).
- Application Information credentials noted, for the wp-admin pop-up and, on the UpdraftPlus route, the MySQL database you created.
- Your old site's permalink structure written down or screenshotted, so you can match it after the import.
- Login access to your DNS provider ready, and propagation checked with a tool like dnschecker.org before you call the move finished.

Pick the standard method if a quiet hour is fine. Pick the hosts-file method if it isn't. If you haven't deployed the new server yet, 👉 [set up WordPress on Vultr](https://www.vultr.com/apps/wordpress/?ref=7268283) and you'll be ready to start the moment your backup finishes.
