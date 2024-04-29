<!-- note marker start -->
NOTE: This repo contains only the documentation for the private BoltsOps repo code.
Original file: https://github.com/boltops-learn/google-ilb-as-next-hop/blob/main/README.md
The docs are publish so they are available for interested subscribers.
For access to the source code, you can become a BoltOps subscriber.
See: https://learn.boltops.com

<!-- note marker end -->

# Google Internal Load Balancer as Next Hop with NAT Servers

Project to show how to set up Internal Load Balancer as Next Hop issue. This based on [Setting up Internal TCP/UDP Load Balancing for third-party appliances](https://cloud.google.com/load-balancing/docs/internal/setting-up-ilb-next-hop).

## Traffic Flow

ILB setup that routes to the NAT VMs and out to the internet.

    Test VM without Public IP
               |
              ILB
               |
            NAT VM
               |
            Internet


## Scripts

The `scripts` folder contains bash scripts to quickly create the setup. You can run the scripts in the scripts/create folder to create resources:

    cd scripts
    create/1-network.sh
    create/2-firewall.sh
    create/3-instance-group.sh
    create/4-load-balancer.sh
    create/5-routes.sh
    create/6-test-vm.sh

You can also run the all.sh script to create everything:

    cd scripts
    create/all.sh

## ILB as Next Hop

Create `0.0.0.0/0` route to one of the Internal Load Balancer.

The [create/5-routes.sh](create/5-routes.sh) script does

    gcloud compute routes create demo-ilb-to-nat \
      --network=demo \
      --priority=900 \
      --destination-range=0.0.0.0/0 \
      --next-hop-ilb=demo-load-balancer \
      --next-hop-ilb-region=us-central1

So `demo-ilb-to-nat` route won't interfere with the nat egress to IGW

    gcloud compute routes create demo-nat-to-igw \
      --network=demo \
      --priority=800 \
      --tags=demo-nat \
      --destination-range=0.0.0.0/0 \
      --next-hop-gateway=default-internet-gateway

The routing shows all the instances so it's routing through the ILB.

    tung_boltops_com@demo-vm-private:~$ traceroute google.com
    traceroute to google.com (108.177.112.101), 64 hops max
      1   10.0.0.10  2.041ms  0.239ms  0.168ms
      2   10.0.0.9  2.562ms  0.355ms  0.371ms
      3   10.0.0.8  4.530ms  0.728ms  0.681ms
      4   *  *  *
      5   * ^C

Another good test is to curl the ILB from within demo-vm-private:

    tung_boltops_com@demo-vm-private:~$ curl -s 10.0.0.11  | grep h1
    <h1>Welcome to nginx!</h1>
    tung_boltops_com@demo-vm-private:~$

## Summary

Tested with routing to the ILB then to the NAT. 

## Cleanup

You can use the delete scripts to clean up the resources:

    cd scripts
    delete/1-test-vm.sh
    delete/2-routes.sh
    delete/3-load-balancer.sh
    delete/4-instance-group.sh
    delete/5-firewall.sh
    delete/6-network.sh

You can also run the all.sh script to delete everything:

    cd scripts
    delete/all.sh

NOTE: The gcloud compute health-checks command prints an error but it deletes successfully.

## Key Debugging Tips

* On "VM Instances" console Look at routes. It helps to sort by priority and focus on the 0.0.0.0/0 dest.
* Look at demo-vm-private. Confirm routes will lead it through the ILB. This can be confirmed als by running traceroute on the demo-vm-private. The first hop should be the ILB IP: `gcloud compute forwarding-rules list`.
* More importantly, also, Confirm routes on the *nat* instances. That should allow the NAT traffic to go out to the internet.

## Direct to NAT Debbuging

For debugging, routing directly to NAT VM first can be helpful.

    Test VM without Public IP
               |
            NAT VM
               |
            Internet

We'll route directly to the NAT instances.

Create 0.0.0.0/0 route to one of the NAT instances. Note: replace `--next-hop-instance=demo-ig-drdz` with one of the instance you have.

List Managed Instances

    $ gcloud compute instance-groups managed list-instances demo-ig --region=us-central1
    NAME          ZONE           STATUS   HEALTH_STATE  ACTION  INSTANCE_TEMPLATE  VERSION_NAME  LAST_ERROR
    demo-ig-drdz  us-central1-b  RUNNING                NONE    demo-template
    demo-ig-jsfx  us-central1-c  RUNNING                NONE    demo-template
    $

Create the route:

    gcloud compute routes create demo-nat-instance \
      --network=demo \
      --priority=900 \
      --tags=demo-vm-private \
      --destination-range=0.0.0.0/0 \
      --next-hop-instance=demo-ig-drdz \
      --next-hop-instance-zone=us-central1-b

SSH into the private instance, note we'll using [IAP tunneling](https://cloud.google.com/iap/docs/using-tcp-forwarding) since the instance has no public IP.

    $ gcloud compute ssh demo-vm-private
    External IP address was not found; defaulting to using IAP tunneling.
    Last login: Sat Jul 18 03:26:34 2020 from 35.235.241.18
    $

You can ping the internet since `0.0.0.0/0` is routing out to the internet using a NAT instance. Note the `tung_boltops_com@demo-vm-private:~$` prompt indicates commands from within the demo-vm-private instance.

    tung_boltops_com@demo-vm-private:~$ ping google.com
    PING google.com (64.233.191.102) 56(84) bytes of data.
    64 bytes from ja-in-f102.1e100.net (64.233.191.102): icmp_seq=1 ttl=114 time=2.49 ms
    64 bytes from ja-in-f102.1e100.net (64.233.191.102): icmp_seq=2 ttl=114 time=1.40 ms
    ^C
    --- google.com ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3004ms
    rtt min/avg/max/mdev = 1.400/1.779/2.499/0.448 ms
    tung_boltops_com@demo-vm-private:~$ traceroute google.com
    traceroute to google.com (64.233.191.102), 64 hops max
      1   10.0.0.10  3.479ms  0.269ms  0.285ms
      2   *  *  *
    ^C
    tung_boltops_com@demo-vm-private:~$

Delete NAT `0.0.0.0/0` route.

    $ gcloud compute routes delete demo-nat-instance -q

