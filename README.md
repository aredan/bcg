# bcg


The automatic BIRD configuration generator with bogon, IRR, RPKI, and max prefix filtering support.

### Installation

bcg depends on [bird2](https://gitlab.nic.cz/labs/bird/), [GoRTR](https://github.com/cloudflare/gortr),
and [bgpq4](https://github.com/bgp/bgpq4). Make sure the `bird` and `gortr` daemons are running and `bgpq4` is in path
before running bcg. Releases can be
downloaded from Github and from my public code repositories - see https://github.com/natesales/repo for more info. You can
also build from source by cloning the repo and running `go build`. It's recommended to run bcg every 12 hours to update
IRR prefix lists and PeeringDB prefix limits. Adding `0 */12 * * * /usr/bin/bcg` to your crontab will update the filters
at 12 AM and PM. If you're using ZSH you might also be interested in
my [birdc completion](https://github.com/natesales/zsh-bird-completions).

### Why the fork ?

The original project from [Natan Sales](https://github.com/natesales/) was missing a few options that I actually use in my BIRD routers, thats why I decide to fork and added those features.

- Peer type for `core routers` for better iBGP routing distribution (peerType = core).
- I'm using OSPF for loopback distribution.
- VRF Support, at least at Kernel Level since BIRD cant interact with routing table from birdc (`coming soon`).

#### Configuration

BCG can be configured in YAML, TOML, or JSON. All config file formats have the same configuration options but follow a
different capitalization structure. YAML and JSON use all lowercase parameter names and TOML uses CapsCase with acronyms
capitalized. For example, `router-id` in YAML and JSON is `Router-ID` in TOML.

An example to configure a peer with bogon, IRR, RPKI, and max prefix filtering.

```yaml
asn: 65530
router-id: 192.0.2.1
prefixes:
  - 192.0.2.0/24
  - 2001:db8::/48

peers:
  Cloudflare:
    asn: 13335
    type: peer
    neighbors:
      - 203.0.113.39
      - 2001:db8:6939::39
```

`bcg` can take the following flags:

```
Usage for bcg https://github.com/natesales/bcg:
  -config string
        configuration file in YAML, TOML, or JSON format (default "/etc/bcg/config.yml")
  -debug
        show debugging messages
  -dryRun
        skip modifying BIRD config
  -noGenerateUi
        disable generating web UI
  -noConfigure
        disable configuring bird at end of run
  -output string
        directory to write output files to (default "/etc/bird/")
  -socket string
        BIRD control socket (default "/run/bird/bird.ctl")
  -uiFile string
        file to store web UI (default "/tmp/bcg-ui.html")
```

#### How does filtering work?

bcg applies a universal pre-filter to all BGP sessions before evaluating IRR or manual prefix lists which rejects the
following:

- Own prefixes as defined in the global `prefixes` list
- Have a [bogon ASN](https://github.com/natesales/bcg/blob/main/templates/global.tmpl#L176) anywhere in the AS_PATH
- Have a total AS_PATH length of more than 100
- RPKI invalid
- IPv4 prefixes that are...
    - length > 24
    - length < 8
    - contained in the [bogons list](https://github.com/natesales/bcg/blob/main/templates/global.tmpl#L126)
- IPv6 prefixes that are...
    - length > 48
    - length < 12
    - contained in the [bogons list](https://github.com/natesales/bcg/blob/main/templates/global.tmpl#L143)

All peers with a type of `peer` will apply further strict filtering by IRR using their AS-Set defined in PeeringDB.
Max-prefix limits are also enforced for every peer.

#### Local Preference

All sessions have a default BGP LOCAL_PREF of 100, except for routes tagged with
community `65535, 0` ([RFC8326 Graceful Shutdown](https://tools.ietf.org/html/rfc8326)). LOCAL_PREF can be adjusted on a
per-peer basis with the `local-pref` option under the peer block.

#### Pre-import and Pre-export conditions

There are many features of BIRD that aren't part of bcg. If you want to add a statement before importing or exporting of
routes, you can supply a multiline string in `pre-import` or `pre-export` in the peer block to include that snippet of
BIRD code after the import prefilter or before the export filter respectively.

#### iBGP

Next hop self will be enabled on BGP sessions where the neighbor ASN and local ASN are the same (iBGP sessions).

#### Manual configuration

If bcg doesn't have a feature you need (and you can't use pre-import/pre-export conditions) then you can supply pure
BIRD config in `/etc/bird/manual*.conf` and bcg will load the config before loading the rest of the config.

#### Single-stack support

bcg *should* have reasonable single-stack peering support but is not fully tested. Peers that don't have any route{,6}
objects will not have sessions of that address family configured.

#### BGP Communities

bcg uses RFC 8092 BGP Large Communities

#### "Peerlock Lite"

Peers with type `peer` or `downstream` reject any route with a Tier 1 ASN in
path ([Peerlock Lite](https://github.com/job/peerlock)).

#### Communities

| Large     | Meaning                   |
|-----------|---------------------------|
| ASN,0,10  | Learned from CORE routers via iBGP |
| ASN,0,100 | Originated                |
| ASN,0,101 | Learned from upstream     |
| ASN,0,102 | Learned from peer         |
| ASN,0,103 | Learned from downstream   |
| ASN,1,200 | Don't export to upstreams |
| ASN,1,301 | Prepend once              |
| ASN,1,302 | Prepend twice             |
| ASN,1,303 | Prepend three times       |
| ASN,1,666 | Blackhole (must be enabled with `allow-blackholes` |

#### Global Configuration Options

| Option    | Usage                                                                                                         |
| --------- | ------------------------------------------------------------------------------------------------------------- |
| asn       | ASN of this router                                                                                            |
| router-id | Router ID of this router                                                                                      |
| prefixes  | List of prefixes to originate                                                                                 |
| irrdb     | IRRDB to query prefix sets from (default is rr.ntt.net which includes generated route objects from RPKI ROAs) |
| rtr-server | IP address or hostname of RPKI RTR server (default is 127.0.0.1)                                             |
| keep-filtered | Should BIRD keep filtered routes                                                                          |
| peers         | Map of name to peer (see below)                                                                           |
| merge-paths   | Enable merge paths on kernel export                                                                       |
| pref-src4     | Preferred source IPv4 to export to kernel                                                                 |
| pref-src6     | Preferred source IPv6 to export to kernel                                                                 |
| filter-default | Should default routes be denied?                                                                         |
| enable-default | Add static default routes                                                                                |

#### Peer Configuration Options

| Option         | Usage                                                                                                     |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| asn            | Neighbor ASN                                                                                              |
| type           | Type of peer (core, upstream, peer, downstream, import-valid)                                                   |
| local-pref     | BGP LOCAL_PREF                                                                                            |
| disabled       | Should neighbor sessions be disabled?                                                                     |
| passive        | Should neighbor sessions listen passively for BGP TCP connections?                                        |
| multihop       | Should neighbor sessions allow multihop?                                                                  |
| password       | BGP MD5 Password                                                                                          |
| port           | BGP Port (default 179)                                                                                    |
| neighbors      | List of neighbor IP addresses                                                                             |
| pre-import     | List of BIRD expressions to execute after the prefilter and before the prefix filter                      |
| pre-export     | List of BIRD expressions to execute before the export filter                                              |
| prepends       | Number of times to prepend local AS to                                                                    |
| import-limit4  | Maximum number of IPv4 prefixes to allow before disabling the session                                     |
| import-limit6  | Maximum number of IPv6 prefixes to allow before disabling the session                                     |
| skip-filter    | Disable the universal bogon filter (Dangerous!)                                                           |
| rs-client      | Enable route server client                                                                                |
| rr-client      | Enable route reflector client                                                                             |
| bfd            | Enable BFD                                                                                                |
| session-global | String to add to session global config                                                                    |
| enforce-first-as | Reject routes that don't have the peer ASN as the first ASN in path                                     |
| enforce-peer-nexthop | Reject routes where the next hop doesn't match the neighbor address                                     |
| export-default | Should a default route be sent over the session? (default false)                                            |
| no-specifics | Don't send specific routes (default false, make sure to enable export-default or else no routes will be exported) |
| allow-blackholes | Accept community (ASN,1,666) to blackhole /32 and /128 prefixes |
| strip-private-asns | Should private ASNs be stripped from path before exporting? (in range `[64512..65534, 4200000000..4294967294]`) |
| communities | List of BGP communities to add on export (two comma-separated values per list element; example `0,0`) |
| large-communities | List of BGP large communities to add on export (three comma-separated values per list element; example `0,0,0`) |
