# Your source of truth can be wrong. Would network CI catch it?

Change one OSPF area in a database. Regenerate the router configuration. Apply it. Now two routers on the same link disagree about which area they belong to, and traffic between two LANs can stop.

That’s the experiment in **NetAlps Infrahub Demo**, a hands-on lab built with Infrahub, Containerlab, FRRouting, and pyATS. It extends the earlier two-router NetAlps demo by generating routing configuration from a source of truth and testing what happens when the data itself introduces a regression.

## From network intent to a running router

Infrahub stores the demo’s device and interface records. The custom schema describes fields such as router IDs, interface addresses, OSPF areas, passive-interface settings, and BFD parameters. The bootstrap script loads the lab data. A Python generator retrieves the router records through GraphQL and renders an `frr.conf` for each router.

Containerlab then wires up the topology and mounts the generated files into the FRRouting containers. The topology YAML also defines the virtual links and assigns the lab’s transit addresses. That split gives you three places to inspect a change: the input record, the rendered configuration, and the state on the running router.

```text
Infrahub records → GraphQL → FRR configuration → Containerlab → pyATS
```

The topology is deliberately easy to trace.

```text
host-left ── R1 ── R2 ── R3 ── R4 ── host-right
```

With one routed path between the two hosts, the edge routers each expect one OSPF neighbor and the transit routers expect two. The three inter-router links use OSPF area 0 in the baseline, with BFD configured on each. Traffic between the hosts crosses all four routers.

## Three questions to ask before trusting a change

**Is the lab ready?** The pyATS pre-check verifies that the six containers are running, checks FRRouting and transit interface addresses, and queries Infrahub for basic health and data availability.

**Does the network work?** The post-check looks for the expected OSPF neighbors, verifies selected remote LAN and loopback routes, checks BFD peers, and sends traffic between the two hosts in both directions. It also tests selected router loopback paths. Neighbor state, route installation, and packet forwarding give you different clues about where a change failed.

**Does deployed state match the source of truth?** A focused audit reads the expected IP for `frr-rtr-01/eth1` from Infrahub and checks that it appears on the deployed interface. The audit can skip if the API query fails or the record has no address. Its scope is that one IP. The OSPF, routing, and connectivity tests are what expose the failure scenario.

## Change the data and see what breaks

The local demo starts with a passing baseline. It saves the original area on `frr-rtr-03/eth1`, then changes that field in Infrahub from **area 0** to **area 1**. The generator renders the new value and the script clears the old interface area before reloading FRRouting. The peer on `frr-rtr-02/eth2` stays in area 0.

The expected result is that the R2–R3 adjacency drops. With no alternate path, the edge routers lose their routes to the opposite LAN and the host-to-host checks fail. The script treats an unexpectedly passing post-check as an error.

Recovery follows the same path as the change. Restore the original Infrahub value, regenerate the configuration, reload the router, and rerun the post-check. A passing recovery run gives you evidence that both routing and reachability came back.

This is the part I’d bring into a change review. A generator can faithfully translate a value that breaks the network. An independent test of the expected neighbor relationships, routes, and traffic paths gives you a way to catch it.

## Try the lab

Start the lab’s Infrahub stack and set up the Python environment using the repository README. Then run the failure demo from the repo root against the lab instance.

```bash
bash scripts/failure_demo.sh
```

The repository also includes a GitLab CI definition for starting Infrahub, loading the schema and data, generating configurations, deploying the lab, validating it, and cleaning up. It targets a shell runner tagged `frr-infrahub`. Its default paths assume a nested `netalps_infrahub/` checkout, so adjust them for the standalone public repository.

[Explore the topology, schema, generator, and tests on GitHub](https://github.com/jeyriku/netalps_infrahub_public)

**If someone entered a valid value that would break your network, which test would catch it before deployment?**

#NetDevOps #NetworkAutomation #Infrahub #Containerlab #pyATS #NetworkEngineering
