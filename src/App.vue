<template>
  <div class="app-container">
    <button @click="openModal" class="connect-button">Open Modal</button>

    <!-- Display account information if connected -->
    <div v-if="isConnected" class="account-info">
      <p><strong>Address:</strong> {{ address }}</p>
      <p><strong>CAIP Address:</strong> {{ caipAddress }}</p>
      <p><strong>Status:</strong> {{ status }}</p>

      <!-- Transaction form -->
      <div class="transaction-form">
        <h3>Send USDT</h3>
        <div class="form-group">
          <label>Recipient Address:</label>
          <input 
            v-model="recipientAddress" 
            type="text" 
            placeholder="0x..."
            class="input-field"
          />
        </div>
        <div class="form-group">
          <label>Amount of USDT:</label>
          <input 
            v-model="amount" 
            type="number" 
            step="0.01" 
            placeholder="0.01"
            class="input-field"
          />
        </div>
        <button 
          @click="sendUSDT" 
          class="send-button"
          :disabled="!isValidTransaction"
        >
          Send USDT
        </button>
        
        <!-- Transaction Status -->
        <div v-if="transactionStatus" :class="['transaction-status', transactionStatus.type]">
          {{ transactionStatus.message }}
        </div>
      </div>
    </div>

    <!-- Connection status if not connected -->
    <div v-else class="connection-status">
      <p>Not connected. Please click "Open Modal" to connect your wallet.</p>
    </div>
  </div>
</template>

<script setup>
import { computed, ref } from 'vue';
import { createAppKit, useAppKit, useAppKitAccount } from '@reown/appkit/vue';
import { EthersAdapter } from '@reown/appkit-adapter-ethers';
import { mainnet, arbitrum } from '@reown/appkit/networks';
import { isAddress, Contract, parseUnits, ethers } from 'ethers';

// USDT Contract ABI - Only the transfer function
const USDT_ABI = [
  {
    constant: false,
    inputs: [
      { name: '_to', type: 'address' },
      { name: '_value', type: 'uint256' },
    ],
    name: 'transfer',
    outputs: [{ name: '', type: 'bool' }],
    type: 'function',
  },
];

// USDT Contract addresses
const USDT_ADDRESSES = {
  ethereum: '0xdAC17F958D2ee523a2206206994597C13D831ec7', // Ethereum Mainnet
  arbitrum: '0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9', // Arbitrum
};

// Project configuration
const projectId = '3242c8fdd637f0a59ed554a4409d60e5';
const metadata = {
  name: 'My Website',
  description: 'My Website description',
  url: 'https://mywebsite.com',
  icons: ['https://avatars.mywebsite.com/'],
};

// Initialize AppKit with ethers adapter
createAppKit({
  adapters: [new EthersAdapter()],
  networks: [mainnet, arbitrum],
  metadata,
  projectId,
  features: { analytics: true },
});

// Setup AppKit and Account
const appKit = useAppKit();
const account = useAppKitAccount();

// State for transaction form
const recipientAddress = ref('');
const amount = ref('0.01');
const transactionStatus = ref(null);

// Computed properties
const address = computed(() => account.value.address);
const isConnected = computed(() => account.value.isConnected);
const caipAddress = computed(() => account.value.caipAddress);
const status = computed(() => account.value.status);

const isValidTransaction = computed(() => {
  return isAddress(recipientAddress.value) && Number(amount.value) > 0;
});

// Function to open connection modal
const openModal = async () => {
  try {
    const result = await appKit.open();
    console.log('Connection successful:', result);
  } catch (error) {
    console.error('Connection failed:', error);
  }
};

// Function to send USDT
const sendUSDT = async () => {
  if (!isValidTransaction.value) {
    transactionStatus.value = {
      type: 'error',
      message: 'Địa chỉ không hợp lệ hoặc số tiền phải lớn hơn 0',
    };
    return;
  }

  try {
    transactionStatus.value = {
      type: 'pending',
      message: 'Đang xử lý giao dịch...',
    };

    const provider = new ethers.BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const network = await provider.getNetwork();
    const chainId = network.chainId;

    const usdtAddress =
      chainId === 42161n ? USDT_ADDRESSES.arbitrum : USDT_ADDRESSES.ethereum;

    const usdtContract = new Contract(usdtAddress, USDT_ABI, signer);
    const amountInUnits = parseUnits(String(amount.value), 6); // Đảm bảo giá trị là chuỗi

    const transaction = await usdtContract.transfer(
      recipientAddress.value,
      amountInUnits
    );

    transactionStatus.value = {
      type: 'pending',
      message: 'Đang chờ xác nhận...',
    };

    await transaction.wait();

    transactionStatus.value = {
      type: 'success',
      message: `Giao dịch thành công! Mã giao dịch: ${transaction.hash}`,
    };

    recipientAddress.value = '';
    amount.value = '0.01';
  } catch (error) {
    console.error('Giao dịch thất bại:', error);
    transactionStatus.value = {
      type: 'error',
      message: `Giao dịch thất bại: ${error.message}`,
    };
  }
};
</script>

<style>
.app-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 50px;
}

.connect-button {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}

.account-info,
.connection-status {
  margin-top: 20px;
  text-align: center;
}

.account-info p,
.connection-status p {
  margin: 5px 0;
}

.transaction-form {
  margin-top: 20px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  width: 100%;
  max-width: 400px;
}

.form-group {
  margin: 10px 0;
  text-align: left;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
}

.input-field {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-bottom: 10px;
}

.send-button {
  width: 100%;
  padding: 10px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.send-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.transaction-status {
  margin-top: 10px;
  padding: 10px;
  border-radius: 4px;
}

.transaction-status.pending {
  background-color: #fff3cd;
  color: #856404;
}

.transaction-status.success {
  background-color: #d4edda;
  color: #155724;
}

.transaction-status.error {
  background-color: #f8d7da;
  color: #721c24;
}
</style>
