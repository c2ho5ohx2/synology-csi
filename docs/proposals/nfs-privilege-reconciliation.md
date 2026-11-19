# NFS privilege reconciliation and node selection

Configurable client addresses are discovered at NFS stage and during
missing-export recovery. Staging can add rules for newly joined nodes when
kubelet first stages a volume there. Node deletion can leave old rules in DSM
because the stage shortcut checks whether every desired client has access.

## Storage node selection

Add an optional `--nfs-client-address-node-selector` flag and expose it as
`node.nfsClientAddress.nodeSelector` in Helm. An empty value includes all nodes
under the existing policy. Parse the selector with Kubernetes `labels.Parse`
at startup and pass it to the Node list operation. Require valid addresses
for every matching node. Fail discovery with a clear error if no nodes match
or if the node requesting stage or export recovery is outside the selection.

Use the same storage eligibility label for the plugin DaemonSet and for
workloads with NFS PVCs. For example, a chart can set `node.nodeSelector` to
`{storage-access: enabled}` for plugin placement and set
`node.nfsClientAddress.nodeSelector` to `storage-access=enabled` for export
eligibility. Workloads can use the matching Pod `nodeSelector` or node
affinity. Drain affected workloads and verify placement before changing
eligibility for a node with active mounts.

Test empty and malformed selectors, match expressions, joins, changes to the
eligibility label, missing eligible-node annotations, and an NFS request from
a node outside the selection. Confirm the default chart renders every node
eligible for discovery and that the selector requires only the existing Node
list permission.

## Automatic privilege updates

Make reconciliation opt-in and define what constitutes a driver-managed NFS
share before enabling revocation. Use CSI PV ownership and volume attributes
to identify the DSM and share. Require explicit ownership for
statically provisioned PVs and for any share with administrator-managed
rules. A save replaces a share's rule list. Authoritative removal requires
agreement that the driver owns the entire list.

Watch Node membership, eligible-node label and address changes, and the
managed NFS PV inventory. Recompute the desired address set from the latest
complete Node snapshot. Coalesce related events and bound retries per DSM.
An incomplete API response or an invalid Node annotation should leave the
existing rules intact and surface an error. Add an explicit policy for empty
eligible sets before allowing a reconciler to clear access.

Coordinate writes from reconciliation, staging, and export recovery across
shares on the same DSM. Synology CSI already verifies saves and can retry
when DSM does not persist them. Investigate one active writer per DSM and a
request path for node operations to await its result. A Kubernetes Lease can
elect the writer, but expiry alone cannot fence an in-flight DSM save.
Establish failure and mixed-version rollout behavior before enabling
concurrent writers from different instances. Keep the forced regeneration
trigger for missing exports.

Compare managed rules against the desired set before saving. Test that
identical sets skip writes, removal and address changes revoke stale rules
under explicit ownership, and retries after transient failures converge
without publishing a partial configuration. Exercise concurrent requests on
different shares using a fake DSM, then validate write serialization and
rule handling in a lab across supported DSM versions.

Add Node and PV watches and coordination permissions only where the selected
design needs them. Audit rule changes and restrict updates to the selected
Node annotation and eligibility labels. Plan staged upgrades with a single
address source and writer policy across all participating node plugins.
