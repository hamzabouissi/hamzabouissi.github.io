+++
title = 'Oops...Etcd went down'
date = 2024-05-30T11:12:57+01:00
draft = false
tags = ['Kubernetes','etcd','talos','go','boltdb']
[cover]
  image= '/img/sink_boat.jpg'
+++

## Introduction with 1-Mistake

On that shiny day, I got a project that required deploying a mongodb cluster,
After a few searches, I found percona Operator, moved into installation section and copied the helm install command.

After installing the required charts, I noticed that the pods weren't in "running" state, so as a civilized kubernetes developer I ran "kubectl describe pod_name -n namespace", and it turned out the problem was mongodb cluster requires either 3 or 5 nodes

That's easy right ? !'am using proxmox for my on-prem VMs and talos for my kubernetes OS, therefore I created a new VM with talos as The OS, disabled DHCP and added custom IP address corresponding to 192.168.1.116. Then, to add the VM a new **"worker"** node we use Talos magic wand:

```bash
talosctl apply-config --insecure --nodes 192.168.1.116 --file worker.yaml
```

But life always hides a few surprises for you , I mistakenly ran another command which added the new VM as a controller plane.
The result was **having two etcd members** fighting for their survivals, but our old master "kubernetes" wasn't happy with the result because both etcd instance went down with their kube-apiserver instances

## 2-Mistake

Because I am so smart, I thought the two controller nodes contradicted with etcd's happy state, so I searched for a solution that led me into either removing the newly created node or adding a new one to balance the cluster number, And hell yeah, Iam removing the second idiot VM.

The node deleted from **"proxmox"**,then I thought the cluster will return to healthy state and I will kiss my "IT" girlfriend saying "we're back to normal" making her think I was mad at her for no reason, hmm but life surprises you once again, my dear.

This time, etcd remained in an unhealthy state, claiming it couldn't find the joined node 192.168.1.116

> **_NOTE:_** you can run 'talosctl -n 192.168.1.110 dmesg' to view node logs .

I thought, I saw a talosctl command that invokes a members list and I said to myself if the "list" subcommand exists, then the remove or delete one will exist also, Well it was there of course, but with a different name: "remove-member". however, it didn't work, etcd wasn't responding to my requests even the command : "talosctl members list" wasn't showing anything.

## Solution: Edit the Etcd Snapshotted DB

After long hours of reading Github issues, walking on the beach and talking with friends about rap songs I realized there was no solution other than to reset the controller node along with the etcd data directory.

While reading the documentation on Talos "Disator Recovery", I was made aware of the snapshot idea but wasn't thinking outside the box.
Until I thought of editing the etcd database, talosctl didn't have a built-in command for this kind of operation so I went for snapshotting the database and inspecting it to see what I can edit there to remove the call for the our beloved dead node.

Let's start with taking a snapshot, there are two commands referenced in the documentation but we will go with latter because etcd is unreachable

```bash
talosctl -n 192.168.1.110 cp /var/lib/etcd/member/snap/db .
```

I ran the 'file' command to check file type which returned: **data**, hmm well this isn't enough linux, thanks for your time, On my second search on google I found the **bbolt file type** and there is this tool **bbolt** for inspecting bbolt databases, Cool now we playing our cards right.

After a few tries, I found a bucket called "members"

```shell
bbolt buckets db
    alarm
    auth
    authRoles
    authUsers
    cluster
    key
    lease
    members <- this one
    members_removed
    meta
```

hmm, I procceded into list members bucket keys and inspecting each value

```shell
bbolt keys db members
    3cf1b5e76f18a513
    920c1b791dddb17e
```

```shell
bbolt get db members 3cf1b5e76f18a513
    {"id":4391491117268903187,"peerURLs":["https://192.168.1.110:2380"],"name":"talos-zrr-lqe"}
```

```shell
bbolt get db members 920c1b791dddb17e
    {"id":10523816636264067454,"peerURLs":["https://192.168.1.116:2380"],"isLearner":true}
```

here we go Kogoro Mouri, I found the culprit this member with key 920c1b791dddb17e must be deleted, so let's call the POLICE(ChatGPT) to exceel him out.
We asked ChatGPT for deleting the key **920c1b791dddb17e** from members buckets

```go
package main

import (
    "log"

    "go.etcd.io/bbolt"
)

func main() {
    // Open the BoltDB database file
    db, err := bbolt.Open("test.db", 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // The key we want to delete
    key := []byte("920c1b791dddb17e")

    // Update the database to delete the key from the "members" bucket
    err = db.Update(func(tx *bbolt.Tx) error {
        // Get the bucket
        bucket := tx.Bucket([]byte("members"))
        if bucket == nil {
            return bbolt.ErrBucketNotFound
        }

        // Delete the key
        return bucket.Delete(key)
    })

    if err != nil {
        log.Fatalf("Could not delete key: %v", err)
    } else {
        log.Println("Key deleted successfully")
    }
}
```

Now, we can reset the etcd node and then recover from the backup db

```bash
talosctl -n 192.168.1.110 reset --graceful=false --reboot --system-labels-to-wipe=EPHEMERAL
```

wait until the node become in preparing mode and run:

```bash
talosctl -n <IP> bootstrap --recover-from=./db.snapshot --recover-skip-hash-check
```

finally run, 'kubectl get nodes -o wide' and you should see your nodes

'kubectl get pods' to check your cluster previous state returned to normal
