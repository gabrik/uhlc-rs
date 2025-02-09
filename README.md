# uhlc-rs

[![build](https://github.com/atolab/uhlc-rs/workflows/build/badge.svg)](https://github.com/atolab/uhlc-rs/actions?query=workflow%3Abuild)
[![crate](https://img.shields.io/crates/v/uhlc.svg)](https://crates.io/crates/uhlc)
[![API](https://img.shields.io/badge/api-latest-green.svg)](https://atolab.github.io/uhlc-rs)

A Unique Hybrid Logical Clock for Rust.

This library is an implementation of an [Hybrid Logical Clock (HLC)](https://cse.buffalo.edu/tech-reports/2014-04.pdf) associated to a unique identifier. Thus, it is able to generate timestamps that are unique across a distributed system, without the need of a centralized time source.

## Usage
Add this to your `Cargo.toml`:

```toml
[dependencies]
uhlc = "0.5"
```

Then in your code:
```rust
use uhlc::*;

// create an HLC with a generated UUID and relying on SystemTime::now()
let hlc = HLC::default();

// generate a timestamp
let ts = hlc.new_timestamp();

// update the HLC with a timestamp incoming from another HLC
if ! hlc.update_with_timestamp(&other_ts).is_ok() {
    println!(r#"The incoming timestamp would make this HLC
             to drift too much. You should refuse it!"#);
}
```

## What is an HLC ?
A Hybrid Logical Clock combines a Physical Clock with a Logical Clock.
It generates monotonic timestamps that are close to the pysical time, but with a
counter part in the last bits that allow to preserve the _"happen before"_ relationship.

You can find more detailled explanations in:
 - This blog: http://sergeiturukin.com/2017/06/26/hybrid-logical-clocks.html
 - The original paper: https://cse.buffalo.edu/tech-reports/2014-04.pdf

## Why "Unique" ?
In this implementation, each HLC instance is associated with an identifier that must be
unique accross the system (by default a UUIDv4). Each generated timestamp, in addition
of the hybrid time, contains the identifier of the HLC that generated it, and it
therefore unique across the system.

Such property allows the ordering all timestamped events in a distributed system, without
the need of a centralized time source or decision.

Note that this ordering preserve the _"happen before"_ relationship only when events can
be correlated. I.e.:

 * if 2 events have the same source, no problem since they will be timestamped by the
   same HLC that will generate 2 ordered timestamps.

 * if an entity receives an event with a timestamp t1, it must update its HLC with t1.
   Thus, all consecutive generated timestamps will be greater than t1.

 * if 2 events have different sources that have not exchanged timestamped events before,
   as the physical clocks on each source might not be synchronized, it may happen that
   the HLCs generate timestamps that don't reflect the real physical ordering.
   But in most cases this doesn't really matter since there is no a real correlation
   between those events (one is not a consequence of the other).

## Implementation details
The `uhlc::HLC::default()` operation generate an UUIDv4 as identifier and uses
`std::time::SystemTime::now()` as physical clock.  
But using the `uhlc::HLCBuilder` allows you to configure the `HLC` differently. Example:  
```Rust
let custom_hlc = HLCBuilder::new()
    .with_id(ID::try_from([0x01, 0x02, 0x03]).unwrap())     // use a custom identifier
    .with_clock(my_custom_gps_clock)                        // use a custom physical clock (e.g. using GPS as time source)
    .with_max_delta(Duration::from_secs(1))                 // use a custom maximum delta (see explanations below)
    .build();

```

A `uhlc::HLC::NTP64` time is 64-bits unsigned integer as specified in
[RFC-5909](https://tools.ietf.org/html/rfc5905#section-6).
The first 32-bits part is the number of second since the EPOCH of the physical clock,
and the second 32-bits part is the fraction of second.
In case its generated by an HLC, the last few bits of the second part are replaced
by the HLC logical counter. The size of this counter currently hard-coded to 4 bits
in `uhlc::CSIZE`.

To avoid a "too fast clock" to make an HLC drift too much in the future, the
`uhlc::HLC::update_with_timestamp(timestamp)` operation will return an error if the
incoming timestamp exceeds the current physical time more than a delta
(100ms by default, configurable declaring the `UHLC_MAX_DELTA_MS` environment variable).
In such case, it could be wise to refuse or drop the incoming event,
since it might not be correctly ordered with further events.

## Usages
**uhlc** is currently used in [Eclipse zenoh](https://github.com/eclipse-zenoh/zenoh).
