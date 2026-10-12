# MediaTek conntrack QoS extension allocation fix

A small fix for the conntrack QoS extension introduced by MediaTek's DSCP-learning patches in a vendor OpenWrt kernel.

## Symptom

Forwarded TCP connections can be reset intermittently when flow offload and conntrack QoS learning are enabled. Both software and hardware flow offload paths can be affected.

## Kernel baseline

- **Linux kernel:** `6.6.93`.
- **Vendor source:** [BPI-SINOVOIP/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel](https://github.com/BPI-SINOVOIP/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel).
- **Baseline commit:** [`56e0e77adad258ba05782fee8f94f00d17b0b991`](https://github.com/BPI-SINOVOIP/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel/commit/56e0e77adad258ba05782fee8f94f00d17b0b991).
- **Patch directory:** `target/linux/mediatek/patches-6.6/`.

The affected extension is introduced by `999-3004-netfilter-add-DSCP-learning-flow-to-xt_FLOWOFFLOAD.patch`. The same QoS storage is used by the nftables path added in `999-3016-netfilter-add-DSCP-learning-flow-to-nft_flow_offload.patch`.

The fix targets this vendor-patched baseline; other kernel trees require checking their extension allocator and DSCP-learning changes before applying it.

## Root cause

The DSCP-learning code introduces `NF_CT_EXT_QOS` and `struct nf_conn_qos`, but omits the extension from `nf_ct_ext_type_len[]`. Its length therefore defaults to zero.

`nf_ct_ext_add()` computes the new extension length from that table. A zero-length QoS entry does not reserve its payload, so a subsequently added extension can receive the same offset. QoS initialization, DSCP-value updates, packet-counter updates, and clearing can then overwrite storage belonging to other conntrack extensions. The affected storage depends on which extensions are enabled and allocated.

The structure is also missing from `total_extension_size()`, which feeds the compile-time size bound check. Separately, `nf_ct_qos_ext_add()` continues to `memset()` after allocation failure, creating a NULL-pointer dereference risk.

## Fix

The [patch](patches/999-3035-netfilter-fix-conntrack-qos-extension-size.patch) changes two files:

- `net/netfilter/nf_conntrack_extend.c`: include the QoS header, register `sizeof(struct nf_conn_qos)` in the extension-length table, and include it in the total-size calculation.
- `include/net/netfilter/nf_conntrack_qos.h`: return `NULL` immediately if `nf_ct_ext_add()` fails.

This preserves QoS learning and flow offload while giving the QoS payload its own allocated space.

## Applying the patch

Copy the patch into the baseline OpenWrt tree's `target/linux/mediatek/patches-6.6/` directory, then perform a clean kernel rebuild. Its sequence number places it after the DSCP-learning patches.

For an already prepared kernel tree containing the vendor patches, apply it with `patch -p1` from that kernel tree's root. Use one method per build to avoid applying it twice.

Deploy the rebuilt kernel together with its matching kernel modules.
