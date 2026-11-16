# Vultr Custom ISO: How Do You Upload, Boot, and Install Your Own Image? (10 GB, Two-ISO, and No-Windows Limits Explained)

Vultr accepts your own `.iso`, but it won't take a file from your laptop. You hand it a public, direct URL, Vultr pulls the image into your account, and from there it can boot a new instance or attach to one you already run. Six rules decide whether that URL gets through, and the mistake that bites hardest comes after the install finishes: an ISO left attached sends the server back into the installer on every reboot.

This walks through the limits first, then the upload in the console, CLI, and API, the install over VNC, the detach step people skip, mounting an ISO on an existing instance, and what the whole exercise costs on an hourly bill.

> **Quick read:** this is for the reader with a Kali build, pfSense, OpenBSD, or a rescue disk that isn't in Vultr's standard library. Before you start you need a direct link with no auth and no query string, a file under 10 GB ending in `.iso`, and fewer than two ISOs already sitting in the account. 👉 [Open a Vultr account to upload your ISO](https://bit.ly/VuLtr)

## The six rules an ISO has to pass before Vultr will pull it

Work through these before you touch the console. Each one is a hard stop, not a warning.

1. **Under 10 GB.** The file must be smaller than 10 GB (10240 MB).
2. **`.iso` on the end.** The filename has to end with the `.iso` extension, so a `.img` or a zip won't work.
3. **An absolute URL with no authentication and no query parameters.** A signed S3 link with `?X-Amz-Signature=...` on the end fails here.
4. **A direct link with no HTTP redirects.** If your URL 302s to a CDN host, Vultr won't follow it.
5. **Two ISOs per account, maximum.** You store at most two at a time. Delete one and re-upload when you need a third.
6. **No Windows.** Vultr does not permit uploading Windows ISOs at all.

Rules 3 and 4 account for most failed attempts. They're also the two that a file host will silently violate: plenty of "direct download" links are really a redirect to a mirror, and plenty of storage providers hand you a pre-signed URL with a query string attached.

### When the link is the problem: the Object Storage workaround

Vultr's own documentation gives the fix, and it's worth knowing because it's the step most tutorials leave out. Download the ISO to your local machine, upload it to Vultr Object Storage, set the object to publicly readable, and use that URL for the ISO upload. You end up with a clean absolute path, no auth, no query parameters, no redirect, which satisfies rules 3 and 4 in one move.

That round trip costs you a local download and a re-upload. For a 4 GB image on a slow connection, budget the time before you start.

## Uploading the ISO: console, CLI, or API

All three routes need one of three things first: the Vultr Console, an API key, or the Vultr CLI. Pick whichever matches how you already work.

**In the console:** open **Orchestration** in the left sidebar, select **ISOs**, click **Add ISO +**, paste the URL into **Enter remote URL**, optionally add a **Name**, then click **Create ISO +**.

**With the CLI:**

console
$ vultr-cli iso create --url ISO_URL
$ vultr-cli iso get ISO_ID


The create call returns an `id`. Feed that into the get call and confirm the `STATUS` field reads **completed**.

**With the API:**

console
$ curl -sS --fail --compressed "https://api.vultr.com/v2/iso" \
  -X POST \
  -H "Authorization: Bearer ${VULTR_API_KEY}" \
  -H "Content-Type: application/json" \
  --data '{
    "url" : "ISO_URL"
  }' \
  | jq .


Then check it with a `GET` to `https://api.vultr.com/v2/iso/ISO_ID`, where the `status` field should read **complete**.

| Route | What you run | What confirms it worked |
| --- | --- | --- |
| Console | Orchestration → ISOs → Add ISO + | Status flips **Pending** → **Complete** |
| CLI | `vultr-cli iso create --url ISO_URL` | `vultr-cli iso get ISO_ID` shows `STATUS: completed` |
| API | `POST /v2/iso` with a JSON `url` body | `GET /v2/iso/ISO_ID` shows `status: complete` |

