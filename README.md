
### Step-by-Step Guide to Configure and Run the Script

#### Step 1: Configure Dogecoin Core

1. **Create/Edit `dogecoin.conf`**:
   - Open your `dogecoin.conf` file located at `E:\Dogecoin\dogecoin.conf`.

   Example `dogecoin.conf` file:
   ```ini
   rpcuser=yourrpcusername
   rpcpassword=yourrpcpassword
   rpcport=22555
   rpcallowip=127.0.0.1
   server=1
   ```

2. **Start Dogecoin Core with Configuration**:
   - Open a command prompt and navigate to the directory where `dogecoin-qt.exe` is located:
     ```sh
     cd E:\DogeCore\Dogecoin
     ```
   - Start Dogecoin Core:
     ```sh
     dogecoin-qt.exe -conf=E:\Dogecoin\dogecoin.conf
     ```

#### Step 2: Prepare the Python Script

1. **Install Python and Required Libraries**:
   - Ensure Python is installed on your machine. If not, download and install it from the [official Python website](https://www.python.org/downloads/).
   - Install the `requests` and `python-dotenv` libraries:
     ```sh
     pip install requests python-dotenv
     ```

2. **Create the JSON File**:
   - Create a JSON file (e.g., `txids.json`) in the same directory as your Python script. This file should contain the transaction IDs and target addresses.

   Example `txids.json`:
   ```json
   {
       "txids": [
           "your_txid_1",
           "your_txid_2",
           "your_txid_3"
       ],
       "target_addresses": [
           "D6K8aym7xs3eXBdMJm9Uj3pRcbXgV8zZSo",
           "DMwWH9GsGy2HQ4iujYpFdPiSv3p6V12Gyj",
           "D7VFRZvcQ3xLbgpSL2gmsGh3cWZxLLpv3D"
       ]
   }
   ```

3. **Set Environment Variables**:
   - Create a `.env` file in the same directory as your script to store your environment variables securely.

   Example `.env` file:
   ```ini
   RPC_USER=yourrpcusername
   RPC_PASSWORD=yourrpcpassword
   RPC_HOST=127.0.0.1
   RPC_PORT=22555
   ```

4. **Create the Python Script**:
   - Create a file named `send_utxos.py` and add the following code:

   ```python
   import json
   import requests
   import os
   import time
   from dotenv import load_dotenv

   # Load environment variables from .env file
   load_dotenv()

   # Configuration
   RPC_USER = os.getenv('RPC_USER')
   RPC_PASSWORD = os.getenv('RPC_PASSWORD')
   RPC_PORT = os.getenv('RPC_PORT', '22555')
   RPC_HOST = os.getenv('RPC_HOST', '127.0.0.1')
   RPC_URL = f'http://{RPC_USER}:{RPC_PASSWORD}@{RPC_HOST}:{RPC_PORT}/'
   DUST_THRESHOLD = 0.01  # Define your dust threshold in DOGE

   def list_unspent():
       payload = {
           "jsonrpc": "1.0",
           "id": "listunspent",
           "method": "listunspent",
           "params": [1, 9999999]  # [minconf, maxconf]
       }
       try:
           response = requests.post(RPC_URL, json=payload)
           response.raise_for_status()
           return response.json().get('result', [])
       except requests.exceptions.RequestException as e:
           print(f"Error listing unspent transactions: {e}")
           return []

   def get_utxos_for_txids(txids):
       utxos = list_unspent()
       return [utxo for utxo in utxos if utxo['txid'] in txids]

   def send_utxos_to_addresses(utxos, target_addresses):
       total_amount = sum(utxo['amount'] for utxo in utxos)
       inputs = [{"txid": utxo['txid'], "vout": utxo['vout']} for utxo in utxos]
       outputs = {addr: total_amount / len(target_addresses) for addr in target_addresses}

       try:
           payload = {
               "jsonrpc": "1.0",
               "id": "createrawtransaction",
               "method": "createrawtransaction",
               "params": [inputs, outputs]
           }
           response = requests.post(RPC_URL, json=payload)
           response.raise_for_status()
           raw_tx = response.json().get('result', '')

           payload = {
               "jsonrpc": "1.0",
               "id": "signrawtransactionwithwallet",
               "method": "signrawtransactionwithwallet",
               "params": [raw_tx]
           }
           response = requests.post(RPC_URL, json=payload)
           response.raise_for_status()
           signed_tx = response.json().get('result', {}).get('hex', '')

           payload = {
               "jsonrpc": "1.0",
               "id": "sendrawtransaction",
               "method": "sendrawtransaction",
               "params": [signed_tx]
           }
           response = requests.post(RPC_URL, json=payload)
           response.raise_for_status()
           return response.json().get('result', '')
       except requests.exceptions.RequestException as e:
           print(f"Error sending transaction: {e}")
           return None

   def verify_transaction(txid):
       payload = {
           "jsonrpc": "1.0",
           "id": "gettransaction",
           "method": "gettransaction",
           "params": [txid]
       }
       try:
           response = requests.post(RPC_URL, json=payload)
           response.raise_for_status()
           result = response.json().get('result', {})
           confirmations = result.get('confirmations', 0)
           return confirmations > 0
       except requests.exceptions.RequestException as e:
           print(f"Error verifying transaction: {e}")
           return False

   def main():
       with open('txids.json') as f:
           data = json.load(f)

       txids = data.get('txids', [])
       target_addresses = data.get('target_addresses', [])

       if not txids or not target_addresses:
           print("No transaction IDs or target addresses found in txids.json.")
           return

       utxos = get_utxos_for_txids(txids)
       if not utxos:
           print("No UTXOs found for the given transaction IDs.")
           return

       txid = send_utxos_to_addresses(utxos, target_addresses)
       if txid:
           print(f"Transaction sent with TXID: {txid}")
           print("Waiting for confirmation...")
           while not verify_transaction(txid):
               print("Transaction not yet confirmed. Checking again in 60 seconds...")
               time.sleep(60)
           print("Transaction confirmed.")
       else:
           print("Failed to send transaction.")

   if __name__ == "__main__":
       main()
   ```

### Running the Script

1. **Ensure Dogecoin Core is Running**:
   - Make sure Dogecoin Core is running and fully synchronized with the network.
   - You can start Dogecoin Core using the command:
     ```sh
     dogecoin-qt.exe -conf=E:\Dogecoin\dogecoin.conf
     ```

2. **Run the Python Script**:
   - Ensure the `txids.json` file is in the same directory as the `send_utxos.py` script.
   - Open a terminal or command prompt and navigate to the directory where your script is located.
   - Run the script:
     ```sh
     python send_utxos.py
     ```

This setup will allow your script to interact with the Dogecoin Core node, read UTXOs from the node, create and sign transactions, and wait for confirmations. Adjust the `txids.json` file and other configurations as needed.
