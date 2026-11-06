# Bridge Util HAL Documentation

## Scope, evidence, and terminology

Bridge Utility HAL provides vendor-facing bridge updates, interface discovery and membership queries, interface-list editing, and OEM/SoC pre/post configuration hooks. Its companion network-interface header defines configuration data, not executable network APIs.

This is the canonical specification, reached through the repository's `README.md` symlink. Public-header guarantees take precedence over broad introductory descriptions. The brief in `include/bridge_util_hal.h` mentions modes, connection enable/disable, QoS and IPv4 configuration, but does not declare dedicated mode, QoS, counter, link-callback, trunk/access-mode, or IPv4 setter/getter APIs. Do not infer those capabilities from that brief.

Evidence labels used throughout:

- **Header guarantee:** a declaration or its associated public comment.
- **Macro observation:** behavior directly visible in the logging macro, not a vendor implementation guarantee.
- **Legacy guidance:** retained expectations from the previous specification, not independently established by the headers.
- **Vendor-dependent:** implementation/platform choices that require integration evidence.
- **Unspecified:** “Unspecified by the current public contract.”

Source references below are repository-relative: **B** means `include/bridge_util_hal.h`; **N** means `include/network_interface.h`. Each table identifies exact symbols. Both headers were read completely. No function implementation or call-site source is present in the inspected tracked repository; no runtime or vendor conformance is claimed.

| Acronym | Meaning |
| --- | --- |
| HAL | Hardware Abstraction Layer |
| RDK-B | Reference Design Kit for Broadband Devices |
| OEM / SoC | Original Equipment Manufacturer / System on Chip |
| OVS | Open vSwitch; referenced in comments, not a universal dependency |
| GRE | Generic Routing Encapsulation |
| MoCA | Multimedia over Coax Alliance |

## Architecture and supported capabilities

~~~mermaid
flowchart TD
    U["Upper-stack bridge and network management consumers"]
    B["Bridge Utility HAL: include/bridge_util_hal.h"]
    N["Network data contract: include/network_interface.h"]
    V["Vendor or platform bridge integration"]
    I["Underlying bridge and network interfaces"]
    P["Physical or logical ports"]
    U -->|"Declared function calls"| B
    U -.->|"Optional use of declared data types"| N
    B -.->|"Conceptual implementation boundary"| V
    N -.->|"Conceptual data consumption; no functions declared"| V
    V -.->|"Vendor-dependent mechanism"| I
    I -.->|"Conceptual attachment"| P
~~~

Solid edges show the public function-call boundary; dashed edges show conceptual or vendor-dependent relationships, not proven runtime dependencies. The headers do not select Linux bridge, OVS, a driver stack, or a particular SDK. `network_interface.h` is not included by `bridge_util_hal.h`, and neither declares a conversion between their structures or enums.

| Capability | Explicit header evidence | Observation, dependency, or limit |
| --- | --- | --- |
| Bridge create/update/delete | B: `updateBridgeInfo`, pre/post hook comments | Actual operation values refer to missing `enum OVS_CMD`; implementation and partial-failure behavior are unspecified. |
| Vendor interface discovery | B: `getVendorIfaces` | NUL-terminated string or NULL for no interfaces; list grammar and ownership are unspecified. |
| Interface existence | B: `checkIfExists` | Vendor-specific names; 0/-1 result, without differentiated error reasons. |
| Bridge membership query | B: `checkIfExistsInBridge` | Tests attachment; no complete bridge-information getter is declared. |
| List editing | B: `removeIfaceFromList` | Missing name causes no action and no error; mutation details and tokenization are unspecified. |
| Platform hooks | B: `HandlePreConfigVendor`, `HandlePostConfigVendor` | OEM/SoC-specific configuration before/after bridge updates; no initialization or shutdown API. |
| Network configuration representation | N: `Gateway_Config_Non_Ovs_Bridge`, `IF_TYPE`, `BR_CMD` | Holds names, IPv4 text, MTU, VLAN ID, interface kind and command. No apply/query function, transport, transaction API or automatic mapping to B is declared. |
| Logging | B: `bridge_util_log` | Macro observation: formats shared buffers and writes/flushes only when `logFp != NULL`. No log-level argument or actual stdout fallback. |

Bridge VLAN and interface-list fields represent configuration information; they do not establish switch VLAN tagging, packet counters, trunk/access modes, or link notification capabilities.

