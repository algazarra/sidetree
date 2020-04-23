
## File Structures

The protocol defines the following three file structures, which house DID operation data and are designed to support key functionality to enable light node configurations, minimize permanently retained data, and ensure performant resolution of DIDs.

<img src="../docs/diagrams/file-topology.svg" style="display: block; margin: 0 auto; padding: 2em 0; width: 100%; max-width: 620px;" />

### Anchor File

Anchor Files contain [Create](#create), [Recover](#recover), and [Deactivate](#deactivate) operation values, as well as a CAS URI for the related Sidetree Map file (detailed below). As the name suggests, Anchor Files are anchored to the target ledger system via embedding a CAS URI in the ledger's transactional history.

::: example
```json
{
  "map_file_uri": CAS_URI,
  "operations": {
    "create": [
      {
        "suffix_data": { // Base64URL encoded
          "delta_hash": DELTA_HASH,
          "recovery_key": JWK_OBJECT,
          "recovery_commitment": COMMITMENT_HASH
        }
      },
      {...}
    ],
    "recover": [
      {
        "did_suffix": SUFFIX_STRING,
        "recovery_reveal_value": REVEAL_VALUE,
        "signed_data": { // Base64URL encoded, compact JWS
            "protected": {...},
            "payload": {
                "recovery_commitment": COMMITMENT_HASH,
                "delta_hash": DELTA_HASH,
                "recovery_key": JWK_OBJECT
            },
            "signature": SIGNATURE_STRING
        }
      },
      {...}
    ],
    "deactivate": [
      {
        "did_suffix": SUFFIX_STRING,
        "recovery_reveal_value": REVEAL_VALUE,
        "signed_data": { // Base64URL encoded, compact JWS
            "protected": {...},
            "payload": {
              "did_suffix": SUFFIX_STRING,
              "recovery_reveal_value": REVEAL_VALUE
            },
            "signature": SIGNATURE_STRING
        }
      },
      {...}
    ]
  }
}
```
:::

A valid [Anchor File](#anchor-file) is a JSON document that ****MUST NOT**** exceed the [`MAX_ANCHOR_FILE_SIZE`](#max-anchor-file-size), and composed as follows:

1. The [Anchor File](#anchor-file) ****MUST**** contain a [`map_file_uri`](#map-file-property){id="map-file-property"} property, and its value ****MUST**** be a _CAS URI_ for the related Map File.
2. If the set of operations to be anchored contain any [Create](#create), [Recover](#recovery), or [Deactivate](#deactivate) operations, the [Anchor File](#anchor-file) ****MUST**** contain an `operations` property, and its value ****MUST**** be an object composed as follows:
    - If there are any [Create](#create) operations to be included in the Anchor File:
      1. The `operations` object ****MUST**** include a `create` property, and its value ****MUST**** be an array.
      2. For each [Create](#create) operation to be included in the `create` array, herein referred to as [_Anchor File Create Entries_](#anchor-file-create-entry){id="anchor-file-create-entry"}, use the following process to compose and include a JSON object for each entry:
          - Each entry object must contain a `suffix_data` property, and its value ****MUST**** be a [_Create Operation Suffix Data Object_](#create-suffix-data-object).
      3. The [Anchor File](#anchor-file) ****MUST NOT**** include multiple [Create](#create) operations that produce the same [DID Suffix](#did-suffix).
    - If there are any [Recovery](#recover) operations to be included in the Anchor File:
      1. The `operations` object ****MUST**** include a `recover` property, and its value ****MUST**** be an array.
      2. For each [Recovery](#recover) operation to be included in the `recover` array, herein referred to as [_Anchor File Recovery Entries_](#anchor-file-recovery-entry){id="anchor-file-recovery-entry"}, use the following process to compose and include entries:
          - The object ****MUST**** contain a `did_suffix` property, and its value ****MUST**** be the [DID Suffix](#did-suffix) of the DID the operation pertains to. An [Anchor File](#anchor-file) ****MUST NOT**** contain more than one operation of any type with the same [DID Suffix](#did-suffix).
          - The object ****MUST**** contain a `recovery_reveal_value` property, and its value ****MUST**** be the last recovery [`COMMITMENT_VALUE`](#commitment-value).
          - The object ****MUST**** contain a `signed_data` property, and its value ****MUST**** be a [_Recovery Operation Signed Data Object_](#recovery-signed-data-object).
    - If there are any [Deactivate](#deactivate) operations to be included in the Anchor File:
      1. The `operations` object ****MUST**** include a `deactivate` property, and its value ****MUST**** be an array.
      2. For each [Deactivate](#deactivate) operation to be included in the `deactivate` array, use the following process to compose and include entries:
          - The object ****MUST**** contain a `did_suffix` property, and its value ****MUST**** be the [DID Suffix](#did-suffix) of the DID the operation pertains to. An [Anchor File](#anchor-file) ****MUST NOT**** contain more than one operation of any type with the same [DID Suffix](#did-suffix).
          - The object ****MUST**** contain a `recovery_reveal_value` property, and its value ****MUST**** be the last recovery [`COMMITMENT_VALUE`](#commitment-value).
          - The object ****MUST**** contain a `signed_data` property, and its value ****MUST**** be a [_Deactivate Operation Signed Data Object_](#deactivate-signed-data-object).

### Map File

The Map file in the Sidetree protocol contains Update operation proving data, as well as the CAS-linked Chunk file chunks.
::: example
```json
{
  "chunks": [
    { "chunk_file_uri": CHUNK_HASH },
    {...}
  ],
  "operations": {
    "update": [
      {
        "did_suffix": DID_SUFFIX,
        "update_reveal_value": REVEAL_VALUE,
        "signed_data": { // Base64URL encoded, compact JWS
            "protected": {...},
            "payload": {
              "delta_hash": DELTA_HASH
            },
            "signature": SIGNATURE_STRING
        }   
      },
      {...}
    ]
  }
}
```
:::

A valid [Map File](#map-file) is a JSON document that ****MUST NOT**** exceed the [`MAX_MAP_FILE_SIZE`](#max-map-file-size), and composed as follows:

1. The [Anchor File](#anchor-file) ****MUST**** contain a `chunks` property, and its value ****MUST**** be an array of _Chunk Entries_ for the related delta data for a given chunk of operations in the batch. Future versions of the protocol will specify a process for separating the operations in a batch into multiple _Chunk Entries_, but for this version of the protocol there ****MUST**** be only one _Chunk Entry_ present in the array. _Chunk Entry_ objects are composed as follows:
    1. The _Chunk Entry_ object ****MUST**** contain a `chunk_file_uri` property, and its value ****MUST**** be a URI representing the corresponding CAS file entry, generated via the [`CID_ALGORITHM`](#cid-algorithm).
2. If there are any [Update](#update) operations to be included in the Map File, the [Map File](#map-file) ****MUST**** include an `operations` property, and its value ****MUST**** be an object composed as follows:
  1. The `operations` object ****MUST**** include an `update` property, and its value ****MUST**** be an array.
  2. For each [Update](#update) operation to be included in the `update` array, herein referred to as [Map File Update Entries](#map-file-update-entry){id="map-file-update-entry"}, use the following process to compose and include entries:
        - The object ****MUST**** contain an `did_suffix` property, and its value ****MUST**** be the [DID Suffix](#did-suffix) of the DID the operation pertains to.
        - The object ****MUST**** contain a `update_reveal_value` property, and its value ****MUST**** be the last update [`COMMITMENT_VALUE`](#commitment-value).
        - The object ****MUST**** contain a `signed_data` property, and its value ****MUST**** be an [_Update Operation Signed Data Object_](#update-signed-data-object).

### Chunk Files

Chunk Files are JSON Documents, compressed via the [COMPRESSION_ALGORITHM](#compression-algorithm) contain Sidetree Operation source data, which are composed of delta-based CRDT entries that modify the state of a Sidetree identifier's DID Document.

For this version of the protocol, there will only exist a single Chunk File that contains all the state modifying data for all operations in the included set. Future versions of the protocol will separate the total set of included operations into multiple chunks, each with their own Chunk File.

::: example Create operation Chunk File entry
```json
{
  "deltas": [
       
    { // JSON.stringify()-ed, and Base64URL encoded
      "patches": PATCH_ARRAY,
      "update_commitment": COMMITMENT_HASH
    },
    ...
  ]
}
```
:::

In this version of the protocol, Chunk Files are constructed as follows:

1. The Chunk File ****MUST**** include a `deltas` property, and its value ****MUST**** be an array containing [_Chunk File Delta Entry_](#chunk-file-delta-entry){id="chunk-file-delta-entry"} objects.
2. Each [_Chunk File Delta Entry_](#chunk-file-delta-entry) ****MUST**** be a `Base64URL` encoded object, assembled as follows:
    1. The object ****MUST**** contain a `patches` property, and its value ****MUST**** be an array of [DID State Patches](#did-state-patches).
    2. The payload ****MUST**** contain an `update_commitment` property, and its value ****MUST**** be the next _Update Commitment_ generated during the operation process associated with the type of operation being performed.

3. Each [_Chunk File Delta Entry_](#chunk-file-delta-entry) ****MUST**** be appended to the `deltas` array as follows, in this order:
    1. If any Create operations were present in the associated Anchor File, append all [_Create Operation Delta Objects_](#create-delta-object) in the same index order as their matching [_Anchor File Create Entry_](#anchor-file-create-entry).
    2. If any Recovery operations were present in the associated Anchor File, append all [_Recovery Operation Delta Objects_](#recovery-delta-object) in the same index order as their matching [_Anchor File Recovery Entry_](#anchor-file-recovery-entry).
    3. If any Update operations were present in the associated Map File, append all [_Update Operation Delta Objects_](#update-delta-object) in the same index order as their matching [_Map File Update Entry_](#map-file-update-entry).
