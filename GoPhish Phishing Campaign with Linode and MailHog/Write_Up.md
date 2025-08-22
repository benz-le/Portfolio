# GoPhish Phishing Campaign with Linode and MailHog

These are my notes for creating a phishing campaign using **GoPhish**, hosted on a Linode VM, with **MailHog** as a fake SMTP server for safe testing.  

- **Linode** is an easy-to-use cloud platform for spinning up virtual machines.  
- **SMTP (Simple Mail Transfer Protocol)** is the standard for sending emails. Instead of delivering real emails, **MailHog** intercepts them for review, simulating an SMTP server for development and testing.  

---

## Set Up Linode

### Step 1: Create a Linode
1. Sign up for a [Linode account](https://www.linode.com/lp/free-credit-100) (includes $100 free credit).  
2. Click **Create Linode**.  
   ![Create Linode](image-placeholder)  
3. Choose the closest region.  
4. Select a Linux distribution (this guide uses **Ubuntu 24.04 LTS**).  
   ![Choose Region and Linux Distribution](image-placeholder)  
5. In the **Linode Plan** section:  
   - Go to the **Shared CPU** tab.  
   - Select the smallest plan: **Nanode 1 GB ($5/month)**.  
   ![Select Plan](image-placeholder)  
6. In **Security**, create a strong root password.  
7. Under **Add-ons**, enable **Private IP**.  
   ![Linode Add-ons](image-placeholder)  
8. Click **Create Linode**. The status will change to **Running** when ready.  
   ![Linode Dashboard](image-placeholder)  

### Step 2: Connect and Set Up the VM
1. Open **PowerShell** on Windows and connect via SSH (IP from Linode dashboard):  
   ```powershell
   ssh root@<ip-address>
   ```  
2. Accept the prompt by typing `yes`.  
3. Enter the root password created earlier.  
   ![SSH via PowerShell](image-placeholder)  
4. Update packages:  
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```  
5. Install unzip:  
   ```bash
   sudo apt install unzip -y
   ```  

---

## Install and Configure GoPhish

### Step 3: Download GoPhish
1. Go to the [GoPhish GitHub releases](https://github.com/gophish/gophish).  
2. Copy the link to the latest **Linux 64-bit** release.  
   ![Download GoPhish](image-placeholder)  
3. Create a directory and download GoPhish:  
   ```bash
   mkdir gophish && cd gophish
   wget <link>
   unzip <file>.zip
   ```  

### Step 4: Configure GoPhish
1. Open `config.json`:  
   ```bash
   nano config.json
   ```  
2. Change the `"listen_url"` to:  
   ```json
   "listen_url" : "0.0.0.0:<desired-port>"
   ```  
   Save and exit (**Ctrl+X → Y → Enter**).  
3. Make GoPhish executable:  
   ```bash
   chmod +x gophish
   ```  
4. Run GoPhish:  
   ```bash
   sudo ./gophish
   ```  
5. Copy the auto-generated **admin username and password** from the output.  
   ![GoPhish Output](image-placeholder)  

---

## Create a GoPhish Campaign

### Step 5: Access the Admin Page
1. Navigate to:  
   ```
   https://<ip-address>:<port>
   ```  
   Example: `https://69.164.203.224:3333`  
2. Log in with the credentials from the GoPhish output.  
3. Change the password when prompted.  

### Step 6: Add Users & Groups
1. Go to **Users & Groups** → **New Group**.  
2. Add test users and emails.  
   ![Add Users](image-placeholder)  

### Step 7: Create an Email Template
1. Go to **Email Templates** → **New Template**.  
2. Copy a real email (e.g., Gmail) → **Show Original** → **Copy to Clipboard**.  
3. Import it into GoPhish → Save.  
4. For **Envelope Sender**, use any test address (spoofing is configured later).  

### Step 8: Add a Landing Page
1. Go to **Landing Pages** → **New Page**.  
2. Import a login page (e.g., `login.wordpress.org`).  
3. Save.  

### Step 9: Configure a Sending Profile
> [!TIP]  
> Gmail and Outlook often require **App Passwords** for SMTP authentication.  
> Helpful guides: [Gmail](https://www.youtube.com/watch?v=QR3Pp1D1y2I), [Outlook](https://www.youtube.com/watch?v=M9b3O_MjTCg).  

1. Go to **Sending Profiles** → **New Profile**.  
2. Add:  
   - Name  
   - SMTP From address (Envelope Sender)  
   - Email + password  
   - SMTP server & port (e.g., `smtp-mail.outlook.com:587`)  
3. Save.  

### Step 10: Launch a Campaign
1. Go to **Campaigns** → **New Campaign**.  
2. Select your group, template, and sending profile.  
3. For URL, enter your GoPhish IP (e.g., `http://69.164.203.224`).  
4. Click **Launch Campaign**.  

---

## Install and Use MailHog

> [!CAUTION]  
> Spoofed email addresses won’t be delivered to Gmail/Outlook inboxes. They will only appear in MailHog for safe review.  

### Step 11: Install MailHog
1. Download the [MailHog release](https://github.com/mailhog/MailHog).  
2. Create a directory and install:  
   ```bash
   mkdir mailhog && cd mailhog
   wget <link>
   chmod +x MailHog_linux_amd64
   ```  
3. Run MailHog in a background session:  
   ```bash
   screen -S mailhog
   sudo ./MailHog_linux_amd64
   ```  
4. Note the MailHog web interface address (default: port `8025`).  

### Step 12: Access MailHog Inbox
- Open in browser:  
  ```
  http://<ip-address>:8025
  ```  

### Step 13: Configure GoPhish with MailHog
1. In **Sending Profiles**, set SMTP From (e.g., `support@wordpress.com`).  
2. Use the MailHog SMTP server:  
   ```
   <ip-address>:1025
   ```  
3. Save the profile.  
4. Re-run the campaign. Emails will now appear in the MailHog inbox.  

---

## Clean Up
- Shut down or delete the Linode VM to avoid charges.  

---

## Notes
- AWS and Azure can be used instead of Linode.  
- Any email provider (Gmail, Outlook, etc.) can integrate with GoPhish.  
- Explore other phishing simulation tools: **Proofpoint**, **Microsoft Defender**, **KnowBe4**.  