## Operational sequence and lifecycle

~~~mermaid
sequenceDiagram
    participant C as Management consumer
    participant H as Bridge Utility HAL vendor implementation
    Note over C,H: Illustrative workflow, not initialization or a transaction guarantee
    C->>H: getVendorIfaces()
    H-->>C: NUL-terminated char pointer or NULL
    Note over C: Ownership, lifetime and release are unspecified
    C->>H: checkIfExists(iface_name)
    H-->>C: int 0 or -1
    C->>H: checkIfExistsInBridge(iface_name, bridge_name)
    H-->>C: int 0 or -1
    opt Local interface-list editing
        C->>H: removeIfaceFromList(str, sub)
        H-->>C: void
    end
    Note over C: Caller supplies bridgeDetails; no bridge-information getter declared
    Note over C: Network data may use Gateway_Config_Non_Ovs_Bridge; no network apply/query API declared
    C->>H: HandlePreConfigVendor(bridgeInfo, InstanceNumber)
    H-->>C: int 0 or -1
    alt Pre-hook succeeds
        C->>H: updateBridgeInfo(bridgeInfo, ifNameToBeUpdated, Opr, type)
        H-->>C: int 0 or -1
        alt Update succeeds
            C->>H: HandlePostConfigVendor(bridgeInfo, Config)
            H-->>C: int 0 or -1
            Note over C,H: Post-hook failure recovery is unspecified
        else Update fails
            Note over C,H: Partial effects, post-hook-on-failure and rollback are unspecified
        end
    else Pre-hook fails
        Note over C,H: Whether to continue, retry or clean up is unspecified
    end
    Note over C,H: No rollback or release API declared; cleanup ownership requires agreement
~~~

**Header guarantee:** B's hook comments say the client calls `HandlePreConfigVendor` before `updateBridgeInfo`, and `HandlePostConfigVendor` after it. The diagram's success-path branching is an illustrative conservative workflow, not a required failure policy. Ordering between discovery, existence queries and list edits is unspecified.

Neither header establishes boot-time initialization, hardware readiness, atomicity, completion timing, retries, idempotency, or rollback. Integer status describes the call result but does not prove synchronous hardware convergence. Whether post-configuration must run after failed or partially applied updates is unspecified. Failure-output state, retained pointers, cleanup responsibilities and recovery actions are also unspecified.

No callback or asynchronous notification API is declared. That absence does not prove that vendor implementations never perform asynchronous work.

## Exhaustive public function inventory

The signatures below reproduce the declarations, including `extern` where present. Source **N** declares **no functions**. There are **7 functions**, all in **B**.

For every function, unless explicitly stated in its row: pointer nullability, allocation/retention, lifetime beyond the call, release responsibility, thread safety, re-entry, blocking duration, timeouts and hardware-completion semantics are **Unspecified by the current public contract.** Input pointer direction does not grant ownership transfer or permission to free a returned pointer.

