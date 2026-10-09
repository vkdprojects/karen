# KAREN network design: what's current in Oct 2026

Researched 2026-10-08. [unverified] = not confirmed by a primary source.

## 0. Versions in use (2026-10)
| Component | Version | Source |
|---|---|---|
| Open vSwitch | **4.0.0** (2026-08-17). LTS is **3.7.1**, 3.6.3 is the last 3.6 release. AF_XDP is still marked *experimental*. Userspace TSO is no longer experimental | [download](https://www.openvswitch.org/download/), [NEWS-4.0](https://www.openvswitch.org/releases/NEWS-4.0.0.txt), [AF_XDP doc](https://docs.openvswitch.org/en/latest/intro/install/afxdp) |
| OVN | **26.09** (2026-09-18). It adds EVPN ECMP/multihoming and EVPN ARP/ND suppression. Native BGP integration has existed since 25.03 | [26.09](https://www.ovn.org/en/releases/26.09), [OVSCon25](http://www.openvswitch.org/support/ovscon2025/index.html) |
| FRR | **10.7.1** (2026-08-25) | [FRR](https://frrouting.org/release/) |
| NetBox | **4.7** (Sep 2026) | [notes](https://netboxlabs.com/docs/netbox/release-notes/) |
| Routinator (Rust RPKI validator) | **0.15.2** (Jun 2026) | [NLnet](https://www.nlnetlabs.nl/news/2026/Jun/08/routinator-0.15.2-released) |
| TCP BBRv3 | **Not in mainline Linux.** Mainline `tcp_bbr.c` is still v1. Google is "preparing a patch series for upstream" (IETF 126, Jul 2026) | [tcp_bbr.c](https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_bbr.c), [IETF126](https://www.ietf.org/proceedings/126/slides/slides-126-ccwg-bbrv3-00.pdf) |

## 1. Host dataplane

### Measured numbers
OVS paper, SIGCOMM'21, ConnectX-6 Dx at 25G ([pdf](https://benpfaff.org/papers/ovs+10.pdf)):

| Metric | Kernel OVS | OVS-DPDK | OVS AF_XDP |
|---|---|---|---|
| VM↔remote host TCP_RR latency, P50/90/99 | 58/68/94 µs | **36/38/45 µs** | ≈ DPDK |
| Container↔container, same host | 15/16/20 µs | 81/136/241 µs | 15/16/20 µs |
| Packet rate into the VM | — | — | tap: **1.3 Mpps**, vhost-user: **6.0 Mpps** |
| P2P forwarding | needs about 8 cores to match DPDK | best Mpps per core | below DPDK |

Other data points:
- **XDP, single core:** drop-only 14 Mpps, parse+drop 8.1, map lookup 7.1, rewrite+forward 4.7 Mpps (same paper).
- **Connection tracking is the main per-packet cost.** OVS-DPDK 3.0 on 1 core: about 4.2–4.7 Mpps without conntrack vs 1.4–1.8 Mpps with it, roughly a 2.5–3× penalty. With 8 cores and 8 queues it reaches 7.2 Mpps ([Red Hat](https://developers.redhat.com/articles/2022/11/17/benchmarking-improved-conntrack-performance-ovs-300)). So: use stateless ACLs where you can, and stateful ones only on ports that opted into a firewall.
- **Hardware offload ceiling:** Azure AccelNet (SR-IOV + FPGA) gets about 10 µs average VM↔VM latency ([NSDI'18](https://www.usenix.org/system/files/conference/nsdi18/nsdi18-firestone.pdf)).

### Options compared
| Option | Strengths | Costs | Who runs it |
|---|---|---|---|
| **Linux routed tap + nftables/tc** | Simplest. No broadcast domain. Static `/32` and `/128` per tap | No VPC, ACL or DHCP model; you build them | Hetzner routed setups ([Hetzner](https://community.hetzner.com/tutorials/install-and-configure-proxmox_ve)) |
| **OVS kernel datapath + OVN** | Logical switches/routers, `port_security` (MAC+IP anti-spoof), stateful ACLs, native DHCP/RA, QoS (`qos_max_rate`), Geneve. BGP via FRR: `connected-as-host` + `redistribute-local-only` announces a `/32` only from the host running the VM ([OVN docs](https://docs.ovn.org/en/latest/topics/dynamic-routing/architecture.html)). Multichassis RARP activation for live migration ([Neutron](https://docs.openstack.org/neutron/latest/contributor/internals/ovn/live_migration.html)) | ovsdb/northd to operate. Geneve adds 58 B, so MTU 1442 on a 1500 underlay ([RH](https://access.redhat.com/solutions/7059376)) | OpenStack, OpenShift. DigitalOcean (OVS + BGP, L3) ([InfoQ](https://www.infoq.com/news/2020/06/scaling-networking-digitalocean/)) |
| **OVS-DPDK + vhost-user** | Lowest latency (36 µs P50) | Dedicated PMD cores you can't sell, hugepages, harder operations | NFV/telco |
| **OVS AF_XDP** | Close to DPDK, no PMD lock-in | Still experimental | — |
| **TC flower hardware offload** (ConnectX-5+ eSwitch; conntrack offload on CX-6 Dx+) | Line rate, frees the CPU | Off by default; NIC-specific ([OVS](https://docs.openvswitch.org/en/latest/howto/tc-offload)) | Red Hat OSP ([RH](https://docs.redhat.com/en/documentation/red_hat_openstack_platform/17.1/html/configuring_network_functions_virtualization/config-ovs-hwol_rhosp-nfv)) |
| **eBPF/XDP** (Cilium netkit, Katran, Cloudflare L4Drop/Unimog) | Fastest filtering. L4Drop dropped >8 Mpps on one server with only about +10% CPU ([CF](https://blog.cloudflare.com/l4drop-xdp-ebpf-based-ddos-mitigations)) | You write the whole dataplane yourself | Meta, Cloudflare |
| **SR-IOV VF** | Near bare-metal performance | Bypasses host ACLs unless switchdev; live migration only with mlx5 VFIO migration ([NVIDIA](https://docs.nvidia.com/networking/display/mlnxofedv24040700/sr-iov+live+migration)) | Azure AccelNet |
| **DPU** | Isolates bare metal; OVN running on the DPU (OVSCon25 talk) | $$, vendor SDK (DOCA) | OCI on BlueField-3 ([NVIDIA](https://nvidianews.nvidia.com/news/oracle-cloud-infrastructure-chooses-nvidia-bluefield-data-center-acceleration-platform)), CoreWeave on BlueField ([NVIDIA](https://www.nvidia.com/en-us/case-studies/core-weave-revolutionizes-data-centers-with-blufield-dpus)), GCP C3 on Intel IPU E2000 at 200G ([Intel](https://medium.com/intel-tech/intel-ipu-e2000-a-collaborative-achievement-with-google-cloud-eb1dda8c0177)). AMD Pensando Salina claims ~1.45× BlueField-3 (vendor claim) ([AMD](https://www.amd.com/en/products/data-processing-units/pensando.html)) |

**vhost-net vs vhost-user:** vhost-net is the kernel path and needs no PMD. vhost-user requires a DPDK or AF_XDP userspace switch, and it is the reason the AF_XDP path rises from 1.3 to 6.0 Mpps.

## 2. Fabric
- **Leaf-spine, L3 all the way down to the host.** No MLAG and no stretched VLANs.
- **eBGP unnumbered** (RFC 5549: IPv4 routes carried with IPv6 link-local next hops) between leaf↔spine and also leaf↔host, using FRR ([Cumulus](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-514/Layer-3/Border-Gateway-Protocol-BGP/Basic-BGP-Configuration)). This gives an IPv6-only underlay with IPv4 via v6 next hops ([draft](https://www.ietf.org/archive/id/draft-ietf-intarea-v4-via-v6-06.html)).
- **How VMs get public IPs without L2:** each `/32` and `/128` is routed to the host, and the host proxies ARP/ND to the guest.
  - Hetzner binds IP to MAC and routes `/32`s onto the host bridge.
  - DigitalOcean moved from L2 to L3 Clos with GoBGP embedded in a hypervisor agent, because ARP broadcast did not scale.
- **IP mobility for live migration:** the destination host starts announcing the `/32`, OVN `local-only` withdraws it from the source, and BFD gives sub-second convergence [inference].
- **EVPN-VXLAN:** only when a customer needs L2 (bare-metal private VLANs). OVN 26.09 does this natively (it was experimental in 25.09).
- **SRv6 uSID:** in production at Alibaba (SONiC) and LINE ([IETF119](https://datatracker.ietf.org/meeting/119/materials/slides-119-srv6ops-alibaba-cloud-02), [Cisco](https://blogs.cisco.com/sp/srv6-is-coming)). It is hyperscaler-grade and not worth it at KAREN's scale.

## 3. Edge (Brazil-first)
- **IX.br:**
  - 50 Tbit/s aggregate; São Paulo alone is **32 Tbit/s** with **>2,500 networks**; about 3,800 ASes nationally (Mar 2026) ([NIC.br](https://nic.br/noticia/releases/ix-br-hits-record-50-tbit-s-of-aggregated-internet-traffic-driven-by-content-and-digital-services)).
  - Peer via the route servers plus bilateral sessions with the big content networks.
  - **Use two separate IX.br SP connections, each in a different facility.** The Mar 2025 fire at Equinix SP4 cut dark fibre and caused IX.br SP outages ([DCD](https://www.datacenterdynamics.com/en/news/equinixs-sp4-data-center-in-s%C3%A3o-paulo-suffers-fire-over-weekend/)).
- **Transit:** at least 2 providers.
- **RPKI:** Routinator → RTR → FRR/BIRD, drop invalids, publish ROAs for your own prefixes.
- **DDoS:**
  - Detection with FastNetMon Community (GPLv2, runs as a separate process). Detection time is 2 s with port mirror, 4 s with sFlow, 10–30 s with NetFlow ([NANOG96](https://storage.googleapis.com/site-media-prod/meetings/NANOG96/5665/20260204_Odintsov_Fastnetmon_Community_Open_v1.pdf)).
  - It triggers ExaBGP/GoBGP to send RTBH upstream, Flowspec ([RFC 8955](https://www.rfc-editor.org/rfc/rfc8955.html)) to your own edge, or diversion to a scrubber.
  - Most transits don't accept customer Flowspec ([RJS](https://www.rjscloudacademy.com/2026/01/why-service-providers-dont-accept.html)).
  - Local on-demand scrubbing options: Huge Networks, UPX, Cloudflare Magic Transit ([Huge](https://www.huge-networks.com/pt/protecao-ddos), [UPX](https://www.upx.com/pt/protecao-de-borda/ddos-defense), [CF](https://www.cloudflare.com/pt-br/magic-transit/service-providers/)).
  - Add an XDP pre-filter on the edge and on hosts.
- **Per-VM rate limits:** OVN `qos_max_rate`/burst and meters in kbps or pktps ([ovn-nbctl](https://www.ovn.org/support/dist-docs-branch-21.03/ovn-nbctl.8.pdf)). Limit outbound PPS so abusive VMs can't take down the host.
- **Anycast:** use for DNS and status pages first.

## 4. Latency
| Path | RTT | Source |
|---|---|---|
| SP↔Brasília | 15.5 ms | [WonderNetwork](https://wondernetwork.com/pings/Sao%20Paulo) |
| João Pessoa↔Brasília (Northeast) | 43.9 ms | [WonderNetwork](https://wondernetwork.com/pings/Joao%20Pessoa) |
| SP↔Miami | about 100 ms | [Melbicom](https://www.melbicom.net/blog/dedicated/faster-apps-lgpd-compliance-brazil) (vendor) |
| NJ↔B3 (Seabras-1, finance tier) | 105.05 ms | [SubTel](https://www.submarinenetworks.com/en/systems/brazil-us/seabras-1/seaborn-networks-delivers-ultra-low-latency-route-from-new-jersey-to-sao-paulo) |
| SP↔NY physical floor | 77 ms | [regionlatency](https://regionlatency.com/azure/brazilsouth) |
| SP↔Ashburn | ≈110–120 ms | [unverified] |

What this means: host in SP. The Northeast (Fortaleza is IX.br's #2 exchange) is a later POP, not an edge you need on day one.

**Tuning:**
- Use `fq` + BBR(v1) from mainline. Avoid out-of-tree BBRv3 kernels.
- Multiqueue virtio-net with queues = vCPUs, plus vhost-net.
- Pin IRQs to NUMA-local cores.
- Underlay MTU 9216; guests 1500 (or 8942 inside a VPC).

**NICs:** ConnectX-6 Dx/7 (switchdev, conntrack offload, kTLS) at **2×25G** for VPS hosts and **2×100G** for dense hosts. Intel E810 is fine for kernel/XDP, but its OVS offload path is less proven [inference].

## 5. IPAM and DNS
- **NetBox 4.7 is the source of truth for infrastructure:** sites, racks, cables, prefixes, ASNs, VNI pools.
- **KAREN's DB owns per-VM leases** (high churn) inside NetBox-defined prefixes, and writes them back to NetBox asynchronously.
- **Addressing:** one `/32` plus one routed **/64 per VM** (Hetzner gives a `/64` per server), with an optional `/56` delegated prefix.
- **rDNS:** PowerDNS API managing the delegated `in-addr.arpa` and `ip6.arpa` zones.
- **ASN/IP space:** from NIC.br/Registro.br (the NIR under LACNIC).

## Recommendation for KAREN

**Pick: L3 routed to the host plus OVN.** Each host runs OVS 3.7 LTS (kernel datapath), OVN 26.09 and FRR 10.7. Hosts BGP-peer unnumbered to their leaves and announce only the VM `/32` and `/128` routes they host (`connected-as-host` + `local-only`).

| Layer | Choice |
|---|---|
| VM NIC | virtio-net multiqueue + vhost-net |
| Anti-spoof | OVN `port_security`; nftables only in routed-tap mode |
| Firewall | Stateless by default; stateful only when the customer enables it (conntrack costs 2.5–3×) |
| Private network | OVN logical switch over Geneve |
| Bare metal | Routed leaf port with switch ACLs → later BlueField-3 running OVN |
| Edge | 2 edge routers, ≥2 transits, 2× IX.br SP, Routinator, FastNetMon (sFlow) → RTBH/Flowspec/scrubber |
| Control-plane code | Rust agent driving `rtnetlink` and the OVN NB (OVSDB JSON-RPC). Keep FRR as an external process; the Rust Holo stack is not mature enough yet [inference] |

### Phased path
| Phase | Topology | Network |
|---|---|---|
| **P0 – single node** (roadmap v0.1–0.2) | One host behind the upstream provider's router | **Routed tap, not a Linux bridge.** Static `/32`/`/128` per tap, proxy ARP/NDP, nftables `netdev` ingress anti-spoof, tc police. Works behind Hetzner/OVH-style upstreams |
| **P1 – one rack** (v0.4) | 2 leaves (SONiC/Cumulus, 25/100G); dual-homed hosts with BGP unnumbered, no LACP | OVN all-in-one → 3-node RAFT northbound/southbound. Own ASN, transits, IX.br, RPKI, FastNetMon |
| **P2 – multi-rack** | 2–4 spines, 3-stage Clos, one ASN per leaf pair | OVN live migration (v0.5), EVPN for bare-metal L2, NetBox + PowerDNS |
| **P3 – scale / regions** | CX-7 TC offload or BlueField-3 on dense and bare-metal racks; US/EU POPs | OVN-IC across availability zones, anycast DNS |

### Avoid
- Stretched L2, VLAN-per-customer, MLAG.
- OVS-DPDK in the default plan (wastes sellable cores).
- AF_XDP in production (still experimental).
- SR-IOV for general VPS.
- SRv6.
- Out-of-tree BBRv3 kernels.
- Relying on transits accepting your Flowspec.
- Concentrating everything in one SP facility.

### Suggested roadmap changes (main agent)
1. **v0.1** "Linux bridge networking" → "routed tap (`/32`+`/128`)".
2. **v0.4** "VLAN, routed, NAT and OVS modes" → collapse to **two modes**: routed-tap and OVN. OVN `localnet` covers VLAN, and OVN NAT covers NAT. Four modes would mean four anti-spoof and rate-limit code paths to maintain.
3. **v0.5** live migration depends on the OVN multichassis binding and the BGP `/32` move.
