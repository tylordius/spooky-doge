# Spooky Doge Wallet - dApp Integration Guide

This guide explains how to integrate your dApp with the Spooky Doge wallet to enable Dogecoin payments and Doginal (inscription) transfers.

## Platform Support

Spooky Doge wallet is available on:
- **Browser Extension** - Chrome/Edge extension
- **Mobile App** - iOS/Android with in-app dApp browser

**Both platforms inject the same `window.dogecoin` provider**, so you can support all users with a single codebase.

## Quick Start

### 1. Detect the Wallet

```javascript
// Check if Spooky Doge wallet is available
function isSpookyWalletAvailable() {
  return typeof window.dogecoin !== 'undefined' && window.dogecoin.isSpookyWallet;
}

// Wait for wallet to be ready (it injects after page load)
window.addEventListener('dogecoin#initialized', () => {
  console.log('Spooky Doge wallet detected!');
});
```

### 2. Connect to the Wallet

```javascript
async function connectWallet() {
  if (!isSpookyWalletAvailable()) {
    alert('Please install Spooky Doge wallet or open in the Spooky Doge app browser');
    return null;
  }

  try {
    // Opens approval popup for user
    const result = await window.dogecoin.connect();
    console.log('Connected! Address:', result.address);
    return result.address;
  } catch (error) {
    console.error('Connection rejected:', error.message);
    return null;
  }
}
```

### 3. Get Account Information

```javascript
// Get connected address
const address = await window.dogecoin.getAddress();

// Get wallet balance in koinu (1 DOGE = 100,000,000 koinu)
const balance = await window.dogecoin.getBalance();
const dogeBalance = balance / 100000000;

// Check if currently connected
const connected = window.dogecoin.isConnected();

// Get all connected accounts
const accounts = await window.dogecoin.getAccounts();
```

### 4. Send DOGE

```javascript
async function sendPayment(recipientAddress, amountInDoge) {
  try {
    // Opens transaction approval popup
    const result = await window.dogecoin.sendTransaction({
      to: recipientAddress,
      amount: amountInDoge  // Amount in DOGE (not koinu)
    });
    
    console.log('Transaction sent! TXID:', result.txid);
    return result.txid;
  } catch (error) {
    console.error('Transaction failed:', error.message);
    throw error;
  }
}
```

### 5. Work with Doginals (Inscriptions)

```javascript
// Get all doginals in the wallet
const doginals = await window.dogecoin.getDoginals();
doginals.forEach(d => {
  console.log('Inscription:', d.inscriptionId);
  console.log('Image URL:', d.imageUrl);
  console.log('Content Type:', d.contentType);
});

// Transfer a doginal
async function transferDoginal(inscriptionId, recipientAddress) {
  try {
    const result = await window.dogecoin.sendDoginal({
      inscriptionId: inscriptionId,
      to: recipientAddress
    });
    console.log('Doginal transferred! TXID:', result.txid);
    return result.txid;
  } catch (error) {
    console.error('Transfer failed:', error.message);
    throw error;
  }
}
```

### 6. Sign Messages

```javascript
// Request a message signature (no cost, requires user approval)
async function signMessage(message) {
  try {
    const result = await window.dogecoin.signMessage(message);
    console.log('Signature:', result.signature);
    console.log('Signed by:', result.address);
    return result;
  } catch (error) {
    console.error('Signing failed:', error.message);
    throw error;
  }
}

// Example: Verify wallet ownership for login
async function verifyWalletOwnership() {
  const address = await window.dogecoin.getAddress();
  const nonce = 'Login to MyDapp at ' + new Date().toISOString();
  
  try {
    const { signature } = await window.dogecoin.signMessage(nonce);
    
    // Send to your backend for verification
    const response = await fetch('/api/verify-signature', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ address, message: nonce, signature })
    });
    
    return response.ok;
  } catch (error) {
    console.error('Verification failed:', error);
    return false;
  }
}
```

### 7. Listen for Events

```javascript
// Account changes (user switches address)
window.dogecoin.on('accountsChanged', (accounts) => {
  if (accounts.length === 0) {
    console.log('Wallet disconnected');
  } else {
    console.log('Active account:', accounts[0]);
  }
});

// Connection events
window.dogecoin.on('connect', ({ address }) => {
  console.log('Connected:', address);
});

window.dogecoin.on('disconnect', () => {
  console.log('Wallet disconnected');
});
```

## API Reference

### Provider Object

Available at `window.dogecoin` or `window.spookyWallet`

| Property | Type | Description |
|----------|------|-------------|
| `isSpookyWallet` | `boolean` | Always `true` - identifies the wallet |
| `isConnected()` | `function` | Returns `true` if site is connected |

