# Guide to Deploying a 5-Node Decentralized Ethereum Network on GCP

This guide provides the steps to set up a five-node, decentralized JMDT testnet using the Kurtosis package on Google Cloud Platform (GCP).

### **Step 1: Set up 5 GCP VMs**

1.  **Create five GCP VM Instances** (e.g., `e2-standard-4` or larger). Use a standard Linux distribution like Ubuntu 22.04 LTS.
2.  **Assign Static IP Addresses:** For each of the five VMs, reserve their ephemeral IP addresses to make them static. Note these public IPs down.
    *   `SERVER_A_PUBLIC_IP` (This will be your bootnode)
    *   `SERVER_B_PUBLIC_IP`
    *   `SERVER_C_PUBLIC_IP`
    *   `SERVER_D_PUBLIC_IP`
    *   `SERVER_E_PUBLIC_IP`

### **Step 2: Configure GCP Firewall Rules**

You can use the same firewall rule as the 2-node setup. Ensure it's applied to all five of your new VMs (e.g., by using a common network tag like `ethereum-node-5`).

*   **Rule Name:** `ethereum-p2p`
*   **Source IP ranges:** `0.0.0.0/0`
*   **Protocols and ports:**
    *   **TCP:** `30303, 9000, 32000-34000`
    *   **UDP:** `30303, 9000, 32000-34000`

### **Step 3: Install Docker and Kurtosis on All 5 VMs**

SSH into each of the five VMs and run the following commands:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Log out and log back in for the group change to take effect

# Install Kurtosis
echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
sudo apt update
sudo apt install kurtosis-cli
bash
```

### **Step 4: Deploy the Bootnode (Server A)**

1.  **Copy your project files** to Server A.
2.  **Update `5_nodes_server_a_params.yaml`:** Replace `YOUR_SERVER_A_PUBLIC_IP` with the static public IP of Server A.
3.  **Start the bootnode:**
    ```bash
    kurtosis run . --args-file gcp-decentralized-deployment/5_nodes_server_a_params.yaml
    ```
4.  **Retrieve the Bootnode's Connection Details (enode and ENR):**
    *   **Get Geth enode:**
        ```bash
        curl -X POST -H "Content-Type: application/json" \
          --data '{"jsonrpc":"2.0","method":"admin_nodeInfo","params":[],"id":1}' \
          http://127.0.0.1:32003 | jq -r .result.enode
        ```
    *   **Get Lighthouse ENR:**
        ```bash
        curl http://127.0.0.1:33001/eth/v1/node/identity | jq -r .data.enr
        ```
5.  **Copy these two values.** You will need them for the other four servers.

### **Step 5: Deploy the 4 Peer Nodes (Servers B, C, D, E)**

For each of the remaining four servers:

1.  **Copy your project files** to the server.
2.  **Choose the correct config file** (`5_nodes_server_b_params.yaml`, `5_nodes_server_c_params.yaml`, etc.).
3.  **Update the file:**
    *   Replace `YOUR_SERVER_X_PUBLIC_IP` (e.g., `YOUR_SERVER_B_PUBLIC_IP`) with the public IP of the current server.
    *   Replace `ENODE_OF_SERVER_A` with the enode you retrieved from Server A.
    *   Replace `ENR_OF_SERVER_A` with the ENR you retrieved from Server A.
4.  **Start the peer node:**
    ```bash
    # Example for Server B
    kurtosis run . --args-file gcp-decentralized-deployment/5_nodes_server_b_params.yaml
    ```
    Repeat for Servers C, D, and E with their respective config files.

### **Step 6: Verify the Connections**

After a few minutes, all nodes should be peered. You can check the peer count on any of the peer nodes (e.g., Server E).

```bash
# Check Geth peer count (should eventually be 4)
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}' \
  http://127.0.0.1:32003

# Check Lighthouse peer count (should eventually be 4)
curl http://127.0.0.1:33001/eth/v1/node/peer_count | jq
```

If the peer counts are showing connections, your 5-node decentralized network is running successfully!