| Function and exact signature | Purpose and documented parameter directions | Valid values, capacities, units and termination | Actual return and documented meaning | Memory / execution caveats and source |
| --- | --- | --- | --- | --- |
| `extern int updateBridgeInfo(bridgeDetails *bridgeInfo, char* ifNameToBeUpdated, int Opr , int type);` | Update bridge details. `[in] bridgeInfo`: complete bridge information. `[in] ifNameToBeUpdated`: interface to delete/update, applicable only during sync. `[in] Opr`: operation. `[in] type`: interface type. | `ifNameToBeUpdated` must be NUL-terminated; examples `moca0`, `wifi0`, `eth0`, with no explicit maximum length. `Opr` must come from `enum OVS_CMD` cast to int, but that enum is absent. `type` comes from `enum INTERFACE_TYPE` cast to int; sync delete uses `IF_OTHER_BRIDGEUTIL`. Struct capacities appear below. | `int`: 0 success; -1 failure. No richer or partial-failure code. | Caller supplies structure and name; mutation despite `[in]`, copying, retention and cleanup unspecified. B: `updateBridgeInfo`. |
| `extern int checkIfExists(char* iface_name);` | Check interface existence. `[in] iface_name`: vendor-specific interface name. | Maximum length, accepted encoding, NUL rule and validation unspecified. | `int`: 0 success / `INTERFACE_EXIST`; -1 failure / `INTERFACE_NOT_EXIST`. No separate operational-error status. | No output allocation declared; pointer retention/release unspecified. B: `checkIfExists`. |
| `extern void removeIfaceFromList(char *str, const char *sub);` | Remove an interface from a list. `[in] str`: interface list, example `wl0 wl11 moca0 ath0 eth3`. `[in] sub`: name to remove, example `moca0`. | Example suggests space separation, but exact token grammar, matching and repeated-name behavior unspecified. No capacity or NUL rule. | `void`: no status. If name is absent, no action and no error. | Mutable `str` suggests in-place editing, but comment labels it input, not output; resulting formatting, required writable capacity and failure outputs unspecified. `sub` is const-qualified. B: `removeIfaceFromList`. |
| `extern int checkIfExistsInBridge(char* iface_name, char *bridge_name);` | Check attachment. `[in] iface_name`: examples `erouter0`, `eth0`, `eth1`, `wlan0`. `[in] bridge_name`: examples `brlan0`, `brlan1`, `br-home`. | No explicit name capacities, NUL rule or validation. Examples are not an exhaustive whitelist. | `int`: 0 success; -1 failure. No differentiated absence/error reasons. | No output allocation declared; pointer retention/release unspecified. B: `checkIfExistsInBridge`. |
| `int HandlePreConfigVendor(bridgeDetails *bridgeInfo,int InstanceNumber);` | OEM/SoC settings before update. `[in] bridgeInfo`: complete bridge information. `[in] InstanceNumber`: configuration instance. | Cast `enum Config` to int; only values enabled by build conditions are declared. Struct capacities below. | `int`: 0 success; -1 failure. | Not an initialization API. Failure continuation and side-effect cleanup unspecified. B: `HandlePreConfigVendor`. |
| `int HandlePostConfigVendor(bridgeDetails *bridgeInfo,int Config);` | OEM/SoC settings after update. `[in] bridgeInfo`: complete bridge information. `[in] Config`: configuration instance. | Cast `enum Config` to int; conditional values below. | `int`: 0 success; -1 failure. | Not a shutdown/rollback API. Whether required after failed update is unspecified. B: `HandlePostConfigVendor`. |
| `char *getVendorIfaces();` | Retrieve vendor-specific interface names for bridge management. No parameters documented. | Header explicitly says zero-terminated vendor interface-name string; capacity, list separators and empty-string meaning unspecified. In C, the declaration's `()` is not a `(void)` prototype. | `char *`: interface-name character array/string; NULL means no interfaces. No separate failure indication is documented. | Allocation origin, static versus allocated storage, mutability, lifetime, invalidation, caller release and matching allocator unspecified. Do not infer `free()` responsibility. B: `getVendorIfaces`. |

No `RETURN_OK` / `RETURN_ERR` macros or boolean-returning function are declared in these headers. N contains an orphan comment about adding a transaction with `trans_uuid` and boolean success, but no corresponding function declaration; it is not an API.

## Public data contract

### Structures and fields

Two struct tags and their same-named typedefs are declared. Capacities below are array storage sizes, not evidence that all string APIs accept those capacities. Examples are illustrative, not complete legal ranges.

| B: `struct bridgeDetails` / typedef `bridgeDetails` field | Exact type / capacity | Documented meaning and limits |
| --- | --- | --- |
| `bridgeName` | `char[BRIDGE_NAME_SIZE]` = 64 | Bridge name; examples `brlan0`, `brlan1`, `privbr`, `br-home`. |
| `vlan_name` | `char[BRIDGE_NAME_SIZE]` = 64 | VLAN name; examples `vlan1`, `vlan10`, `guest_vlan`. |
| `VirtualParentIfname` | `char[BRIDGE_NAME_SIZE]` = 64 | Virtual parent name; examples `eth0`, `eth1`, `wan0`. |
| `vlanID` | `int` | VLAN ID; examples 1, 2, 3, 10, 100. No complete numeric range or special-value semantics. |
| `ethIfList` | `char[IFLIST_SIZE]` = 256 | Ethernet list; examples `eth0`, `eth1`, `eth2`. |
| `MoCAIfList` | `char[IFLIST_SIZE]` = 256 | MoCA list; examples `moca0`, `moca1`. |
| `GreIfList` | `char[IFLIST_SIZE]` = 256 | GRE list; examples `gre0`, `gre1`. |
| `WiFiIfList` | `char[IFLIST_SIZE]` = 256 | WiFi list; examples `wlan0`, `wlan1`. |

