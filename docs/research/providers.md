# Inside real hosting and cloud providers (as of 2026-10-08)

Researched 2026-10-08. [unverified] = not confirmed by a primary source.

## 1. Global providers

| Provider | Hypervisor / VMM | Control plane | Network fabric | Storage | Bare-metal provisioning | DDoS |
|---|---|---|---|---|---|---|
| **DigitalOcean** | KVM/QEMU [unverified: widely reported, not stated in DO docs] | In-house services in Go | L3 Clos fabric with label switching and BGP. **GoBGP is embedded in a Go hypervisor daemon** that tracks Droplet events. OVS is on every hypervisor. It moved from L2 because of ARP flooding, and VXLAN was rejected because the NICs had no offload ([InfoQ](https://www.infoq.com/news/2020/06/scaling-networking-digitalocean)) | **Ceph** for Volumes and Spaces: 65 clusters, 250+ PB raw, 30k+ OSDs on 1,700+ nodes. Ceph runs in containers, deployed with Ansible/AWX, with a "no ssh" operating rule ([Ceph Days 2025](https://ceph.io/assets/pdfs/events/2025/ceph-days-silicon-valley/03%20-%20Alex%20-%20Ceph%20operations%20-%20Ceph%20days%20SV%20_25.pdf), [why Ceph](https://www.digitalocean.com/blog/why-we-chose-ceph-to-build-block-storage)) | n/a | In-house [unverified] |
| **Vultr** | KVM ([VPS docs](https://docs.vultr.com/products/compute/instances/cloud-compute/features/ddos-protection)) [unverified detail] | Proprietary | Proprietary | Local NVMe plus block storage [unverified backend] | Bare-metal product exists ([site](http://vultr.com/products/bare-metal)) | Own "Attack Mitigation Farm". Traffic is rerouted there within about 60 s, filtered for L3/L4, and clean traffic is tunnelled back to the server. 10 Gbps per instance ([docs](https://docs.vultr.com/ddos-protection.md)) |
| **Linode / Akamai** | UML (2003) → Xen (2008) → **KVM (2015)** ([Linode](https://www.linode.com/community/questions/10196/linode-kvm-beta), [CIO](https://www.cio.com/article/244279/why-linode-moved-to-kvm.html)) | In-house; Akamai acquired Linode in 2022 ([PR](https://www.akamai.com/newsroom/press-release/akamai-completes-acquisition-of-linode)) | Akamai backbone | Block Storage Volumes ([docs](https://techdocs.akamai.com/cloud-computing/docs/block-storage)); Ceph backend [unverified] | — | Akamai Prolexic is available ([Akamai](https://www.akamai.com/newsroom/press-release/akamai-extends-ddos-defense-with-prolexic-on-prem-and-hybrid-options)) |
| **Hetzner** | KVM [unverified]. **Live migration**: copies NVMe and RAM while tracking dirty pages, then a <1 s blackout ([docs](https://docs.hetzner.com/cloud/servers/technical-concepts/architecture/)) | In-house (Robot for dedicated, Cloud API for cloud) | Juniper backbone ([managed](https://www.hetzner.com/managed-server?country=de)) | Volumes are **triple-replicated** on 3 physical servers ([FAQ](https://docs.hetzner.com/cloud/volumes/faq)) | Robot: rescue system, installimage, KVM-over-IP console on request ([docs](https://docs.hetzner.com/robot/dedicated-server/maintenance/kvm-console)) | **Arbor + Juniper**, free, automatic, filters by attack pattern (for example, it treats a 500 kpps SYN flood differently from a 500 kpps UDP flood) ([Hetzner](https://www.hetzner.com/unternehmen/ddos-schutz)) |
| **OVHcloud** | OpenStack (KVM) for Public Cloud ([OVH](https://www.ovh.co.uk/public-cloud/instances/technologies)) | OpenStack plus in-house manager/API | Own global backbone and vRack L2 private network | Ceph [unverified] | In-house reinstall API | **VAC**, built in-house: HCAP policers at the PoPs, then Edge Network Firewall, then Armor. Uses **FPGA and x86** filtering, NetFlow/sFlow detection, and >50 Tbps of capacity. Mitigated a **4.2 Tbps** attack and a **1.9 Bpps** packet-rate attack ([mitigation](https://us.ovhcloud.com/security/anti-ddos/ddos-attack-mitigation), [2024 recap](https://blog.ovhcloud.com/en/posts/a-brief-retrospective-of-network-layer-ddos-attacks-in-2024-at-ovhcloud)) |
| **Scaleway** | KVM [unverified] | Gateway turns CLI/API/Terraform requests into **internal gRPC**; services in Go | Fabric network shared by VMs and bare metal | — | **They tried Ironic for 3 months and dropped it (needed 12+ dependencies). They evaluated Tinkerbell and then built their own, "BATMAN"**. Elastic Metal has an in-house BMC/IPMI service and blocks SMTP by default, unlocked by a trust score ([blog](https://www.scaleway.com/en/blog/elastic-metal-story/)) | Own anti-DDoS team |
| **Fly.io** | **Firecracker** microVMs (Rust) | `flyd` (Go) orchestrator; containerd converts OCI images to rootfs; Rust `init` inside each VM; Rails/Postgres GraphQL API | `fly-proxy` (Rust, Tokio, Hyper) with **Anycast**; **corrosion** (Rust, SWIM gossip) for state; WireGuard mesh ([stack](https://fly.io/docs/hiring/stack), [corrosion](https://www.fly.io/blog/corrosion)) | `vold`, encrypted local volumes | — | Anycast edge |
| **Equinix Metal** | — | — | — | — | Created **Tinkerbell**. **Sunset 30 June 2026**; docs removed 30 Sept 2026 ([Equinix](https://docs.equinix.com/metal), [DCD](https://www.datacenterdynamics.com/en/news/equinix-officially-retires-bare-metal-offering)) | — |
| **Latitude.sh** | VMs added 2026 ([changelog](https://www.latitude.sh/changelog)) | Proprietary | **BGP**; managed K8s (LKS) runs RKE2 + MetalLB ([blog](https://www.latitude.sh/blog/kubernetes-on-bare-metal-without-the-setup-hell)) | — | Core product; strong in Web3 and Solana | [unverified] |
| **Oracle OCI** | **Off-box virtualization**: network virtualization runs on a SmartNIC that is isolated from the host, so even a compromised hypervisor cannot reach the network ([Oracle](https://www.oracle.com/security/cloud-security/isolated-network-virtualization)). Acceleron SmartNIC is used for the data plane ([Acceleron](https://www.oracle.com/cloud/networking/acceleron/smartnic)) | — | — | — | Bare metal uses the same off-box path ([BM](https://www.oracle.com/cloud/compute/bare-metal)) | — |
| **AWS Nitro** (reference) | Thin KVM-based Nitro Hypervisor plus **Nitro Cards** (VPC, EBS and NVMe offload) and the Nitro Security Chip. The **Isolation Engine is formally verified** ([whitepaper](https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/the-components-of-the-nitro-system.html), [blog](https://aws.amazon.com/blogs/compute/aws-nitro-isolation-engine-formally-verifying-the-hypervisor-in-the-aws-nitro-system)) | — | — | EBS via NVMe over the Nitro card | Bare-metal instances use the same cards | Shield |
| **Hostinger** | KVM VPS ([VPS](https://www.hostinger.com/vps-hosting)) | **In-house hPanel** | Own CDN; Cloudflare-fronted nameservers | **CephFS** for shared hosting since 2016 ([blog](https://www.hostinger.com/blog/hostinger-joined-yahoo-cern-bloomberg-creating-scalable-hosting-using-cephfs)) | — | In-house firewall; BitNinja on VPS. Shared hosting is LiteSpeed + **CloudLinux LVE**, with a Brazil DC ([tech](https://www.hostinger.com/technology)) |

**Firecracker reference numbers:** under 5 MiB memory overhead per VM, ≤125 ms to application code, up to 150 microVMs per second per host ([NSDI paper](https://css.csail.mit.edu/6.5660/2026/readings/firecracker.pdf)).

## 2. Brazilian providers

| Player | What is verifiable |
|---|---|
| **StayCloud** (Varginha/MG) | NVMe, LiteSpeed, cache and CDN by default; "isolamento entre contas"; automatic backups; 3 regions (BR/US/EU); 2,400+ agencies and devs; new **Stay Deploy** for AI-built apps ([about](https://staycloud.com/arquitetura)); VPS tuned for n8n ([help](https://scsr.zendesk.com/hc/pt-br/requests/new)); Cloudflare in front (the 301 is served by Cloudflare: [search](https://staycloud.com.br/)). The cPanel/WHMCS details and the SP + Ashburn backup sites are from the brief, not checked here. |
| **Locaweb** (LWSA3) | Owns a DC in Brazil ([Locaweb](https://www.locaweb.com.br/conteudos/servidor-de-hospedagem)); new "Locaweb Cloud" VMs from R$20/mo; reseller hosting on its **own panel** ([revenda](https://www.locaweb.com.br/revenda-de-hospedagem)); Servers API for dedicated / Cloud Server Pro / VPS ([API](https://developer.locaweb.com.br/docs/api-servidores)). Hypervisor not disclosed [unverified]. |
| **KingHost** | Bought by Locaweb in 2019 (300k sites, 60k clients) ([Telecompaper](https://www.telecompaper.com/news/locaweb-buys-kinghost--1293233)); VPS from R$53 ([site](https://site.king.host/servidor-vps)). Stack not disclosed. |
| **HostGator BR** | Classic **cPanel/WHM** shared hosting, resellers and dedicated servers ([support](https://suporte.hostgator.com.br/hc/pt-br/articles/30808065412371-Quais-as-funcionalidades-do-Gerenciador-de-arquivos-do-cPanel), [status](https://status.hostgator.com.br/)). Newfold ownership [unverified]. |
| **Magalu Cloud** | Regions br-se1 and br-ne1; VMs, upstream K8s, DBaaS, Block, Object, VPC ([docs](https://docs.magalu.cloud/docs)); 1,200 external clients and about 55% of Magalu's own workloads on its infra ([RAD 2025](http://ri.magazineluiza.com.br/Download/MGLU_RAD_2025_ENG?=6t0xQZz1fRWdx5K%2FD%20%2FXqQ%3D%3D&linguagem=en)); R$300M from BNDES ([DPL](https://dplnews.com/magalu-cloud-recebe-r-300-mi-bndes-expandir-data-centers-brasil)). IaaS stack not disclosed (OpenStack is a guess) [unverified]. |
| **Square Cloud** (Goiânia) | Container PaaS for bots, APIs and databases: "ambientes virtuais isolados". **Infra is in the US**: moved to Hivelocity in 2022 and survived Hurricane Idalia in Tampa. Proxies rewritten in Go (2024), logs in Grafana Loki, 500k+ devs, 4B+ requests per month ([about](https://squarecloud.app/pt-br/about)). |
| **Discloud** | Container-based PaaS with Docker/Dockerfile support, AMD EPYC, NVMe, anti-DDoS ([site](https://discloud.com/), [GitHub](https://github.com/discloud)). |

**Brazil colocation and peering**
- **IX.br** hit 50 Tbps aggregate in March 2026. **IX.br SP hit 32 Tbps** with 2,500+ networks. Across Brazil there are about 3,800 ASes in 39 metros ([NIC.br](https://nic.br/noticia/releases/ix-br-hits-record-50-tbit-s-of-aggregated-internet-traffic-driven-by-content-and-digital-services)). Peering is free, so being on IX.br SP is mandatory.
- **Equinix SP4** carried about 10 Tbps during the 2026 World Cup ([Folha](https://www1.folha.uol.com.br/colunas/painelsa/2026/06/partidas-do-brasil-impulsionam-trafego-recorde-em-data-centers-no-pais.shtml)).
- **Ascenty** (Digital Realty + Brookfield): 60 MW SP campus, SPO05 live; US$1.2B for 150 MW of AI capacity ([Ascenty](https://ascenty.com/en/blog/news-ascenty-en/ascenty-sao-paulo-campus), [DCD](https://www.datacenterdynamics.com/en/news/ascenty-announces-12-billion-investment-in-ai-with-a-record-150-mw/)).
- **Scala**: Tamboré hyperscale campus ([DCD](https://www.datacenterdynamics.com/en/news/scala-launches-second-phase-of-data-center-campus-in-sao-paulo-brazil)).
- Small hosts rent racks in Equinix SP2/SP3/SP4, Ascenty or ODATA and peer at IX.br SP [inference].

## StayCloud (observed 2026-10-08)

Fingerprinted from public DNS/HTTP and the StayCloud site this session.

| Aspect | Observed |
|---|---|
| Edge | All hostnames (`staycloud.com`, `beta.`, `movie.`) behind Cloudflare (AS13335, `cf-ray …-GRU`); NS on Cloudflare |
| Mail | Google Workspace |
| Checkout | `cart.staycloud.com` is **WHMCS** |
| Shared hosting | **cPanel/WHM + LiteSpeed**; "each application in its own container" |
| VPS | 1-click apps: n8n, Evolution API, Supabase, Chatwoot, Hermes, OpenClaw |
| Stay Deploy | GitHub / Claude Code → build → deploy to the user's VPS with domain, SSL, CDN |
| Backups | Every 12 h in São Paulo, replicated to Ashburn |
| Regions | São Paulo, Ashburn, Frankfurt; "Tier III, ISO 27001, SOC 2 Type II" datacenters |
| Hypervisor | [not disclosed] |
| Orchestration | [not disclosed] |

**Conclusion:** StayCloud = commodity stack (cPanel + LiteSpeed + WHMCS + Cloudflare) plus product, UX and support. KAREN's `web` + `vps` SKUs must cover all of it without licensed components.

## 3. Web-hosting panels in 2026

| Panel | License (2026) | Web server | Isolation model | Notes |
|---|---|---|---|---|
| **cPanel/WHM** | Solo $29.99 → Premier $69.99/mo, plus **$0.49 per account over 100** ([cPanel](https://support.cpanel.net/hc/en-us/articles/30117774089879-2026-cPanel-Store-License-Pricing)); up to ~20% more in 2027 ([WHT](https://webhosting.today/2026/10/08/plesk-whmcs-and-cpanel-cost-up-to-20-percent-more-in-2027/)) | Apache / LiteSpeed | Unix user; **CloudLinux LVE + CageFS** add-on | WebPros (CVC) also owns Plesk, WHMCS and SolusVM, so the **WHMCS lock-in** keeps hosts on cPanel ([WHT](https://webhosting.today/2026/08/14/the-field-of-cpanel-alternatives-keeps-widening-a-brand-new-one-comes-from-inside-cpanels-ecosystem/)). An exploited cPanel bug was reported in April 2026 ([TechCrunch](https://techcrunch.com/2026/04/30/hackers-are-actively-exploiting-a-bug-in-cpanel-used-by-millions-of-websites/)) |
| **Plesk** | Web Admin $18 → Web Host $50 ([CostBench](https://costbench.com/software/cloud-infrastructure/plesk)) | Apache + nginx | Per-subscription user; CloudLinux optional | Same owner as cPanel |
| **Enhance** | Per-site billing ([FAQ](https://enhance.com/faqs)) | Apache / nginx / LiteSpeed / OLS, switchable | **Each site in its own lightweight container**, and roles (email, DB, DNS, backup) are containerized too. **Cluster of 1 to 10,000 servers**; zero-downtime moves of sites between servers ([features](https://enhance.com/product/features)) | Users complain about container RAM use ([forum](https://community.enhance.com/d/1917-moving-away-from-enhance)) |
| **DirectAdmin** | Flat tiers, unlimited accounts on the top tier ([pricing](https://www.directadmin.com/pricing.php)) | Apache / LiteSpeed / nginx | Unix user + CloudLinux | Cheapest credible cPanel replacement |
| **CloudPanel** | Free | nginx + PHP-FPM | Unix user per site [unverified] | Single admin, not built for multi-tenant resale |
| **CyberPanel** | Free | OpenLiteSpeed | Unix user | **PSAUX ransomware hit about 22k instances** via CVE-2024-51567/51378 ([SOCRadar](https://socradar.io/blog/over-22000-cyberpanel-servers-at-risk-from-critical-vulnerabilities-exploitation-by-psaux-ransomware)); more auth-bypass CVEs in 2026 ([CVE](https://www.cve.org/CVERecord?id=CVE-2026-71964)) |
| **In-house panels** | — | — | — | Hostinger hPanel, SiteGround Site Tools, DreamHost, 20i StackCP, ScalaHosting SPanel ([WHT](https://webhosting.today/2026/08/14/the-field-of-cpanel-alternatives-keeps-widening-a-brand-new-one-comes-from-inside-cpanels-ecosystem/)) |
| **OpenPanel** | — | — | Each user gets their own container with their own web server, DB and network ([GitHub](https://github.com/stefanpejcic/openpanel)) | — |

**CloudLinux** is a cgroup-based LVE (CPU, RAM, IO, IOPS and process limits per user) plus the **CageFS** per-user filesystem namespace ([docs](https://docs.cloudlinux.com/cloudlinuxos/cloudlinux_os_components)). It added **cgroup v2** support in 2026 ([blog](https://blog.cloudlinux.com/cloudlinux-now-supports-cgroup-v2)). **LiteSpeed Enterprise** is a drop-in Apache replacement that reads .htaccess. **OpenLiteSpeed** is free but only partly supports .htaccess ([Enhance](https://enhance.com/product/features)).

## 4. What every serious provider converges on

1. **KVM everywhere.** Linode went UML → Xen → KVM; Nitro is KVM-based; Firecracker and Cloud Hypervisor run on KVM. The serious providers rely on a minimal VMM rather than on libvirt [inference]. Note: a $50k KVM escape bounty paid by Vercel had no CVE and no patch as of 2026-10-08 ([WHT](https://webhosting.today/2026/10/08/vercel-paid-50000-for-a-kvm-escape-that-has-no-cve-and-no-patch-every-vps-host-is-waiting-for-the-write-up/)).
2. **L3 routed fabric with BGP down to the host** (DO with GoBGP on each hypervisor, Latitude, Fly Anycast). L2 and ARP do not scale.
3. **Ceph for network block and object storage** (DO at 250 PB, Hostinger CephFS, Hetzner triple replication), with **local NVMe for performance tiers**.
4. **Own the control plane and the bare-metal installer.** Scaleway rejected Ironic and Tinkerbell. Equinix built Tinkerbell and then shut Metal down, so Tinkerbell's sponsor is gone. Hosts are leaving cPanel for in-house panels.
5. **Always-on DDoS in-house at the edge**: flow telemetry → divert to scrubbing → tunnel clean traffic back (OVH VAC, Vultr AMF, Hetzner Arbor). Cloudflare covers L7 for web hosting.
6. **Isolate network and storage from the hypervisor** (Nitro, OCI SmartNIC) at the high end; per-site containers or cgroups at the web-hosting end (Enhance, LVE).
7. **Live migration** as standard (Hetzner: <1 s blackout).

## Recommendation for KAREN

**Pick:**
- **Hypervisor:** KVM with **Cloud Hypervisor** (Rust, full VMs, hotplug, live migration) for VPS. Add **Firecracker** as a second runtime for app/container hosting, which covers StayCloud-style "each app in its own container" and Square Cloud-style workloads with VM-grade isolation.
- **Networking:** routed L3. Each host speaks BGP to the ToR (GoBGP-style or a Rust BGP crate), with /32 and /128 routes per VM. Use OVS or eBPF/XDP on the host for security groups.
- **Storage:** local NVMe as the default tier; **Ceph RBD** for volumes and snapshots, operated through automation only (DO's "no ssh" rule).
- **Bare metal:** build a small **in-house Rust provisioner** (Redfish/IPMI + iPXE + image streaming), following Scaleway's BATMAN. Do not depend on Ironic or Tinkerbell.
- **Web hosting:** a **native per-site container model** (Enhance/OpenPanel style: cgroups v2 + user namespaces + per-site PHP-FPM/OLS), clustered across nodes. Make cPanel/DirectAdmin an *optional* integration, not the core.
- **DDoS:** sFlow/NetFlow detection → BGP FlowSpec/RTBH → an **XDP scrubbing tier**. Cloudflare in front of shared web hosting. Peer at **IX.br SP** and colocate at Equinix SP4 or Ascenty.
- **Billing:** ship a WHMCS module on day one, because that is the lock-in hosts actually face.

**Avoid:**
- libvirt as the core abstraction (this is the VirtFusion path).
- OpenStack and Ironic (Scaleway's lesson).
- L2/VXLAN-everywhere fabrics without NIC offload.
- Per-account licensed panels as the foundation.
- Shipping a CyberPanel-like admin surface exposed to the internet.
- Depending on Equinix Metal or Tinkerbell's original sponsor.
