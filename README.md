const express = require('express');
const axios = require('axios');
const ethers = require('ethers'); // For Worldcoin transactions
const app = express();
app.use(express.json());

// Your Worldcoin address
const YOUR_WORLDCOIN_ADDRESS = '0x9cfc2bd986e313785dbd1841b02a744c314055b8';

// Mock M-Pesa API
const MPESA_API_URL = 'https://api.mpesa.com';
const MPESA_API_KEY = 'your_mpesa_api_key';

// Mock Worldcoin API
const WORLDCOIN_API_URL = 'https://api.worldcoin.org';
const WORLDCOIN_API_KEY = 'your_worldcoin_api_key';

// Worldcoin wallet setup
const provider = new ethers.providers.JsonRpcProvider('https://mainnet.infura.io/v3/YOUR_INFURA_KEY');
const wallet = new ethers.Wallet('YOUR_PRIVATE_KEY', provider);

// Transfer endpoint
app.post('/transfer', async (req, res) => {
  const { userId, direction, amount, mpesaNumber, worldcoinAddress } = req.body;

  try {
    // Step 1: Verify World ID
    const worldIdVerified = await verifyWorldId(userId);
    if (!worldIdVerified) throw new Error('World ID verification failed.');

    // Step 2: Deduct 0.1 WLD fee
    const feeTx = await wallet.sendTransaction({
      to: YOUR_WORLDCOIN_ADDRESS,
      value: ethers.utils.parseEther('0.1'), // 0.1 WLD
    });
    await feeTx.wait();

    // Step 3: Execute transfer
    if (direction === 'mpesa_to_worldcoin') {
      // Deduct from M-Pesa
      await axios.post(`${MPESA_API_URL}/withdraw`, {
        phoneNumber: mpesaNumber,
        amount,
        apiKey: MPESA_API_KEY,
      });

      // Deposit to Worldcoin
      const depositTx = await wallet.sendTransaction({
        to: worldcoinAddress,
        value: ethers.utils.parseEther(amount.toString()),
      });
      await depositTx.wait();
    } else if (direction === 'worldcoin_to_mpesa') {
      // Deduct from Worldcoin
      const withdrawTx = await wallet.sendTransaction({
        to: YOUR_WORLDCOIN_ADDRESS, // Temporary holding
        value: ethers.utils.parseEther(amount.toString()),
      });
      await withdrawTx.wait();

      // Deposit to M-Pesa
      await axios.post(`${MPESA_API_URL}/deposit`, {
        phoneNumber: mpesaNumber,
        amount,
        apiKey: MPESA_API_KEY,
      });
    }

    res.status(200).json({ message: 'Transfer successful!' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// World ID verification (mock)
async function verifyWorldId(userId) {
  // Call World ID API to verify
  return true; // Assume verification succeeds
}

app.listen(3000, () => console.log('Server running on port 3000'));