The transfer moves at whatever speed the source server gives you, so a small ISO from a fast mirror can land in minutes while a 9 GB image from a slow host can sit in Pending for a long while. Don't deploy against an ISO that hasn't reached Complete. 👉 [Create a Vultr account that can hold your ISO](https://bit.ly/VuLtr)

## Booting a new instance from the ISO

In the console the path runs: **Compute** → **Instances** → **Create Instance +**, then choose a **Type**, a **Location**, and a **Plan**. At **Configure Software**, under **Image Selection and Configuration**, click the **ISO** tab and pick your custom ISO from the list.

Before deploying, set the optional pieces you'd normally set on any server: **SSH Keys**, a **Firewall Group**, a **Server Hostname**, an optional **Server Label**, and any **Additional Features** you want. Review the Deploy Summary and hit **Deploy**.

Wait for the instance to show **Running** in the list, click its name, then click the **View Console** icon. That VNC console is where the actual installation happens: partitioning, package selection, the whole interactive installer. There is no unattended path here; you're driving a remote screen.

The CLI reaches the same place from a different direction:

console
$ vultr-cli iso list
$ vultr-cli instance create --region REGION --plan PLAN --iso ISO_ID --label EXAMPLE_LABEL --host EXAMPLE_HOST


The API version lists ISOs with `GET /v2/iso`, picks the `id` out of the response, and posts to `/v2/instances` with a body carrying `region`, `plan`, `label`, `iso_id`, `backups`, `hostname`, and `tags`. Either way, you still open the VNC console in the web panel to finish the install. Command-line deployment gets the machine created; it doesn't get the OS installed.

## Detach the ISO, or the instance reinstalls forever

This is the one step to put on a checklist. Vultr's documentation is blunt about it: detach the custom ISO when installation is complete, because if it stays attached the instance boots from the ISO after every reboot.

Picture the failure. You install pfSense, configure it, reboot to test, and the machine comes back up in the installer. Nothing you configured is gone, but you can't reach it, because the boot order is still pointing at installation media.

In the console: **Settings** tab → **Custom ISO** in the left menu → **Remove ISO**.

Over the API it's a single call:

console
$ curl -sS --fail --compressed "https://api.vultr.com/v2/instances/INSTANCE_ID/iso/detach" \
  -X POST \
  -H "Authorization: Bearer ${VULTR_API_KEY}"


Vultr detaches the ISO and reboots the instance automatically, so the machine comes back on the OS you just installed.

## Mounting an ISO on an instance you already have

The same feature works on running instances, which is what makes it useful for repair work rather than just fresh installs. The console entry lives under the instance's **Settings** tab → **Custom ISO**, where you choose either an ISO already in your account or one from the ISO Library, then trigger the attach-and-reboot action. The exact button label is worth confirming on screen when you get there; the location under Settings → Custom ISO is the stable part.

Expect a reboot as part of attaching. Don't do this to a production box without a maintenance window, and don't do it at all if you haven't confirmed you can get back in through VNC afterward.

## What the install time actually costs you

Vultr bills every server hourly, and the monthly cap depends on the product category:

| Server category | Hours billed per month |
| --- | --- |
| GPU servers and VX1 Cloud Compute | 730-hour month, prorated to the actual month |
| All other servers | Capped at 672 hours per month |

The minimum billing unit is one hour, and charges are invoiced on the 1st of each month. Two consequences matter for anyone doing an ISO install:

**A powered-down server still costs money.** Stopped instances keep reserving RAM, SSD storage, IP aliases, and vCPU, so they keep accruing charges until you destroy them. If you spin up a test box, install an ISO, and shut it down to "save money," the meter keeps running. The **DESTROY** button in the customer portal is what stops it.

**The install itself is billed.** Sitting at a VNC console for an hour partitioning disks and picking packages is an hour of running a server. On a fresh image that's a rounding error; across a fleet it isn't.

The fix is the snapshot workflow Vultr documents for this feature: customize once, snapshot the finished server, then deploy the rest from that image in any region. You pay the interactive install cost exactly once instead of once per machine. 👉 [Deploy an instance and watch the hourly meter](https://bit.ly/VuLtr)

## Rescue work, regions, and the built-in ISO library

Custom ISO uploads are available in all 33 Vultr regions across six continents, so an image you upload once can be mounted in Sydney, Frankfurt, São Paulo, or Johannesburg without re-uploading. That's the practical argument for the feature if you run machines in more than one location.

The rescue case is the one people discover too late. When kernel changes or a non-standard upgrade leave a system corrupted or unbootable, you can attach a rescue ISO and get back in to fix the configuration or pull data off the disk. Vultr's framing is that this ensures your data is not automatically lost. It's a better position than opening a support ticket and hoping.

You don't have to supply the rescue disk yourself. Vultr's curated ISO library currently lists:

| ISO | Version / architecture |
| --- | --- |
| Finnix | 124 x64 |
| GParted | 1.4.0-5 x64 |
| Hiren's BootCD PE | 1.0.2 x64 |
| SystemRescue | 9.03 x64 |

Those cover partition repair, data recovery, and general rescue work without occupying one of your two custom ISO slots.

## Questions that come up before the first upload

**Can I upload a Windows ISO?** No. Vultr doesn't permit it.

**Can I run several instances from one uploaded ISO?** Yes, one ISO in the account can deploy as many instances as you like, one at a time or in parallel, as long as you stay inside the two-ISO storage limit.

**What if my ISO is over 10 GB?** It won't upload. You'll need a slimmer build or a different distribution image.

**Does the ISO survive if I delete the instance?** Yes. ISOs live in the account, separate from instances, and stay there until you delete them, which is also how you free a slot for the third image.

**Is the server ready to use after the install?** It's whatever you installed, and nothing more. Vultr's default images ship as a clean OS with no cPanel/WHM, WordPress, PHP, or MySQL preinstalled; One-Click Apps cover that if you want it. A custom ISO is even barer, since it's your build.

## Which route to take

If you're installing a system once, deploy from your ISO, run the installer over VNC, detach the ISO, then snapshot the finished machine and build everything else from that snapshot. If the OS you want is already in Vultr's standard library or the ISO library above, skip this entirely. The custom ISO path costs you an interactive install and a manual detach, and it only pays off when the image is something the standard library doesn't carry.

Two things to confirm before you click Deploy: your URL passes all six rules, and the account isn't already holding two ISOs. Both are quiet failures that cost you twenty minutes of confusion.

👉 [Upload your custom ISO on Vultr](https://bit.ly/VuLtr)