B does not specify termination, list separators, initialization defaults, validation, truncation, or unused-byte rules for these fields.

| N: `struct Gateway_Config_Non_Ovs_Bridge` / typedef `Gateway_Config_Non_Ovs_Bridge` field | Exact type / capacity | Documented meaning and limits |
| --- | --- | --- |
| `if_name` | `char[MAX_IF_NAME_SIZE]` = 16 | Network interface name. |
| `inet_addr` | `char[MAX_IP_ADDR_SIZE]` = 16 | Network IP address. |
| `netmask` | `char[MAX_IP_ADDR_SIZE]` = 16 | Network netmask. |
| `gre_remote_inet_addr` | `char[MAX_IP_ADDR_SIZE]` = 16 | GRE remote IP address. |
| `gre_local_inet_addr` | `char[MAX_IP_ADDR_SIZE]` = 16 | GRE local IP address. |
| `parent_ifname` | `char[MAX_IF_NAME_SIZE]` = 16 | Parent network interface name. |
| `parent_bridge` | `char[MAX_BRIDGE_NAME_SIZE]` = 16 | Parent network bridge name. |
| `mtu` | `int` | MTU packet size in bytes; valid range unspecified. |
| `vlan_id` | `int` | VLAN ID; range and sentinel values unspecified. |
| `if_type` | `IF_TYPE` | Network interface type; enumerators below. |
| `if_cmd` | `BR_CMD` | Network interface/bridge command; enumerators below. |

N's size macros explicitly include the terminating NUL. Names therefore have space for at most 15 characters plus NUL; IPv4 textual capacity is 15 characters plus NUL, documented as `xxx.xxx.xxx.xxx`. No IPv6 representation is declared. Validation, truncation, prefix-versus-netmask interpretation beyond the field description, and invalid/failure output rules are unspecified.

All fields are inline arrays or scalar values; neither structure contains pointer fields or declares allocation/release methods. Function-level ownership and retention are not established by this layout.

### Enumerations and typedefs

There are **5 enums** and **33 unconditional enumerators**, plus **4 conditional enumerators**. Values from implicit increments are shown explicitly below for readability.

| Source / enum | Enumerator | Value | Meaning / build condition |
| --- | --- | --- | --- |
| B: `Config` | `PRIVATE_LAN` | 1 | Private LAN |
| B: `Config` | `HOME_SECURITY` | 2 | Home security |
| B: `Config` | `HOTSPOT_2G` | 3 | Hotspot 2G |
| B: `Config` | `HOTSPOT_5G` | 4 | Hotspot 5G |
| B: `Config` | `LOST_N_FOUND` | 6 | Lost and found |
| B: `Config` | `HOTSPOT_SECURE_2G` | 7 | Secure hotspot 2G |
| B: `Config` | `HOTSPOT_SECURE_5G` | 8 | Secure hotspot 5G |
| B: `Config` | `MOCA_ISOLATION` | 9 | MoCA isolation |
| B: `Config` | `MESH_BACKHAUL` | 10 | Mesh backhaul |
| B: `Config` | `ETH_BACKHAUL` | 11 | Ethernet backhaul |
| B: `Config` | `MESH` | 12 | Mesh |
| B: `Config` | `MESH_WIFI_BACKHAUL_2G` | 13 | Mesh WiFi backhaul 2G |
| B: `Config` | `MESH_WIFI_BACKHAUL_5G` | 14 | Mesh WiFi backhaul 5G |
| B: `Config` | `MESH_ONBOARD` | 18 | Mesh onboarding |
| B: `Config` | `MESH_WIFI_ONBOARD_2G` | 19 | Mesh WiFi onboarding 2G |
| B: `Config` | `MANAGE_WIFI_BRIDGE` | 17 | Only if `WIFI_MANAGE_SUPPORTED` defined |
| B: `Config` | `AMENITY_BRIDGE_2G` | 20 | Only if `AMENITIES_NETWORK_ENABLED` defined and `_CBR2_PRODUCT_REQ_` not defined |
| B: `Config` | `AMENITY_BRIDGE_5G` | 21 | Same amenity condition |
| B: `Config` | `AMENITY_BRIDGE_6G` | 22 | Same amenity condition |
| B: `INTERFACE_TYPE` | `IF_BRIDGE_BRIDGEUTIL` | 1 | Bridge |
| B: `INTERFACE_TYPE` | `IF_VLAN_BRIDGEUTIL` | 2 | VLAN |
| B: `INTERFACE_TYPE` | `IF_GRE_BRIDGEUTIL` | 3 | GRE |
| B: `INTERFACE_TYPE` | `IF_MOCA_BRIDGEUTIL` | 4 | MoCA |
| B: `INTERFACE_TYPE` | `IF_WIFI_BRIDGEUTIL` | 5 | WiFi |
| B: `INTERFACE_TYPE` | `IF_ETH_BRIDGEUTIL` | 6 | Ethernet |
| B: `INTERFACE_TYPE` | `IF_OTHER_BRIDGEUTIL` | 7 | Other/unspecified; implicit value |
| B: `BridgeOpr` | `DELETE_BRIDGE` | 0 | Delete bridge |
| B: `BridgeOpr` | `CREATE_BRIDGE` | 1 | Create bridge |
| N: `IF_TYPE` | `OTHER_IF_TYPE` | 0 | Other |
| N: `IF_TYPE` | `BRIDGE_IF_TYPE` | 1 | Bridge; implicit value |
| N: `IF_TYPE` | `ETH_IF_TYPE` | 2 | Ethernet; implicit value |
| N: `IF_TYPE` | `GRE_IF_TYPE` | 3 | GRE; implicit value |
| N: `IF_TYPE` | `VLAN_IF_TYPE` | 4 | VLAN; implicit value |
| N: `BR_CMD` | `IF_UP_CMD` | 0 | Interface up |
| N: `BR_CMD` | `IF_DOWN_CMD` | 1 | Interface down; implicit value |
| N: `BR_CMD` | `IF_DELETE_CMD` | 2 | Interface delete; implicit value |
| N: `BR_CMD` | `BR_REMOVE_CMD` | 3 | Bridge removal; implicit value |