### Methods

#### `connect()`
Request wallet connection. Opens approval popup.

**Returns:** `Promise<{ address: string }>`

---

#### `disconnect()`
Disconnect the dApp from the wallet.

**Returns:** `Promise<void>`

---

#### `getAddress()`
Get the currently connected address.

**Returns:** `Promise<string | null>` - Null if not connected

---

#### `getAccounts()`
Get all connected accounts.

**Returns:** `Promise<string[]>`

---

#### `getBalance()`
Get wallet balance in koinu.

**Returns:** `Promise<number>` - Balance in koinu (divide by 100,000,000 for DOGE)

---

#### `getDoginals()`
Get all doginals (inscriptions) in the wallet.

**Returns:** `Promise<Doginal[]>`

```typescript
interface Doginal {
  inscriptionId: string;  // Unique inscription identifier
  output: string;         // UTXO reference (txid:vout)
  contentType: string;    // MIME type (e.g., 'image/png', 'text/html')
  imageUrl: string;       // CDN URL for the content
  value?: number;         // UTXO value in DOGE (optional)
}
```

---

#### `sendTransaction(params)`
Request a DOGE transaction. Opens approval popup.

**Parameters:**
```typescript
{
  to: string;      // Recipient Dogecoin address
  amount: number;  // Amount in DOGE (not koinu)
}
```

**Returns:** `Promise<{ txid: string }>`

---

#### `sendDoginal(params)`
Request a doginal transfer. Opens approval popup.

**Parameters:**
```typescript
{
  inscriptionId: string;  // The inscription ID to transfer
  to: string;             // Recipient Dogecoin address
}
```

**Returns:** `Promise<{ txid: string }>`

---

#### `signMessage(message)`
Request a message signature. Opens approval popup. **Signing is free (no DOGE cost).**

**Parameters:**
- `message` (string) - The message to sign

**Returns:** `Promise<{ signature: string, address: string }>`

The signature is in Bitcoin/Dogecoin signed message format (base64-encoded, 65 bytes), compatible with `verifymessage` in Dogecoin Core.

---

#### `request(args)`
EIP-1193 style request method for compatibility.

```javascript
// Examples
const accounts = await window.dogecoin.request({ method: 'doge_requestAccounts' });
const balance = await window.dogecoin.request({ method: 'doge_getBalance' });
const doginals = await window.dogecoin.request({ method: 'doge_getDoginals' });
const result = await window.dogecoin.request({
  method: 'doge_sendTransaction',
  params: { to: 'DAddress...', amount: 10 }
});
```

**Supported Methods:**

| Method | Description |
|--------|-------------|
| `doge_requestAccounts` | Request wallet connection |
| `doge_accounts` | Get connected accounts |
| `doge_getBalance` | Get wallet balance |
| `doge_getDoginals` | Get doginals in wallet |
| `doge_sendTransaction` | Send DOGE |
| `doge_sendDoginal` | Transfer a doginal |
| `doge_signMessage` | Sign a message (free, no DOGE cost) |
| `doge_chainId` | Get chain identifier (`"dogecoin:mainnet"`) |

---

#### `on(event, callback)`
Subscribe to wallet events.

**Events:**
- `connect` - Fired when connected, receives `{ address: string }`
- `disconnect` - Fired when disconnected
- `accountsChanged` - Fired when active address changes, receives `string[]`

---

#### `removeListener(event, callback)`
Unsubscribe from wallet events.

## Fee Structure

| Transaction Type | Network Fee | Dev Fee | Total |
|------------------|-------------|---------|-------|
| Regular DOGE send | Dynamic (based on tx size) | 0.01 DOGE | Variable |
| Doginal transfer | 0.10 DOGE | 0.01 DOGE | 0.11 DOGE |
| Multi-doginal transfer | 0.10 DOGE x count | 0.01 DOGE | Variable |

## Complete Example: Payment Page

