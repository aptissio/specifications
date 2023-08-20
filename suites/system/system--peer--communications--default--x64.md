# Overview
The system-layer Peer Communications suite abstracts the underlying communications transport from designs focussed on sending and receiving serialised data.

The suite provides sub-channels to allow multiple, segregated communications pathways between design elements to avoid conflicts in decentralised protocol design.

# Key Protocols
- <span style="color:#f37d2e">/system/peer-communications/default/x64</span> ([spec](../../protocols/system/peer-communications/default/x64/system--peer-communications--default--x64.pspec))
- <span style="color:#f37d2e">/system/peer-communications-channel/default/x64</span> ([spec](../../protocols/system/peer-communications-channel/default/x64/system--peer-communications-channel--default--x64.pspec))
- <span style="color:#f37d2e">/system/peer-communications-anc/default/x64</span> ([spec](../../protocols/system/peer-communications-anc/default/x64/system--peer-communications-anc--default--x64.pspec))

# Key Contracts
## <span style="color:#3482ba">/system/new/peer-communications/anc/linux-x64</span> ([spec](../../contracts/system/new/peer-communications/anc/linux-x64/system--new--peer-communications--anc--linux-x64.cspec))
Currently the only design to host the <span style="color:#f37d2e">/system/peer-communications/default/x64</span> ([spec](../../protocols/system/peer-communications/default/x64/system--peer-communications--default--x64.pspec)) protocol. It provides peer communications via the Agent Network Communications system developed by Code Valley.
```
sub /system/new/peer-communications/anc/linux-x64@aptissio(anc, cfg, log, task_sched,
    tman, PEER_COMMS_TASK_LABEL, PEER_COMMS_TASK_PRI) ->
    pcomms, anc_pcomms, local_peer_id
```
## <span style="color:#3482ba">/system/connect/peer-communications/anc/x64</span> ([spec](../../contracts/system/connect/peer-communications/anc/x64/system--connect--peer-communications--anc--x64.cspec))
Queue a request to connect to a new ANC peer.
```
sub /system/connect/peer-communications/anc@aptissio($, anc_pcomms, client_anp_addr) ->
    client_peer_id, request_queued, request_failed
```

## <span style="color:#3482ba">/system/send/peer-communications/default/x64</span> ([spec](../../contracts/system/send/peer-communications/default/x64/system--send--peer-communications--default--x64.cspec))
Send a message to a peer in the global messaging scope (without a channel).
```
sub /system/send/peer-communications/default/x64(send, pcomms, message, 
    peer_id_in, token_in, cb_proc) cb_token, cb_result_code
```

## <span style="color:#3482ba">/system/new/peer-communications-channel/default/x64</span> ([spec](../../contracts/system/new/peer-communications-channel/default/x64/system--new--peer-communications-channel--default--x64.cspec))
Create a new sub-channel for communications to peers
```
sub /system/new/peer-communications-channel@aptissio(pcomms, "BCH") -> pc_chan_bch
```

## <span style="color:#3482ba">/system/send/peer-communications-channel/default/x64</span> ([spec](../../contracts/system/send/peer-communications-channel/default/x64/system--send--peer-communications-channel--default--x64.cspec))
Send a message to peer via a peer communications sub-channel.
```
sub /system/send/peer-communications-channel/destination-channel@aptissio($, 
    pc_chan_bch, local_peer_id, mon_ch_id, minus_one, "den", MSG_LEN_MAX) -> 
    pack_msg_format, send_msg_den_token_out, send_successful, send_failed
```

## <span style="color:#3482ba">/system/receive/peer-communications-channel/default/x64</span> ([spec](../../contracts/system/receive/peer-communications-channel/default/x64/system--receive--peer-communications-channel--default--x64.cspec))
Define the process to receive messages for a peer communications sub-channel.
```
sub /system/receive/peer-communications-channel@aptissio(pc_chan_mon, "den",
    SEND_BUF_CAPACITY) -> unpack_msg_format, den_msg_src_peer_id, new_msg_arrived
```

## <span style="color:#3482ba">/system/send/peer-communications-channel/destination-channel/x64</span> ([spec](../../contracts/system/send/peer-communications-channel/destination-channel/x64/system--send--peer-communications-channel--destination-channel--x64.cspec))
Send a message directly to a peer communications sub channel without design time access to that channel.
```
sub /system/send/peer-communications-channel/destination-channel@aptissio($,
    pc_chan_bch, local_peer_id, mon_ch_id, minus_one, "den", MSG_LEN_MAX) ->
    pack_msg_format, send_msg_den_token_out, send_successful, send_failed
```