Four typedef names exist: B's `bridgeDetails`, and N's `IF_TYPE`, `BR_CMD`, `Gateway_Config_Non_Ovs_Bridge`. B's `Config`, `INTERFACE_TYPE` and `BridgeOpr` are enum tags, not typedef names. There are **no callback typedefs**.

`BridgeOpr` is not documented as the `Opr` argument domain: that argument explicitly references `OVS_CMD`. Nor is `BR_CMD` a documented substitute. `IF_TYPE` and `INTERFACE_TYPE` have different names, membership and numeric assignments; no mapping is declared.

### Constants, macros, and conditional compilation

There are **14 non-guard macros**: 10 B object-like constants, 1 B function-like logging macro, and 3 N size constants. Two additional macros are include guards.

| Source / symbol | Definition | Contract meaning / caveat |
| --- | --- | --- |
| B: `BRIDGE_UTIL_LOG_FNAME` | `"/rdklogs/logs/bridgeUtils.log"` | Log path constant; macro does not itself open it. |
| B: `GRE_HANDLER_SCRIPT` | `"/etc/utopia/service.d/service_multinet/handle_gre.sh"` | Script path constant, not proof of execution or platform availability. |
| B: `TOTAL_IFLIST_SIZE` | `1024` | Total interface-list size constant; no function capacity parameter binds it. |
| B: `BRIDGE_NAME_SIZE` | `64` | Bridge/name field storage. |
| B: `IFACE_NAME_SIZE` | `64` | Interface-name size constant; no universal API bound declared. |
| B: `IFLIST_SIZE` | `256` | Interface-list field storage. |
| B: `MAX_LOG_BUFF_SIZE` | `1024` | Shared log buffer capacity. |
| B: `TIMESTAMP` | `64` | Timestamp storage allowance, not a time unit or interval. |
| B: `INTERFACE_EXIST` | `0` | Existence success code. |
| B: `INTERFACE_NOT_EXIST` | `-1` | Existence failure code. |
| B: `bridge_util_log(fmt ...)` | Variadic formatting/write macro | Shared-state behavior detailed below; GNU-style named variadic syntax. |
| N: `MAX_IF_NAME_SIZE` | `16` | Includes terminating NUL. |
| N: `MAX_IP_ADDR_SIZE` | `16` | Dotted IPv4 text storage including NUL. |
| N: `MAX_BRIDGE_NAME_SIZE` | `16` | Includes terminating NUL. |
| B: `_BRIDGE_UTIL_OEM_H` | Defined without replacement text | Include guard. |
| N: `_NETWORK_INTERFACE_H_` | Defined without replacement text | Include guard. |