```html
<!DOCTYPE html>
<html>
<head>
  <title>My Doge Shop</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: -apple-system, sans-serif; padding: 20px; background: #1a1a2e; color: #fff; }
    button { padding: 12px 24px; margin: 8px; border: none; border-radius: 8px; background: #9333ea; color: white; font-size: 16px; cursor: pointer; }
    button:disabled { opacity: 0.5; }
    .connected { color: #22c55e; }
    .disconnected { color: #ef4444; }
  </style>
</head>
<body>
  <h1>My Doge Shop</h1>
  
  <div id="wallet-status" class="disconnected">Wallet not connected</div>
  <div id="wallet-address"></div>
  <div id="wallet-balance"></div>
  
  <button id="connect-btn" onclick="handleConnect()">Connect Wallet</button>
  <button id="disconnect-btn" onclick="handleDisconnect()" style="display:none">Disconnect</button>
  
  <hr>
  
  <h2>Buy a Widget - 10 DOGE</h2>
  <button id="buy-btn" onclick="handlePurchase()" disabled>Buy Now</button>
  <div id="tx-result"></div>

  <script>
    const SHOP_ADDRESS = 'DShopAddressHere123456789';
    const WIDGET_PRICE = 10;

    function isSpookyWalletAvailable() {
      return typeof window.dogecoin !== 'undefined' && window.dogecoin.isSpookyWallet;
    }

    function init() {
      if (isSpookyWalletAvailable()) {
        setupWallet();
      }
    }

    window.addEventListener('dogecoin#initialized', () => {
      setupWallet();
    });

    function setupWallet() {
      if (window.dogecoin.isConnected()) {
        updateUI(true);
        refreshBalance();
      }
      
      window.dogecoin.on('accountsChanged', (accounts) => {
        if (accounts.length > 0) {
          updateUI(true);
          document.getElementById('wallet-address').textContent = 'Address: ' + accounts[0];
          refreshBalance();
        } else {
          updateUI(false);
        }
      });
    }

    async function handleConnect() {
      if (!isSpookyWalletAvailable()) {
        alert('Please install Spooky Doge wallet or open in the app browser');
        return;
      }

      try {
        const result = await window.dogecoin.connect();
        document.getElementById('wallet-address').textContent = 'Address: ' + result.address;
        updateUI(true);
        await refreshBalance();
      } catch (error) {
        alert('Connection failed: ' + error.message);
      }
    }

    async function handleDisconnect() {
      await window.dogecoin.disconnect();
      updateUI(false);
    }

    async function refreshBalance() {
      const balance = await window.dogecoin.getBalance();
      const dogeBalance = (balance / 100000000).toFixed(2);
      document.getElementById('wallet-balance').textContent = 'Balance: ' + dogeBalance + ' DOGE';
    }

    function updateUI(connected) {
      document.getElementById('wallet-status').textContent = connected ? 'Wallet Connected' : 'Wallet not connected';
      document.getElementById('wallet-status').className = connected ? 'connected' : 'disconnected';
      document.getElementById('connect-btn').style.display = connected ? 'none' : 'inline';
      document.getElementById('disconnect-btn').style.display = connected ? 'inline' : 'none';
      document.getElementById('buy-btn').disabled = !connected;
      if (!connected) {
        document.getElementById('wallet-address').textContent = '';
        document.getElementById('wallet-balance').textContent = '';
      }
    }

    async function handlePurchase() {
      const resultDiv = document.getElementById('tx-result');
      resultDiv.textContent = 'Processing payment...';
      
      try {
        const result = await window.dogecoin.sendTransaction({
          to: SHOP_ADDRESS,
          amount: WIDGET_PRICE
        });
        
        resultDiv.innerHTML = 'Payment successful!<br>TXID: ' + result.txid;
        await refreshBalance();
      } catch (error) {
        resultDiv.textContent = 'Payment failed: ' + error.message;
      }
    }

    init();
  </script>
</body>
</html>
```

## Complete Example: Doginal Marketplace