# Ancillary Contracts
- none
# Contract Voids
## <span style="color:#3482ba">/system/receive/peer-communications-channel/default/x64</span>
Define the process to receive messages for the global peer communications scope.
## <span style="color:#3482ba">/system/send/peer-communications-channel/timed/x64</span>
A variation of /system/send/peer-communications-channel/default/x64 which includes a response time out timer.


# Behaviours
The peer communications system creates a peer ID upon creation of a connection with a new peer. The peer ID will become void and potentially re-used if no communications with that peer is performed within a 24 hour period.

# Examples
## Example 1
```
defaults: data, default, x64, codevalley

9223372036854775807 -> ARCH_INT_MAX
-9223372036854775808 -> ARCH_INT_MIN

250 -> MSG_LEN_MAX
MSG_LEN_MAX + 1 -> SEND_BUF_CAPACITY

1 -> PEER_COMMS_TASK_PRI
"peer-comms" -> PEER_COMMS_TASK_LABEL


[0, 1] -> APPFLOW_SUCCESS, APPFLOW_FAILURE
[0, 1, 2] -> APPFLOW_PRE_ALWAYS, APPFLOW_PRE_SUCCESS, APPFLOW_PRE_FAILURE

asset("test--system--new--peer-communications--anc--linux-x64.elf") -> SITE
sub /behaviour/new/application/single-process/linux-x64(SITE, "", "", "", "", "app.config", "") -> bundle, cmdargs, cfg, log, sig, task_sched, start, shut

len("255.255.255.255:65536.255") -> ANP_LEN_MAX

sub /system/new/task-scheduler/init(task_sched) -> {
  sub new/integer($, ARCH_INT_MIN, ARCH_INT_MAX, -1) -> minus_one
  minus_one -> mon_eid
  sub new/bytesequence/reserve($, SEND_BUF_CAPACITY) -> send_buffer

  sub new/bytesequence/constant($, "MON") -> mon_ch_id

  sub new/anp-address/reserve($) -> client_anp_addr
  //   sub set/anp-address/constant($, client_anp_addr, "127.0.0.1", 8000, 1)
  sub new/bytesequence/reserve($, ANP_LEN_MAX) -> anp_ascii

  sub decode/bytesequence($, client_anp_raw, true) -> {
    sub unpack/anp-address/ascii($) -> client_anp_addr_cfg
  }, prog_index, _, {
    sub copy/anp-address($, client_anp_addr_cfg, client_anp_addr)
    sub set/bytesequence($, anp_ascii, "")
    sub serialise/ascii-anp-address($, anp_ascii, client_anp_addr)
    sub write/bytesequence/bookended/linux-x64@aptissio($, "CLIENT ADDRESS: ", anp_ascii, "")
  }, {
    sub clear/flag($, client_anp_valid_flag)
    sub set/bytesequence($, client_anp_invalid_reason, "Invalid client ANP address format.")
  }

  $ -> ginit
  sub write/constant/./linux-x64($, "Application initialisation complete.")
}

sub /system/new/tcp-client-manager/./linux-x64(log, cfg, task_sched, "tcp-client", PEER_COMMS_TASK_PRI + 1, PEER_COMMS_TASK_PRI) -> tcp_client
sub /system/new/tcp-server-manager/./linux-x64(log, cfg, task_sched, "tcp-server", PEER_COMMS_TASK_PRI, PEER_COMMS_TASK_PRI + 1) -> tcp_server
sub /system/new/timer-manager/./linux-x64(log, cfg, task_sched, "timer", 0) -> tman
sub /system/new/agent-network-comms/./linux-x64(log, cfg, task_sched, "agent-comms", 0, tman, tcp_client, tcp_server) -> anc, _, _, _, _, _, _, _, _, _

sub /system/new/peer-communications/anc/linux-x64@aptissio(anc, cfg, log, task_sched, tman, PEER_COMMS_TASK_LABEL, PEER_COMMS_TASK_PRI) -> pcomms, anc_pcomms, local_peer_id


sub /system/register/configuration-manager/string(cfg, false, true, "client.anp-address", "Client ANP address", ANP_LEN_MAX, 0, "127.0.0.1:8000.1") -> client_anp_raw, client_anp_valid_flag, client_anp_invalid_reason, {
  sub decode/bytesequence($, client_anp_raw, true) -> {
    sub unpack/anp-address/ascii($) -> _
  }, _, _, _, {
    sub clear/flag($, client_anp_valid_flag)
    sub set/bytesequence($, client_anp_invalid_reason, "Invalid client ANP address format.")
  }
}, _

sub /system/new/peer-communications-channel@aptissio(pcomms, "BCH") -> pc_chan_bch

sub /system/receive/peer-communications-channel@aptissio(pc_chan_bch, "def", MSG_LEN_MAX) -> {
  sub unpack/uint8($, 0, MSG_LEN_MAX) -> msg_len
  sub unpack/bytesequence($, msg_len, msg_len) -> bch_msg
}, def_msg_src_peer_id, {
  sub write/constant/inline/linux-x64@aptissio($, "BCH:")
  sub write/integer/inline/linux-x64@aptissio($, def_msg_src_peer_id, " : ")
  sub write/bytesequence/./linux-x64($, bch_msg)

  // Schedule a message to be send to the local MON channel
  sub observe/bytesequence@aptissio(bch_msg) -> _, bch_msg_len, _

  sub /system/send/peer-communications-channel/destination-channel@aptissio($, pc_chan_bch, \
      local_peer_id, mon_ch_id, minus_one, "den", MSG_LEN_MAX) -> {
    sub pack/uint8($, bch_msg_len)
    sub pack/bytesequence($, bch_msg)
  }, send_msg_den_token_out, {
      sub write/constant/inline/linux-x64@aptissio($, " MON ACK:")
      sub write/integer/./linux-x64($, send_msg_den_token_out)
  }, {
    sub write/constant/inline/linux-x64@aptissio($, "MON NACK:")
    sub write/integer/./linux-x64($, send_msg_den_token_out)
  }
}





sub /system/new/peer-communications-channel@aptissio(pcomms, "MON") -> pc_chan_mon
sub /system/receive/peer-communications-channel@aptissio(pc_chan_mon, "den", SEND_BUF_CAPACITY) -> {
  sub unpack/uint8($, 0, MSG_LEN_MAX) -> mon_msg_len
  sub unpack/bytesequence($, mon_msg_len, mon_msg_len) -> mon_msg
}, den_msg_src_peer_id, {
  sub write/constant/inline/linux-x64@aptissio($, "MON:")
  sub write/integer/inline/linux-x64@aptissio($, den_msg_src_peer_id, " : ")
  sub write/bytesequence/./linux-x64($, mon_msg)
}

sub /behaviour/read/text/standard-input/linux-x64@aptissio(bundle, start, MSG_LEN_MAX) -> msg_out, send_msg
sub observe/bytesequence@aptissio(msg_out) -> _, msg_out_len, _

sub /system/register/app-flow(send_msg, 0) -> PID, eid, ret_val, _, _, {

  sub write/constant/inline/linux-x64@aptissio($, "Send msg to BCH peer. [anp=")
  sub set/bytesequence($, anp_ascii, "")
  sub serialise/ascii-anp-address($, anp_ascii, client_anp_addr)
  sub write/bytesequence/inline/linux-x64@aptissio($, anp_ascii, ", token=")
  sub write/integer/inline/linux-x64@aptissio($, eid, ", len=")
  sub write/integer/inline/linux-x64@aptissio($, msg_out_len, "]\n")

  sub /system/connect/peer-communications/anc@aptissio($, anc_pcomms, client_anp_addr) -> client_peer_id, {
    sub /system/send/peer-communications-channel@aptissio($, pc_chan_bch, client_peer_id, eid, "def", SEND_BUF_CAPACITY) -> {
      sub pack/uint8($, msg_out_len)
      sub pack/bytesequence($, msg_out)
    }, eid_out, {
      sub write/constant/inline/linux-x64@aptissio($, " BCH ACK:")
      sub write/integer/./linux-x64($, eid_out)
    }, {
      sub write/constant/inline/linux-x64@aptissio($, "BCH NACK:")
      sub write/integer/./linux-x64($, eid_out)
    }

  }, {
    sub write/constant/inline/linux-x64@aptissio($, "Failed to connect to BCH peer: ")
    sub set/bytesequence($, anp_ascii, "")
    sub serialise/ascii-anp-address($, anp_ascii, client_anp_addr)
    sub write/bytesequence/./linux-x64($, anp_ascii)
  }
  // Always return successful message transmission.
  sub set/integer($, ret_val, 0)
}

```