Build-control symbols are tested, not defined, in B: `WIFI_MANAGE_SUPPORTED`, `AMENITIES_NETWORK_ENABLED`, `_CBR2_PRODUCT_REQ_`. Their only visible contract effect here is the conditional `Config` members above; base values remain explicitly assigned. No API is conditionally declared. Neither header provides an `extern "C"` wrapper or a public version macro; C++ linkage strategy and ABI version negotiation are unspecified.

### Externally visible globals

B declares **15 extern objects**; N declares **none**. Declarations are not definitions: integration must supply definitions/linkage and agree initialization, access and synchronization. B itself includes a TODO to make the module self-contained and remove dependencies on variables from other modules.

| Exact declaration in B | Documented meaning / integration implication |
| --- | --- |
| `extern int DeviceMode;` | Router = 0, bridge = 2; other values unspecified. |
| `extern int MocaIsolation_Val;` | MoCA isolation value; domain unspecified. |
| `extern int need_wifi_gw_refresh;` | WiFi gateway refresh flag; numeric convention unspecified. |
| `extern int need_switch_gw_refresh;` | Switch gateway refresh flag; numeric convention unspecified. |
| `extern int syncMembers;` | Synchronization purpose; protocol unspecified. |
| `extern int BridgeOprInPropgress;` | Bridge-operation-in-progress indicator; spelling is part of the declared symbol. |
| `extern FILE *logFp;` | Logging stream; who opens/closes it is unspecified. |
| `extern char log_buff[MAX_LOG_BUFF_SIZE];` | Shared 1024-byte formatting buffer. |
| `extern char log_msg_wtime[MAX_LOG_BUFF_SIZE+TIMESTAMP];` | Shared 1088-byte timestamp/message buffer. |
| `extern char primaryBridgeName[64];` | Primary bridge-name storage; termination/ownership protocol unspecified. |
| `extern int PORT2ENABLE;` | Port status; valid values unspecified. |
| `extern int ethWanEnabled;` | Ethernet WAN status; valid values unspecified. |
| `extern char ethWanIfaceName[64];` | Ethernet WAN interface-name storage; termination unspecified. |
| `extern struct tm *timeinfo;` | Shared time conversion pointer, assigned by logging macro. |
| `extern time_t utc_time;` | Shared time value, assigned by logging macro. |

No global-access locking, volatile/atomic protocol, process-sharing arrangement, or object lifetime is declared. Inclusion of `<pthread.h>` is not evidence of implemented locking.

## Buffer, representation, and logging behavior

### Names and configuration data

B uses 64-byte name arrays and 256-byte member lists; its external name arrays are also 64 bytes. N uses 16-byte name and IPv4-text arrays including NUL. B's `TOTAL_IFLIST_SIZE` is 1024, not the size of the four list fields.

These capacities differ, but the headers do not declare a cross-interface copy or conversion. They are not proven incompatible. Any integration that transfers B data to N needs an owner-defined length-checking/conversion policy rather than assuming lossless copying or silently truncating names. This is a recommendation, not an existing requirement.

Only `updateBridgeInfo`'s interface-name input and `getVendorIfaces`'s result explicitly state termination in B. N's size constants reserve the NUL. The headers do not define general truncation handling, validation, NULL argument acceptance, or output state on errors.

### Logging: comment versus actual macro

B's `bridge_util_log` comment promises a log-file write or stdout fallback. Direct macro inspection instead shows:

1. `snprintf(log_buff, MAX_LOG_BUFF_SIZE-1,fmt)` executes unconditionally.
2. If `logFp == NULL`, nothing is printed; only shared `log_buff` is updated.
3. If `logFp != NULL`, the same formatting expression is evaluated again, then `utc_time = time(NULL)` and `timeinfo = gmtime(&utc_time)`.
4. The macro formats a UTC-looking timestamp from `timeinfo` into `log_msg_wtime`, calls `fprintf(logFp,"%s", log_msg_wtime)`, then `fflush(logFp)`.
5. It does not check the time conversion pointer, formatting results, write results or flush result. It does not append an explicit newline.

