# Observability Runtime Acceptance — 2026-08-13

## Result

The three-VM GPU `fl-closure-smoke` run completed the intended full-core business
closure and validated the new observability contracts against live runtime state.
Six UEs registered and established PDU sessions, both Path subscriptions delivered
callbacks independently, PyMTLF-A/B trained on the RTX 3080, PyMTLF-C published and
cut over a federated model, and the current-run summary reported post-cutover
evaluated accuracy without an FL failure signature.

Normal teardown deleted both Consumer-owned subscriptions and stopped all runtime
domains. Internal Model Monitor cleanup did not converge within the bounded 210
second wait because the NWDAF-C gateway returned repeated 503 responses. The script
reported the two pending resource IDs and continued teardown as designed. This is a
partial cleanup result, not a business-closure failure.

## Identity and preflight

- Infrastructure revision: `95cf6ab`
- Config set: `observability-acceptance-20260813`
- Effective config hash: `d466833f1c21064289d85be4832a57feac13c9a425cfccc2f9c0a9f82271c02f`
- Scenario: `fl-closure-smoke`
- Dataset set: `2915b05719f997d135d8a64c40f7d684e1f78e0ab2a3c483595b2bf545de4029`
- PyAnLF revision: `08798f15c369`
- PyMTLF revision: `7e8ab7f23bf5`
- NWDAF revision: `318ac19d8b02`
- UPF revision: `234bae063ffb`
- GPU: RTX 3080 10 GiB, driver `535.183.01`
- VM allocation: Core 4 GiB, Path A 3 GiB, Path B 3 GiB

Preflight reported zero failures and one warning: only 2 MiB of the 2 GiB swap was
free. Host `MemAvailable` was about 27 GiB before VM boot, 24 GiB after boot, and no
less than about 19 GiB during the observed runtime. Workspace, VirtualBox, and Docker
storage had about 224 GiB free before startup. The run used Ubuntu 22.04 VMs and the
existing provisioned artifacts; no VM rebuild occurred.

## Startup and business readiness

`make vm-up` brought Core, Path A, and Path B to `running`. Persistent network
reconciliation verified 14 Core aliases and seven aliases on each Path VM. Both Path
VMs loaded a `gtp5g` module whose vermagic matched kernel `5.15.0-171-generic`.

`make experiment-start` then completed without rollback:

- all 23 Guest units became active;
- all six current systemd invocations reported Registration and PDU Session
  `successful`;
- WebConsole became HTTP-ready at `http://192.168.56.10:5000`;
- all five ML containers became healthy with the same config identity;
- PyMTLF-A/B reported `cuda:0`, NVIDIA runtime, CDI
  `nvidia.com/gpu=all`, and `CUDA=true`;
- Consumer discovered NWDAF-A/B through NRF and created two distinct resources.

The two Consumer resources used the intended boundaries:

| Path | Provider NF instance | TAC | API root |
|---|---|---|---|
| A | `11111111-1111-4111-8111-111111111111` | `000001` | `http://192.168.57.41:8080` |
| B | `22222222-2222-4222-8222-222222222222` | `000002` | `http://192.168.57.51:8080` |

The new per-Path counters started at zero and increased independently to 41/41 before
the final observation and 42/42 before stop. Their final callback timestamps differed
by about one millisecond. The retained legacy global count was therefore not needed
to establish dual-Path delivery.

## FL closure evidence

PyMTLF-C created two active Model Monitor subscriptions. The live current-run summary
then reported the first complete closure:

- degradation trigger: `2026-08-13T09:33:09.848447898Z`;
- process: `3ea8fb28-a158-4c27-80ff-1ae42246b185`, two scopes;
- preparation: both `111...` and `222...` participants;
- A/B local rounds: 0 and 1, with final validation from both clients;
- base WAPE: `0.4216419625395333`;
- candidate WAPE: `0.2641478662251894`;
- publication: model `1786613593269`, two required scopes;
- both scopes adopted and cutover complete at `09:33:13.672018497Z`;
- post-cutover accuracy: `evaluated=true`, `triggered=false` at
  `09:34:43.666870028Z`;
- explicit FL failure signature: `not-seen`.

During training, PyMTLF-A/B remained healthy at about 706 MiB container memory each.
The two GPU processes used about 400 MiB VRAM each; total GPU memory reported about
829 MiB of 10 GiB.

