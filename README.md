# beldex-multi-mn

The deb produced here gives you systemd service to help you run multiple master nodes on
one machine.

## Warning - Risk

Doing this is increasing your risk: if the machine goes down *all* the master nodes on it go down,
while if you use multiple VPSes the loss of one VPS affects only that one master node and not the
others.

### Setting up one master node

Choose a two-digit number from 00 to 99.  I recommend you start at 00 or 01 and then move up by one
for each one, but if you have some other system in mind go ahead.  It must be numeric, however, and
I highly recommend that you start values in 0-9 with an extra leading 0.

The number you choose will be used in the ports that your master nodes use: for service number with
number XX:

Public address listeners:
- storage server will listen on port 299XX (on the public IP)
- beldexd p2p will listen on port 197XX (on the public IP)
- beldexd quorumnet will listen on port 199XX (on the public IP)
- belnet will listen on UDP port 109XX (on the public IP)

Internal (localhost) listeners:
- beldexd rpc will listen on port 198XX
- belnet rpc will listen on port 119XX
- the belnet router will add and use local network 10.1XX.0.0/16  on a virtual interface named
  beldextunXX, on which the local snode is available at 10.1XX.0.1 (you probably don't need to worry
  about this).

As an example, let's say I have chosen the number 45.  Then my public IP will have services on ports
29945, 19745, 19945 and 10945; with internal (localhost) ports 19845 and 11945.  (If you are using
a firewall that blocks everything, add appropriate exceptions for *each* MN's four public ports).

### Enabling the service

Enable and start your master node cluster using the number you chose above (I'll continue using 45
as an example):

    sudo beldex-multi-mn-create 45

This will set up basic configurations for the master node components, and enable and start these
services:

    beldex-node@45.service
    beldex-storage-server@45.service
    belnet-router@45.service

If you want to control just one service, you use these templated names to manage the services.  For
example, to stop just this beldexd:

    systemctl stop beldex-node@45.service

or to view beldex-storage-server log output:

    journalctl -u beldex-storage-server@45 -af

The beldexd data will be inside /var/lib/beldex/node-45, the storage server data will be inside
/var/lib/beldex/storage-45, and the belnet data will be inside /var/lib/belnet/router-45.
Configuration for each will be in /etc/beldex/node-45.conf, /etc/beldex/storage-45.conf, and
/etc/beldex/belnet-router-45.ini.

You will most likely want one additional piece to be able to query the master node: inside your
~/.bashrc add the following:

    for n in /etc/beldex/node-*.conf; do
        p=${n/*node-/}
        p=${p/.conf/}
        alias beldexd-$p="beldexd --config=$n"
    done
    beldexd_all() {
        for n in /etc/beldex/beldex/node-*.conf; do
            p=${n/*node-/}
            p=${p/.conf/}
            echo -e "\nbeldexd-$p:"
            beldexd --config=$n "$@"
        done
    }

When you log out and log in again you will now have a `beldexd-45` alias that invokes commands on your
node 45, such as checking the status:

    $ beldexd-45 status
    2022-06-22 10:20:14.246	I Beldex 'Bucephalus' (v4.0.2-f47be2250)
    Height: 1271971, v4.0.2-53ff2bd75(net v17), 8(out)+7(in) connections, uptime 0d2h12m21s
    MN: 436c598b20e4811efe9ef6aa55af0f3b6541beaf3c71b09f67ebc08c411dbea6 active, proof: 1.8 minutes ago, last pings: 14sec (storage), 1sec (belnet)

and you will have a `beldexd_all` command that runs a command on *all* the beldexds, such as:

    $ beldexd_all status

    beldexd-45:
    2022-06-22 09:44:20.813	I Beldex 'Bucephalus' (v4.0.2-f47be2250)
    Height: 475765/1271912 (37.4%), syncing, net hash 39.13 kH/s, v4.0.0-0ffabefca(net v12), 8(out)+10(in) connections, uptime 0d0h2m48s
    MN: 9280e29ea2acaa02a6db69a21cf7a5a4bc4322cdb9ca390da448247018a00c30 not registered, last pings: 16sec (storage), NOT RECEIVED (belnet)

    beldexd-65:
    2022-06-22 11:33:00.848	I Beldex 'Bucephalus' (v4.0.2-1b6b13f3e)
    Height: 1272085, v4.0.2-1b6b13f3e(net v17), 7(out)+30(in) connections, uptime 9d1h21m56s
    MN: 434790acce5989b6b17e24ee943e244586dba6484b47300786a6310c784f90dc active, proof: 52.8 minutes ago, last pings: 1sec (storage), 18sec (belnet)

### Upgrades

You need to be a bit more careful when updating.  When you upgrade one or more of the debs, only the
basic services (`beldex-node.service`, `beldex-storage-server.service`, and/or `belnet-router.service`)
will be restart but *not* the templated versions, so you will need to do this manually.  One way is
to specify all the numbers, such as:

    sudo systemctl restart beldex-node@01 beldex-node@02 beldex-node@03

which you can also shorten to:

    sudo systemctl restart beldex-node@{01,02,03}

The templates services, however, also get a "target" which you can use:

    sudo systemctl restart beldex-nodes.target

Similarly there are targets for storage server and routers: `beldex-storage-servers.target` and
`belnet-routers.target`.

Note, however, that targets only apply to currently running services, so if you have stopped some
you *cannot* use `sudo systemctl start beldex-nodes.target` to start them all.