The formatting size arguments are 1023 and 1087 for buffers of 1024 and 1088 bytes. Under normal `snprintf` semantics, nonnegative successful formatting leaves at most 1022 and 1086 characters respectively plus a terminating NUL. Truncation is not reported by this macro; formatting errors are unchecked. The final array byte is outside the supplied formatting capacity.

Format arguments/expressions with side effects are evaluated once when `logFp` is NULL and twice when non-NULL. Thus side-effectful arguments can change behavior; this is verified in the macro expansion, not merely a theoretical vendor risk.

The macro mutates shared global buffers/time state with no lock, uses `gmtime` storage without a declared synchronization strategy, and dereferences `timeinfo` without a NULL check. Concurrent/re-entrant safety is not guaranteed. Stream operations and flush may block; no timing bound is declared. No stdout, syslog, severity filtering, or automatic opening of `BRIDGE_UTIL_LOG_FNAME` is implemented by this macro. The braced multi-statement expansion is not a `do ... while (0)` wrapper; call-site control-flow usage requires care.

## Contract gap register

All unresolved items below are **Unspecified by the current public contract.** They require owner decisions, not invented vendor guarantees. IDs are stable references for future clarification.

| ID | Evidence | Impact | Clarification / owner decision required |
| --- | --- | --- | --- |
| BU-G01 | B: `getVendorIfaces` promises NUL-terminated string or NULL for no interfaces. | Consumers cannot reliably parse multiple names or distinguish an empty string from no result/error. | Define separators, encoding, maximum size, empty-string meaning and whether errors can also yield NULL. |
| BU-G02 | B: `getVendorIfaces` returns mutable `char *` with no storage contract or release API. | Risk of leaks, invalid free, stale pointers or accidental shared-storage mutation. | Define allocation origin, ownership, mutability, lifetime, invalidation, concurrency and whether/how callers release storage. |
| BU-G03 | B: pre/update/post comments establish relative ordering only. | Preconditions, partial effects and hardware completion cannot be inferred. | Define readiness, validation, synchronous/asynchronous completion, success semantics and per-stage failure states. |
| BU-G04 | B: `updateBridgeInfo`, `HandlePreConfigVendor`, `HandlePostConfigVendor` have no rollback/recovery contract. | Failed updates may leave unknown state; retry or post-hook may worsen it. | Define retry/idempotency, post-hook after failure, rollback authority and cleanup of caller/vendor resources. |
| BU-G05 | B: `updateBridgeInfo.Opr` references absent `enum OVS_CMD`; `BridgeOpr` and N's `BR_CMD` are separately declared. | No complete portable operation argument domain. | Publish authoritative operation values and mappings; do not substitute similarly named enums without owner approval. |
| BU-G06 | B: `updateBridgeInfo.type` uses `INTERFACE_TYPE`; N uses `IF_TYPE`; sync delete selects `IF_OTHER_BRIDGEUTIL`. | Cross-contract mapping and permitted operation/type combinations are unknown. | Define valid combinations, conversions, invalid-value handling and sync preconditions. |
| BU-G07 | B: `bridge_util_log` comment promises stdout; expansion has no fallback. | Logs disappear when stream is NULL despite documented expectation. | Decide whether comment or macro behavior is intended; specify logging destination/setup and error policy. |
| BU-G08 | B: shared logging globals, unchecked `gmtime`, duplicated formatting. | Races, re-entry hazards, conversion failure and repeated side effects. | Define serialization/re-entry policy, safe timestamp handling and argument-evaluation expectations. |
| BU-G09 | B: 64/256-byte arrays; N: 16-byte NUL-inclusive arrays; no conversion API. | Integrators may truncate names or assume nonexistent validation rules. | Specify per-API capacities, list grammar, NUL rules, validation/truncation policy and any actual cross-interface conversion. |
| BU-G10 | B: pointer inputs and `removeIfaceFromList` input annotation despite removal operation. | Unclear mutability, retained storage and outputs after failure. | Define writable capacity, matching/token behavior, nullability, copying/retention and failure-output guarantees. |
| BU-G11 | B: extern state and all function declarations lack concurrency/timing guarantees. | Integration cannot assume thread/process safety or bounded completion. | Assign global definitions/initialization, locking, process model, re-entry and blocking/timeout requirements. |
| BU-G12 | N: configuration fields and orphan transaction comment, but no functions. | No declared way to apply/query the data or correlate transactions. | Identify the consuming interface externally or explicitly document this as data-only; clarify MTU/VLAN ranges and command semantics. |
| BU-G13 | Existing specification names both `libhalbridge_utils.so` and `libbridge_utils.so`; headers declare neither. | Build/link integration is ambiguous. | Confirm canonical library artifact/link dependency and supported ABI/version policy. |
| BU-G14 | B: 0/-1 status comments and pointer/void APIs. | Callers cannot distinguish not-found, invalid input, internal failure or partial failure consistently. | Define differentiated diagnostics if required, without imposing integer return conventions on pointer/void APIs. |

