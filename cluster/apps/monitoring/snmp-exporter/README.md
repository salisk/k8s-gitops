# Shared SNMP exporter

Flux deploys chart **9.20.0** (exporter **v0.31.0**), dependent on victoria-metrics.
The internal ClusterIP Service listens on TCP 9116; there is no ingress or HTTPRoute.
`serviceMonitor.selfMonitor.enabled: true` enables `/metrics` self-monitoring;
`serviceMonitorSelf.enabled` is not the chart API. VictoriaMetrics discovers it
through Prometheus resource conversion. Device monitoring stays disabled with
`serviceMonitor.enabled: false` and `serviceMonitor.params: []`; no NAS is contacted.
Gatus checks `/-/healthy` for HTTP 200. This checks exporter availability, not SNMP access.

## Enable NAS monitoring

Enable SNMPv3 in DSM with a read-only monitoring account and `authPriv`.
Match authentication/privacy protocols to DSM. Allow UDP 161 from the exporter's
actual cluster egress addresses through DSM/network firewalls; confirm cluster routing.
Provision an encrypted or externally managed Secret in `monitoring` with an `auth.yml`
key containing only top-level `auths` and a unique profile with `version: 3`,
`security_level: authPriv`, `username`, `password`, `auth_protocol`, `priv_protocol`,
and `priv_password`. No credentials or Secret are supplied here.

Mount it with `extraSecretMounts` (`name`, `secretName`, `mountPath`, `readOnly: true`,
`defaultMode`) at `/run/secrets/snmp-exporter`, readable by UID 1000.
Keep `/etc/snmp_exporter` intact and chart `config` unset to preserve vendor modules.
Container args replace the default CMD, so supply **both** files:

```yaml
extraArgs:
  - --config.file=/etc/snmp_exporter/snmp.yml
  - --config.file=/run/secrets/snmp-exporter/auth.yml
```

Avoid duplicate auth/module names across files. Restart the workload after auth changes
or arrange Secret reloads. Once authentication and connectivity work, configure:

```yaml
serviceMonitor:
  enabled: true
  params:
    - name: nas # Stable identifier; replace target and auth before enabling.
      target: <NAS address>
      auth: [<Secret auth profile>]
      module: [synology, if_mib, hrStorage]
  relabelings:
    - sourceLabels: [__param_target]
      targetLabel: instance
```

The chart field is singular `module`. Relabeling sets `instance` to the NAS address;
chart metric relabeling also sets `instance` to the target and `target` to its name.
Review scrape interval/timeout for the combined walks before enabling.

Synology dashboard 14284 revision 10 has dashboard UID `i55CRAlMk`.
Its panels reference datasource UID `Prometheus`, but the configured datasource UID
is `prometheus` (name `Prometheus`). The `DS_PROMETHEUS` input mapping does not fix
those panel references. Fixing these bindings is a separate follow-up from this deployment.
