# Hathor Integration Update Request

## Introduction

We’re currently in the testing phase of a project called Feature Activation, which is a mechanism used to activate new consensus-agreed features on Hathor Network. In the future, it will be used to release important features such as Nano Contracts and increasing the maximum Merkle Path length, which will allow miners to find more valid merge mined blocks. Here’s a [blog post](https://blog.hathor.network/hathors-feature-activation-process-miners-lead-the-way-d7f43b12978d) about it.

In order for that to work, we need miners to propagate a support signal for features through their mined blocks. This is simply a repurposed byte in the block data. We’ll soon have a complete guide on how to interact with feature signals, but everything should work by default and transparently. However, at this time, we kindly request your cooperation.

It’s likely that you need to update your integration with Hathor to correctly propagate those signals. The block template that you currently use has a new field called `signal_bits`. This field is an integer and represents a single byte. You simply have to set this as the first byte in the Hathor blocks you generate, and everything should work. Before, the `version` field occupied the first 2 bytes. Now, the `signal_bits` is the first byte, and the `version` field is the second byte.

We of course are available for any questions or help that you might need. Thank you in advance!

## Guide: Updating the Integration

### Summary

The first byte of a Hathor block was reserved for future use and ignored. Now, this byte is being used for bit signaling of Feature Activation, starting from [hathor-core v0.59.0](https://github.com/HathorNetwork/hathor-core/releases/tag/v0.59.0) on mainnet. In order to support this new mechanism, a simple change must be made in miner integration with Hathor. This is a one-time change and after it, everything
  will work automatically.

#### What is Feature Activation?

Feature Activation is a mechanism for upgrading consensus rules on Hathor Network while preventing forks from happening. It's inspired by [Bitcoin's BIP 8](https://github.com/bitcoin/bips/blob/master/bip-0008.mediawiki). Read the [blog post](https://blog.hathor.network/hathors-feature-activation-process-miners-lead-the-way-d7f43b12978d) for a general overview, or the [RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md) for internal technical details (out of the scope for this guide).

#### What is Bit Signaling?

In order to activate an upgrade in the network, a super majority of miners must signal that their full node is up-to-date and ready to accept the change. This is signaled though bits in the first byte of mined blocks, which was previously unused. Miners will have the flexibility to decide whether they want to support a new change or not, but by default no action is necessary and everything will work automatically. Internal technical details can be found in the [Feature Activation RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#bit-signaling-implementation) and the [Bit Signaling RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0002-bit-signaling.md), but are out of scope for this guide.

#### What should I do?

Here are the necessary steps:

1. Make sure hathor-core is updated to the [latest version](https://github.com/HathorNetwork/hathor-core/releases).
2. Nothing has to be changed in the way the full node is operated. Everything will work automatically.
3. Verify that Hathor blocks are being generated with the correct first byte. If not, update the integration to support it. Read the examples below for a detailed explanation.

### 1. Identifying where the Hathor block is generated

Your current integration most certainly reads Block templates from one of our HTTP or WebSocket endpoints. The relevant fields provided in those endpoints are the following:

```python
versions: set[int]
signal_bits: int
reward: int
weight: float
timestamp_now: int
timestamp_min: int
timestamp_max: int
parents: list[bytes]
parents_any: list[bytes]
height: int
score: float
```

Identify where your integration code reads this data and constructs the block.

### 2. Verify that your generated block includes `signal_bits`

For reference, you can use the [websocat](https://github.com/vi/websocat) tool to get a block template from our WebSocket API by running `$ websocat wss://node1.mainnet.hathor.network/v1a/mining_ws`. Then, you'll get:

```json
{
  "id": null,
  "method": "mining.notify",
  "params": [
    {
      "data": "0400010000032000000040516f9397b2cc0d66205cfb03000000000000000001b1f30acdddd6f519fed153ad4bc4f107cde88a6176022a000054281a233cace0503977300affcd6fd72f4aa92c35da826d40c49b786ef200000a04a8cc7097f5701c810d4c5234ad85acf81f6609e2c4e680519f52ed3e00",
      "versions": [
        0,
        3
      ],
      "reward": 800,
      "weight": 69.74338333569195,
      "timestamp_now": 1713397058,
      "timestamp_min": 1713396987,
      "timestamp_max": 1713397358,
      "parents": [
        "000000000000000001b1f30acdddd6f519fed153ad4bc4f107cde88a6176022a",
        "000054281a233cace0503977300affcd6fd72f4aa92c35da826d40c49b786ef2",
        "00000a04a8cc7097f5701c810d4c5234ad85acf81f6609e2c4e680519f52ed3e"
      ],
      "parents_any": [],
      "height": 4404735,
      "score": 88.38130312675703,
      "signal_bits": 4
    }
  ]
}
```

It's also possible that you use the HTTP API instead, from `GET /v1a/get_block_template`. While the returned schema is a bit different, the relevant fields are the same.

Using the example from above, notice that `"versions": [0, 3]` and `"signal_bits": 4`. The `versions` field includes the possible block versions to be used, where `0` is a Regular Block and `3` is a Merge Mined Block, which is likely what you are using. Both `versions` and `signal_bits` values represent a single byte each. That is, considering this example, the following block would be generated:

- 1st byte: `0x04` (the value from `signal_bits`).
- 2nd byte: `0x03` (the chosen value, `3`, from `versions`).
- Then, the rest of the block bytes.

Or, as a string of bytes: `[0x04, 0x03, ...]`.

Check that the blocks generated by your integration are correctly assigning the `signal_bits` in the first byte. Before this update, the first byte would always be zero, `0x00`.

> [!NOTE]
> Every previous version of the full node ignores the value of the first byte, so using the `signal_bits` will not affect the propagation of blocks to peers running nodes with outdated versions.

### 3. What if my generated block does not include `signal_bits`?

It's likely that your integration ignores the `signal_bits` field, which didn't exist before. You probably set the first byte of the block data as `0x00`, or just use the `version` field to set the first two block bytes (the `version` value is never greater than one byte). You must change this so the `signal_bits` field is the first block byte, and the `version` field is the second block byte.

#### Example

A simple Python script is provided to illustrate how the first bytes can be constructed:


```python
import json

MERGE_MINED_BLOCK_VERSION = 3

json_response_from_ws = '{"id":null,"method":"mining.notify","params":[{"data":"0400010000032000000040516f9397b2cc0d66205cfb03000000000000000001b1f30acdddd6f519fed153ad4bc4f107cde88a6176022a000054281a233cace0503977300affcd6fd72f4aa92c35da826d40c49b786ef200000a04a8cc7097f5701c810d4c5234ad85acf81f6609e2c4e680519f52ed3e00","versions":[0,3],"reward":800,"weight":69.74338333569195,"timestamp_now":1713397058,"timestamp_min":1713396987,"timestamp_max":1713397358,"parents":["000000000000000001b1f30acdddd6f519fed153ad4bc4f107cde88a6176022a","000054281a233cace0503977300affcd6fd72f4aa92c35da826d40c49b786ef2","00000a04a8cc7097f5701c810d4c5234ad85acf81f6609e2c4e680519f52ed3e"],"parents_any":[],"height":4404735,"score":88.38130312675703,"signal_bits":4}]}'
json_dict = json.loads(json_response_from_ws)

fields = json_dict['params'][0]
versions = fields['versions']
signal_bits = fields['signal_bits']

assert MERGE_MINED_BLOCK_VERSION in versions, 'in this example we are generating a merge mined block'

def prepare_first_two_bytes(signal_bits: int, version: int) -> bytes:
    r"""
    Prepare first two bytes for a Hathor block.
    Notice that the return value will always have length equal to two.

    >>> prepare_first_two_bytes(0, 0)
    b'\x00\x00'

    >>> prepare_first_two_bytes(0b101, 0x04)
    b'\x05\x04'

    >>> prepare_first_two_bytes(0b1111_0001, 0xFF)
    b'\xf1\xff'

    >>> prepare_first_two_bytes(0xFFF, 0x00)
    Traceback (most recent call last):
     ...
    AssertionError: the signal_bits must be at most one byte
    """
    assert signal_bits <= 0xFF, 'the signal_bits must be at most one byte'
    assert version <= 0xFF, 'the version must be at most one byte'
    return bytes([signal_bits, version])

first_two_bytes = prepare_first_two_bytes(signal_bits, MERGE_MINED_BLOCK_VERSION)
assert first_two_bytes == b'\x04\x03'

# ...Then you would go on to configure the rest of the block bytes.
```

> [!IMPORTANT]
> Depending on when you run this example, it's possible that you get a different value for `signal_bits` from the Mining API. It might be that you receive `"signal_bits": 0`, which would result in the first byte being `0x00`, just like before this update. Make sure that you're correctly assigning `signal_bits` to the first block byte, event if it's zero.

### 3. Done!

Make this change so all your blocks contain this new first byte. No further change is necessary.
