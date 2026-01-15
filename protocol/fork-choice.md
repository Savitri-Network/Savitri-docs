# Fork Choice Rule

## Overview

This document describes the fork choice algorithm used to select the canonical chain in Savitri Network.

## Fork Choice Algorithm

### Selection Rule

```
1. Select longest finalized chain
2. If multiple chains with same length:
   a. Select chain with most certificates
   b. If tie: Select chain with highest validator support
```

### Finality Priority

Finalized blocks always preferred over unfinalized blocks.

### Chain Comparison

Compare chains by:
1. Finality status
2. Chain length
3. Certificate count
4. Validator support

---

*The fork choice rule ensures deterministic chain selection and prevents forks.*


