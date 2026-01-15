# Wallet Integration

## Overview

Connect wallets to Savitri Network applications.

## Supported Wallets

- MetaMask
- WalletConnect
- Hardware wallets
- Mobile wallets

## Integration Steps

1. Detect wallet
2. Request connection
3. Get accounts
4. Sign transactions
5. Handle events

## Example

```javascript
if (window.ethereum) {
  await window.ethereum.request({ method: 'eth_requestAccounts' });
  const accounts = await window.ethereum.request({ method: 'eth_accounts' });
}
```

---

*Integrate wallet connectivity into your application.*