```html
<!DOCTYPE html>
<html>
<head>
  <title>Doginal Marketplace</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: -apple-system, sans-serif; padding: 20px; background: #1a1a2e; color: #fff; }
    button { padding: 12px 24px; margin: 8px; border: none; border-radius: 8px; background: #9333ea; color: white; font-size: 16px; cursor: pointer; }
    button:disabled { opacity: 0.5; }
    .doginal-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(120px, 1fr)); gap: 12px; margin-top: 20px; }
    .doginal-card { background: #2a2a4a; border-radius: 12px; padding: 12px; text-align: center; cursor: pointer; }
    .doginal-card img { width: 100%; border-radius: 8px; }
    .doginal-card.selected { border: 2px solid #9333ea; }
  </style>
</head>
<body>
  <h1>Your Doginals</h1>
  
  <div id="wallet-status">Not connected</div>
  <button id="connect-btn" onclick="handleConnect()">Connect Wallet</button>
  
  <div id="doginal-grid" class="doginal-grid"></div>
  
  <div style="margin-top: 20px;">
    <input type="text" id="recipient" placeholder="Recipient address" style="padding: 12px; width: 100%; max-width: 400px; border-radius: 8px; border: none;">
    <button id="send-btn" onclick="handleSend()" disabled>Send Selected</button>
  </div>

  <script>
    let selectedDoginal = null;

    function isSpookyWalletAvailable() {
      return typeof window.dogecoin !== 'undefined' && window.dogecoin.isSpookyWallet;
    }

    window.addEventListener('dogecoin#initialized', () => {
      if (window.dogecoin.isConnected()) {
        loadDoginals();
      }
    });

    async function handleConnect() {
      if (!isSpookyWalletAvailable()) {
        alert('Please use Spooky Doge wallet');
        return;
      }

      try {
        const result = await window.dogecoin.connect();
        document.getElementById('wallet-status').textContent = 'Connected: ' + result.address.slice(0, 12) + '...';
        loadDoginals();
      } catch (error) {
        alert('Connection failed: ' + error.message);
      }
    }

    async function loadDoginals() {
      try {
        const doginals = await window.dogecoin.getDoginals();
        const grid = document.getElementById('doginal-grid');
        grid.innerHTML = '';
        
        doginals.forEach(doginal => {
          const card = document.createElement('div');
          card.className = 'doginal-card';
          card.onclick = () => selectDoginal(doginal.inscriptionId, card);
          card.innerHTML = `
            <img src="${doginal.imageUrl}" alt="Doginal" onerror="this.src='data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 width=%22100%22 height=%22100%22><rect fill=%22%239333ea%22 width=%22100%22 height=%22100%22/></svg>'">
            <p style="font-size: 10px; margin-top: 8px;">${doginal.inscriptionId.slice(0, 12)}...</p>
          `;
          grid.appendChild(card);
        });

        if (doginals.length === 0) {
          grid.innerHTML = '<p>No doginals found</p>';
        }
      } catch (error) {
        console.error('Failed to load doginals:', error);
      }
    }

    function selectDoginal(inscriptionId, cardElement) {
      document.querySelectorAll('.doginal-card').forEach(c => c.classList.remove('selected'));
      cardElement.classList.add('selected');
      selectedDoginal = inscriptionId;
      document.getElementById('send-btn').disabled = false;
    }

    async function handleSend() {
      const recipient = document.getElementById('recipient').value.trim();
      if (!selectedDoginal || !recipient) {
        alert('Select a doginal and enter recipient address');
        return;
      }

      try {
        const result = await window.dogecoin.sendDoginal({
          inscriptionId: selectedDoginal,
          to: recipient
        });
        alert('Doginal sent! TXID: ' + result.txid);
        selectedDoginal = null;
        loadDoginals();
      } catch (error) {
        alert('Transfer failed: ' + error.message);
      }
    }
  </script>
</body>
</html>
```

## Error Handling

| Error Message | Cause |
|---------------|-------|
| `"Connection rejected by user"` | User declined connection request |
| `"Site not connected"` | dApp not connected to wallet |
| `"Wallet not unlocked"` | User hasn't unlocked the wallet |
| `"Transaction rejected by user"` | User declined transaction |
| `"Doginal transfer rejected by user"` | User declined doginal transfer |
| `"Message signing rejected by user"` | User declined signature request |
| `"Insufficient funds"` | Not enough DOGE for transaction + fees |
| `"Inscription not found in wallet"` | Requested inscription not in wallet |

## Security Notes

1. **User Approval Required** - All transactions require explicit user approval via popup
2. **Per-Site Permissions** - Each website must be approved separately; users can manage connections in Settings
3. **Non-Custodial** - Private keys never leave the wallet
4. **Inscription Protection** - UTXOs under 0.1 DOGE are protected from regular spending
5. **Dev Fee Transparency** - 0.01 DOGE dev fee included in all transactions

## Platform Notes

### Browser Extension
- Install from Chrome Web Store or load unpacked from `dist-extension` folder
- Provider is injected automatically on all websites

### Mobile App
- Users must open your website through the in-app dApp browser (Browser tab)
- Provider is injected into the WebView automatically
- Ensure your site is mobile-responsive

### Suggesting Users Open in Spooky Doge

```javascript
if (!isSpookyWalletAvailable()) {
  // Show a banner or modal
  showModal({
    title: 'Use Spooky Doge Wallet',
    message: 'For the best experience, open this site in the Spooky Doge app browser.',
    buttons: [
      { text: 'Get the App', url: 'https://spooksociety.xyz/wallet' },
      { text: 'Continue Anyway', action: 'dismiss' }
    ]
  });
}
```

## Support

For integration support or questions:
- Website: https://spooksociety.xyz
- GitHub: https://github.com/spooksociety/spooky-doge
