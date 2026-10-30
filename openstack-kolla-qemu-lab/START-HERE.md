# 🚀 Upload this project to GitHub — browser only

No Git installation or GitHub token is needed for this first upload.

## 1️⃣ Download and extract

1. Download the ZIP provided with the guide.
2. Windows: right-click → **Extract All**.
3. Open the extracted project folder.
4. Verify `README.md`, `docs`, `assets`, `configs`, `scripts` and `systemd` are present.

**Upload extracted files, not only the ZIP.** GitHub needs the files to render the homepage and guides.

## 2️⃣ Create the repository

1. Sign in to [GitHub](https://github.com).
2. Click **+ → New repository**.
3. Name: `openstack-kolla-qemu-lab`.
4. Description:

   > Single-node OpenStack lab using Kolla Ansible, QEMU, Neutron networking and persistent host NAT, with explained commands and reboot recovery.

5. Choose **Public** for a portfolio after reviewing the sanitized files, or **Private** to review first.
6. Do not initialize another README; this bundle already has one.
7. Do not select a license without deciding how you want the work reused.
8. Click **Create repository**.

## 3️⃣ Upload the contents

1. On the empty repo page, click **uploading an existing file**.
2. Drag the **contents** of the extracted project folder into the upload area, including its subfolders.
3. Confirm `README.md` is at the repository root, not inside an extra wrapper folder.
4. Check `.gitignore` is included. If omitted by your upload flow: **Add file → Create new file**, filename `.gitignore`, paste the provided content.
5. Commit message:

   > docs: document verified single-node OpenStack QEMU lab

6. Click **Commit changes**.

For subsequent uploads use **Add file → Upload files**. UI wording may vary. Browser upload supports individual files up to 25 MiB; this bundle is below that limit. Do not upload qcow2/ISO files.

## 4️⃣ Inspect the result

- [ ] README appears on the Code tab.
- [ ] Architecture image appears.
- [ ] Badges and tables render.
- [ ] Links open the correct guides.
- [ ] Mermaid diagrams render.
- [ ] Code indentation is preserved.
- [ ] Pending work is still labeled pending.

Broken image? Check the exact path `assets/architecture.png` and capitalization.

## 5️⃣ Improve discoverability

Edit the repository's **About** panel and add topics:

`openstack` · `kolla-ansible` · `qemu` · `neutron` · `openvswitch` · `docker` · `homelab` · `linux` · `infrastructure-as-code`

Pin the repo from your GitHub profile. Do not link an unprotected live admin dashboard.

## 6️⃣ Edit a file later

1. Open the file.
2. Click the pencil / **Edit this file**.
3. Edit, then choose **Preview**.
4. Commit with a specific message, for example:

   > docs: record final guest DNS verification

For larger changes, use a branch + pull request to keep a review history.

## 7️⃣ Add sanitized evidence

Suggested uploads under `assets/evidence`:

| Image | Show | Redact |
|---|---|---|
| `services-up.png` | Nova/Neutron UP | Account/project IDs |
| `guest-console.png` | Hostname, IP, tests | Console token/password |
| `reboot-recovery.png` | Service/NAT status | Public IP and unrelated details |

Example Markdown inside a guide:

~~~markdown
![Guest connectivity verification](../assets/evidence/guest-console.png)
~~~

Original session screenshots are not automatically included.

## 8️⃣ Keep it useful

- Short homepage; detailed commands in phase guides.
- Explain **what**, **where**, **why** and **expected result**.
- Use emojis for navigation, not decoration on every line.
- Use tables, Mermaid diagrams and collapsible troubleshooting.
- Update the roadmap only after testing.
- Keep OS images, logs, live config and credentials out.

## ✅ Before the first public commit

- [ ] No actual public IP, UUIDs, passwords, private keys or cloud files.
- [ ] No disk images, database exports or raw debug logs.
- [ ] Final guest network check honestly marked pending.
- [ ] Planned features clearly separate from completed work.
- [ ] Relative links and image paths work.

Deleting a leaked file does not revoke the credential or erase Git history. Rotate exposed credentials.

Official help: [create a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository) · [upload files](https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository).