## Retained integration and quality guidance

This section preserves useful expectations from the previous specification as **legacy guidance**, rather than presenting them as header-proven behavior.

### Runtime, threading, memory, and persistence

The previous specification expected synchronous calls and timeouts appropriate to operation complexity, but supplied no numeric limits. The headers do not establish hardware-completion or blocking guarantees; timeout targets remain an owner decision (BU-G03, BU-G11).

The previous specification did not require thread safety and asked callers to serialize access. Treat serialization as prudent integration guidance, especially for the logging macro, not proof that any function is safe under a particular lock. Its reference to “MTA HAL” was unrelated to this Bridge Utility interface. Multiple-process expectations and vendor internal-thread cleanup likewise lack a declared protocol.

Callers should manage storage they allocate and vendors should manage their internal allocations without leaks or corruption. These general principles do not settle `getVendorIfaces` ownership or authorize freeing its result (BU-G02). No initialization, closure, or shutdown API is declared; the hooks must not be relabeled as such.

The previous specification assigned no power-management or persistence requirement. No dedicated power/persistence API is declared here; absence is not evidence of vendor-side behavior. Memory footprint, CPU budget and performance targets remain unquantified.

### Logging and quality control

Legacy guidance asks implementations to record errors and useful diagnostic information, prefer consistent logging, and use suitable static, licensing/security and runtime memory tools (examples: Coverity, Black Duck, Valgrind). It expects no memory leaks or corruption.

The earlier text suggested printf/syslog, `/var/tmp/` or `/rdklogs/logs/` output, and severity categories FATAL, ERROR, WARNING, NOTICE, INFO, DEBUG, TRACE. Those are suggestions, not logging facilities or standard-level mappings established by these headers. The actual public path constant is `/rdklogs/logs/bridgeUtils.log`; the actual macro behavior and discrepancy are documented above.

### Build, licensing, versioning, and customization

Legacy guidance targets a Linux Yocto environment and a shared library, but conflicts between `libhalbridge_utils.so` and `libbridge_utils.so`; neither name is confirmed by the headers (BU-G13). Consumers include the relevant public headers; correct linkage requires integration confirmation. The old explanation that N avoids an OVS build-time dependency is contextual intent, not evidence that every platform uses OVS.

The previous specification expects Apache License 2.0 implementations and proposes interface evolution using [Semantic Versioning 2.0.0](https://semver.org/) with vendor version agreements. Both inspected headers carry Apache License 2.0 notices; no interface-version constant or runtime negotiation API is declared. License obligations must be checked against repository licensing, not inferred solely from this prose.

There is no dedicated WAN/product customization function. Public globals such as `ethWanEnabled`, `ethWanIfaceName`, and `PORT2ENABLE`, and conditional `Config` values, nevertheless expose integration/product variability. Their detailed policy remains vendor-dependent.

## Inventory reconciliation and verification boundary

| Category | B | N | Total |
| --- | --- | --- | --- |
| Public functions | 7 | 0 | 7 |
| Structs / fields | 1 / 8 | 1 / 11 | 2 / 19 |
| Enums | 3 | 2 | 5 |
| Enumerators | 24 unconditional + 4 conditional | 9 | 33 unconditional + 4 conditional |
| Typedef names | 1 | 3 | 4 |
| Callback typedefs | 0 | 0 | 0 |
| Non-guard macros | 11 | 3 | 14 |
| Include-guard macros | 1 | 1 | 2 |
| Extern objects | 15 | 0 | 15 |

The inventory includes both headers' complete declarations, conditional members, guards and externally visible objects. Standard-library declarations imported by B's includes are not HAL API declarations. No implementation, DUT run, hardware readiness, platform mechanism, or vendor conformance has been verified. Mermaid diagrams use declared function names and deliberately distinguish conceptual relationships and unspecified failure behavior.
