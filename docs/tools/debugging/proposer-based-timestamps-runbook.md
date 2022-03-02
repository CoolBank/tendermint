---
order: 3
---

# Proposer-Based Timestamps Runbook

Version v0.36 of Tendermint added new constraints for the timestamps included in
each block created by Tendermint. The new constraints mean that validators may 
fail to produce valid blocks or may issue `nil` `prevotes` for proposed blocks
depending on the configuration of the validator's local clock.

## What is this document for? 

This document provides a set of actionable steps for application developers and
node operators to diagnose and fix issues related to clock synchronization and
configuration of the Proposer-Based Timestamps [SynchronyParams]().

## Requirements

To use this runbook, you must be running a node that has [prometheus metrics endpoint enabled]()
and the Tendermint RPC endpoint enabled.

It is helpful, but not entirely necessary, to also run a Prometheus metrics collector
gather and analyze metrics from the Tendermint node.

## Debugging a Single Node

Use this runbook if you observe that your validator is frequently voting `nil` for a block that the rest
of the network votes for or if your validator node is frequently producing block proposals
that are not voted for by the rest of the network.

### Check Timely Metric

Tendermint exposes a histogram metric for the difference between the timestamp in the proposal
the and the time read from the node's local clock when the proposal is received.

The histogram exposes multiple metrics on the Prometheus `/metrics` endpoint called
* `tendermint_consensus_proposal_timestamp_difference_bucket`.
* `tendermint_consensus_proposal_timestamp_difference_sum`.
* `tendermint_consensus_proposal_timestamp_difference_count`.

Each metric is also label with the key `is_timely`, which can have a value of
`true` or `false`. 

#### From the Prometheus Collector UI
If you are running a Prometheus collector, navigate to the query web interface and select the 'Graph' tab.

Issue a query for the following:

```
tendermint_consensus_proposal_timestamp_difference_count{is_timely="false"} /
tendermint_consensus_proposal_timestamp_difference_count{is_timely="true"}
```

This query will graph the ratio of proposals the node considered timely to those it
considered not timely. If the ratio is increasing, it means that your node is consistently
seeing more proposals that are far from its local clock. If this is the case, you should
check to make sure your local clock is properly synchronized to NTP.

#### From the `/metrics` url

If you are not running a Prometheus collector, navigate to the `/metrics` endpoint
exposed on the Prometheus metrics port.

Search for the `tendermint_consensus_proposal_timestamp_difference_count` metrics.
This metric is labeled with `is_timely`. Investigate the value of
`tendermint_consensus_proposal_timestamp_difference_count` where `is_timely="false"`
and where `is_timely="true"`. Refresh the endpoint and observe if the value of `is_timely="false"`
is growing.

If you observe that `is_timely="false"` is growing, it means that your node is consistently
seeing proposals that are far from its local clock. If this is the case, you should check
to make sure your local clock is properly synchronized to NTP.

### Check Clock sync w/ NTP

`NTP` configuration and tooling is very dependent on the operating system and distribution
that your validator node is running on. This guide assumes you have `timedatectl` installed with 
[chrony](https://chrony.tuxfamily.org/), a popular tool for interacting with time 
synchronization on Linux distributions. If you are using an operating system or 
distribution with a different time synchronization mechanism, please consult the 
documentation for your operating system to check the status and re-synchronize the daemon.

#### Check your NTP Daemon Status

Check the status of your local `chrony` `ntp` daemon using by running the following:

```
$ chronyc tracking
```

If the `chrony` daemon is running, you will see output that indicates its current status.
If the `chrony` daemon is not running, restart it and re-run `chronyc tracking`. The
`System time` field of the response should show a value that is much smaller than a 100
milliseconds. If the value is very large, restart the `chronyd` daemon.

```
$ timedatectl
```

From the output, ensure that `NTP enabled` is `yes`. If `NTP enabled` is `no`, run:

```
$ timedatectl set-ntp true
```
Re-run the `timedatectl` command and verify that the change has taken effect.

## Debugging a Network

If you observe that a network is frequently failing to produce blocks and suspect
it may be related to clock synchronization, use the following steps to debug and correct the issue.

### Check Prevote Message Delay

Tendermint exposes metrics that help determine how synchronized the clocks on a network are.

These metrics are visible on the Prometheus `/metrics` endpoint and are called:
* `tendermint_consensus_quorum_prevote_delay`
* `tendermint_consensus_full_prevote_delay`

These metrics calculate the difference between the timestamp in the proposal message and
the timestamp of a prevote that was issued during consensus. 

The `tendermint_consensus_quorum_prevote_delay` metric is the interval in seconds
between the proposal timestamp and the timestamp of the earliest prevote that
achieved a quorum during the prevote step.

The `tendermint_consensus_full_prevote_delay` metric is the interval in seconds
between the proposal timestamp and the timestamp of the latest prevote in a round
where 100% of the validators voted.

#### From the Prometheus Collector UI
If you are running a Prometheus collector, navigate to the query web interface and select the 'Graph' tab.

Issue a query for the following:

```
sum(tendermint_consensus_quorum_prevote_delay) by (proposer_address)
```
This query will graph the difference for each proposer on the network.

If the value is much larger for some proposers, then the issue is likely related to the clock
synchronization of their nodes. Contact those proposers and ensure that their nodes
are properly sync'd to NTP using the steps for [debugging a single node][].

If the value is relatively similar for all proposers you should next compare this
value to the `SynchronyParams` values for the network.

#### From the `/metrics` url

### Checking Synchrony

To determine the currently configured `SynchronyParams` for your network, issue a 
request to your node's RPC endpoint. For a node running locally with the RPC server
exposed on port `26657`, run the following command:

```
# curl localhost:26657/consensus_params
```

The json output will contain a field named `synchrony`, with the following structure:

```json
{
  "precision": "500000000",
  "message_delay": "3000000000"
}
```

The `precision` and `message_delay` values returned are listed in milliseconds.
If the `tendermint_consensus_quorum_prevote_delay` often approaches `precision` + `message_delay`,
then the value selected for these parameters is too small. Your application will
need to be updated to set larger values 

### Updating SynchronyParams

Updates to the consensus parameters must be initiated by the application. If
the application was built using the CosmosSDK, then these parameters can be updated
programatically using a governance proposal. For more information, see the [CosmosSDK
documentation]().

If the application does not implement a way to update the consensus parameters
programatically, then it must be updated to do so. The consensus parameters are
updated as part of the `FinalizeBlock` method. For more information on updating
consensus parameters see the [FinalizeBlock documentation]().