During the following observation step, elevated execution awaited user approval while
the degraded replay remained active. A second FL process later triggered and completed
two rounds, publication, adoption, cutover, and post-cutover accuracy. Its candidate
WAPE was worse than the current model (`0.264009146534429` to
`0.35605916968210677`), but the bounded smoke profile intentionally had
`enforcePerformanceGate=false`, so the candidate was still published. The available
timestamps establish that the experiment remained active long enough for a second
closure; they do not attribute that interval to log-query execution. This does not
invalidate the first closure, but it shows that a single-closure acceptance should
stop promptly after its first accepted post-cutover evidence.

## Observability acceptance

The new interfaces behaved as follows:

- `services-status` distinguished process state from current-invocation UE business
  readiness and correctly showed six successful UEs while running and six inactive
  UEs after stop;
- `subscriptions-status` showed A/B provider, TAC, correlation, exact Location,
  resource state, callback count, and last callback time;
- focused Consumer, Network, and ordinary NRF queries selected their actual systemd
  units and returned UTC timestamps;
- `experiment-status` returned a complete, successful snapshot while all domains
  were available;
- `ml-status` limited evidence to matching config identities and current Container
  starts, then displayed the complete first FL closure and post-cutover accuracy.

One status scaling limitation and two observations requiring controlled measurement
were exposed:

1. `ml-status` reads all PyMTLF-A/B/C logs since each Container `StartedAt`. Once the
   run accumulated descriptor and health traffic, a query took about 54–70 seconds.
   The result remained correct, but cost grows with current-run log volume.
2. The tool call containing the all-source, no-follow `--tail 1` query reported about
   578 seconds of wall time, but that interval included waiting for elevated-execution
   approval. It is not a measurement of `logs.sh` execution time and provides no
   evidence that the logger itself took 578 seconds. A pre-authorized, command-only
   benchmark is required before making a performance claim. Independently, `--tail 1`
   applies to the combined owned-unit journal per VM, so it cannot visibly list every
   selected unit even though focused queries prove Consumer and Network selection.
3. One concurrent observation attempt returned `failed to query Vagrant machine
   states` for `services-status`, while an immediate serial retry passed. The logger
   implementation also launches one Vagrant SSH process per VM concurrently. These
   facts make Vagrant CLI contention a hypothesis worth testing, not a confirmed cause
   of either the status failure or the tool-call wall time.

## Teardown and cleanup evidence

`make experiment-stop` reconstructed two active PyMTLF-C Model Monitor subscription
IDs before deleting the Consumer resources. Both Consumer DELETEs succeeded and the
callback server stopped. The internal cleanup wait then timed out after 210 seconds
with these IDs pending:

- `f2e256c2-fef5-4109-bc38-f9dd82be52aa`
- `f3abffde-f335-45e5-bd3d-a291260cb294`

For the first ID, the retained PyMTLF-C log shows five DELETE attempts to the
NWDAF-C internal gateway between `09:46:35Z` and `09:49:05Z`; every response was 503.
No DELETE or removed evidence for the second ID appeared before bounded timeout and
application shutdown. Source inspection shows the gateway can return 503 when local
AnLF backend admission fails or when a peer NWDAF delete fails. The retained Host
Container log does not identify which downstream branch produced this run's 503, so
the report does not assign a narrower root cause.

After timeout, WebConsole, all five ML containers, all 23 Guest units, and all three
VMs stopped normally. Final state:

- Core, Path A, Path B: `poweroff`;
- five ML containers: retained, `Exited (143)`;
- Consumer resources: local state `deleted`, Consumer service inactive;
- 23 Guest units: inactive before VM halt;
- repository and all submodules: clean and unchanged;
- Host `MemAvailable`: about 27 GiB after shutdown;
- workspace free storage: about 222 GiB.

## Follow-up boundaries

Infrastructure-only follow-up can improve the two read paths without changing
NWDAF, PyAnLF, or PyMTLF:

1. benchmark pre-authorized serialized and concurrent all-VM log queries before
   deciding whether to serialize Vagrant CLI calls or replace them with resolved SSH
   transports;
2. make FL summaries incremental or bounded by a retained cursor/current-process
   window rather than reparsing all current-container logs;
3. ensure runtime acceptance stops immediately after the first post-cutover success
   when the scenario is intended to prove one closure.

The persistent 503 cleanup result is component/runtime behavior, not something to
mask by merely extending the Infrastructure timeout: all five attempts over more than
two minutes failed. Any NF/ML change should be proposed separately with the Guest
gateway and downstream provider logs available.
