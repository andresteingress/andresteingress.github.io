---
layout: post
title: GCloud - Container-Optimized Images
categories:
- java
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

This article is about Google Cloud container-optimized images and the auto update functionality.

### Container-Optimized Images in GCloud

As its name implies, container-optimized images is a special image type in Google Cloud. It is an image provided by Google being optimized for Docker containers. As a matter of fact, it can barely be used for anything else except running containers. 

It uses the Container-Optimized OS which is based on the open-source project [Chromium OS](https://www.chromium.org/chromium-os). If you are running your applications in Docker containers and host in Google Cloud, this is certainly the image type you should prefer. It has several advantages compared to Debian or Ubuntu images, being available in Google Cloud too. One of the advantages is its _auto update_ functionality (the other big advantage being more advanced security). 

### Create Snapshots

Before we talk about auto updates, you should be aware of the concept of snapshots. Always create [snapshots](https://cloud.google.com/compute/docs/disks/create-snapshots) of your persistent disks before running any experiments/commands which might have an influence on your productive systems. Creating snapshots is rather easy with Google Cloud. You can either use the Google Console web UI or the `gcloud` utility to do so: 

{% highlight shell %}

$ gcloud compute snapshots list

# Listing snapshots ...

$ gcloud compute disks snapshot DISK

# Creating snapshot for persistent disk named DISK

{% endhighlight %}

If you have a look on GitHub, you will find various "shell script projects" with shell scripts to be scheduled with cron jobs. One of them is [google-cloud-snapshot](https://github.com/jacksegal/google-compute-snapshot). We use an adapted version based on that GitHub project to create daily snapshots of our container images:

{% highlight shell %}

0 5 * * * root /opt/google-compute-snapshot/gcloud-snapshot.sh >> /var/log/cron/snapshot.log 2>&1

{% endhighlight %}

Note that we run the `gcloud-snapshot.sh` script directly inside the Google Cloud container VM instances. We usually use one Google Cloud VM instance to run multiple Docker containers. You could also run one Docker container inside one Google Cloud VM instance. You can even configure the Docker instance with [cloud-init](https://cloudinit.readthedocs.io/en/latest/index.html) in these scenarios (`cloud-init` is pre-installed in COS images), [as described here](https://cloud.google.com/container-optimized-os/docs/how-to/create-configure-instance?hl=en#using_cloud-init). However, we do not apply that pattern in our projects yet, we have one or more Docker containers inside one Google Cloud VM.

As the container-optimized image can basically only run Docker containers, we also have to run our cron job inside a dedicated Docker container. To get more information on how to run cron jobs inside Docker, [read this blog post](https://www.ekito.fr/people/run-a-cron-job-with-docker/).

### Auto Updates

One advantage of the container-optimized image is the auto update functionality. The image uses a so-called active-passive root partition scheme. That means, updates will not being executed with a package manager like in other operating systems but as an image entirely in the passive root partition. Once the new operating system version has been downloaded into the passive partition, the VM instance is eligable for re-booting with the new version. That means, after the next instance reboot the image will be run on the newly updated image, the passive partition will become the active root partition. 

In order to find out information about the currently running container-optimized OS, you can use the `update_engine_client` tool inside the Google Cloud VM shell:

{% highlight shell %}

# Connect to your Google Cloud instance

$ sudo gcloud compute ssh INSTANCE

# Run update_engine_client

$ sudo update_engine_client --help
Chromium OS Update Engine Client

  --app_version  (Force the current app version.)  type: string  default: ""
  --block_until_reboot_is_needed  (Blocks until reboot is needed. Returns non-zero exit status if an error occurred.)  type: bool  default: false
  --can_rollback  (Shows whether rollback partition is available.)  type: bool  default: false
  --channel  (Set the target channel. The device will be powerwashed if the target channel is more stable than the current channel unless --nopowerwash is specified.)  type: string  default: ""
  --check_for_update  (Initiate check for updates.)  type: bool  default: false
  --cohort_hint  (Set the current cohort hint to the passed value.)  type: string  default: ""
  --eol_status  (Show the current end-of-life status.)  type: bool  default: false
  --follow  (Wait for any update operations to complete.Exit status is 0 if the update succeeded, and 1 otherwise.)  type: bool  default: false
  --help  (Show this help message)  type: bool  default: false
  --interactive  (Mark the update request as interactive.)  type: bool  default: true
  --is_reboot_needed  (Exit status 0 if reboot is needed, 2 if reboot is not needed or 1 if an error occurred.)  type: bool  default: false
  --last_attempt_error  (Show the last attempt error.)  type: bool  default: false
  --omaha_url  (The URL of the Omaha update server.)  type: string  default: ""
  --p2p_update  (Enables ("yes") or disables ("no") the peer-to-peer update sharing.)  type: string  default: ""
  --powerwash  (When performing rollback or channel change, do a powerwash or allow it respectively.)  type: bool  default: true
  --prev_version  (Show the previous OS version used before the update reboot.)  type: bool  default: false
  --reboot  (Initiate a reboot if needed.)  type: bool  default: false
  --reset_status  (Sets the status in update_engine to idle.)  type: bool  default: false
  --rollback  (Perform a rollback to the previous partition. The device will be powerwashed unless --nopowerwash is specified.)  type: bool  default: false
  --show_channel  (Show the current and target channels.)  type: bool  default: false
  --show_cohort_hint  (Show the current cohort hint.)  type: bool  default: false
  --show_p2p_update  (Show the current setting for peer-to-peer update sharing.)  type: bool  default: false
  --show_update_over_cellular  (Show the current setting for updates over cellular networks.)  type: bool  default: false
  --status  (Print the status to stdout.)  type: bool  default: false
  --update  (Forces an update and waits for it to complete. Implies --follow.)  type: bool  default: false
  --update_over_cellular  (Enables ("yes") or disables ("no") the updates over cellular networks.)  type: string  default: ""
  --watch_for_updates  (Listen for status updates and print them to the screen.)  type: bool  default: false

{% endhighlight %}

The first helpful command is `sudo update_engine_client --status`. It can be used to show the current update status of the image:

{% highlight shell %}
$ sudo update_engine_client --status
[1008/074003:INFO:update_engine_client.cc(506)] Querying Update Engine status...
LAST_CHECKED_TIME=1507447598
PROGRESS=0.000000
CURRENT_OP=UPDATE_STATUS_IDLE
NEW_VERSION=0.0.0.0
NEW_SIZE=0
{% endhighlight %}

In order to force an update on the currently running instance (without rebooting though), you can run:

{% highlight shell %}
$ sudo update_engine_client --update
{% endhighlight %}

`--status` and `--update` are the two commands which will be in use the most. 

Please take care that you have a snapshot before you restart the VM. In case you should run into a problem with the updated COS image version, you can still try to rollback with 

{% highlight shell %}
$ sudo update_engine_client --rollback
{% endhighlight %}

However, if the `update_engine_client` isn't working, you can create a new VM instance with a persistent disk based on the snapshot and then disable the auto update functionality via the `cos-update-strategy` meta key:

{% highlight shell %}
# Create a new persistent disk from the snapshot

$ sudo gcloud compute disks create restored-disk --source-snapshot=my-snapshot

# Create a new VM instance based on the restored disk with auto updates disabled

$ gcloud compute instances create restored-instance --disk name=restored-disk boot=yes --metadata cos-update-strategy=update_disabled --zone europe-west1-c
{% endhighlight %}  

The important part is adding `--metadata cos-update-strategy=update_disabled` to the newly created VM instance. This will disable the auto update mechanism and you won't run into the same update again once starting the new instance. 

There are more options to use when running `gcloud compute instances create`. To get an overview, you can run `gcloud compute instances create --help`.

### Summary

This article gave an introduction to Google Cloud container-optimized images running on a variant of Chromium OS. This Google provided image has an auto update functionality which updates the image weekly on an automic basis. This article shows how to retrieve the auto udpate status and how to recover from potential problems if an automic update causes problems. 

