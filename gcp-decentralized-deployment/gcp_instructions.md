# Guide to Deploying a Decentralized Multi-Node Ethereum Network on GCP

This guide provides the steps to set up a two-node, decentralized JMDT testnet using the Kurtosis package on Google Cloud Platform (GCP).

### **Step 1: Set up GCP VMs**

1.  **Create two GCP VM Instances** (e.g., `e2-standard-4` or larger is recommended for a validator node). Use a standard Linux distribution like Ubuntu 22.04 LTS.
2.  **Assign Static IP Addresses:** For each VM, go to "VPC network" -> "IP addresses" and reserve the ephemeral IP addresses to make them static. Note these public IPs down. You'll need them for the configuration files.
    *   `SERVER_A_PUBLIC_IP`
    *   `SERVER_B_PUBLIC_IP`

### **Step 2: Configure GCP Firewall Rules**

You need to allow the nodes to communicate with each other over the internet.

1.  Go to "VPC network" -> "Firewall" and create a new firewall rule.
2.  **Name:** `ethereum-p2p`
3.  **Targets:** Apply it to your specific VMs using tags (e.g., `ethereum-node`) or apply to all instances in the network.
4.  **Source IP ranges:** `0.0.0.0/0` (allows any node to connect)
5.  **Specified protocols and ports:**
    *   Select **TCP** and enter `30303, 9000, 32000-34000`
    *   Select **UDP** and enter `30303, 9000, 32000-34000`
6.  Click **Create**. This rule allows the Execution and Consensus clients to discover and peer with each other.

### **Step 3: Install Docker and Kurtosis on Both VMs**

SSH into each VM and run the following commands:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Log out and log back in for the group change to take effect

# Install Kurtosis
curl -fsSL https://github.com/kurtosis-tech/kurtosis/releases/latest/download/install-kurtosis-cli.sh | bash
```

### **Step 4: Deploy the Bootnode (Server A)**

1.  **Copy your project files** to Server A.
2.  **Update `server_a_params.yaml`:** Replace the placeholder `YOUR_SERVER_A_PUBLIC_IP` with the actual static public IP of Server A.
3.  **Start the bootnode:**
    ```bash
    kurtosis run . --args-file server_a_params.yaml
    ```
4.  **Retrieve the Bootnode's Connection Details:** Once the service is running, you need to get its enode (for Geth) and ENR (for Lighthouse).
    *   **Get Geth enode:**
        ```bash
        # The RPC port will be 32003 with this config
        curl -X POST -H "Content-Type: application/json" \
          --data '{"jsonrpc":"2.0","method":"admin_nodeInfo","params":[],"id":1}' \
          http://127.0.0.1:32003 | jq -r .result.enode
        ```
    *   **Get Lighthouse ENR:**
        ```bash
        # The CL port will be 33001 with this config
        curl http://127.0.0.1:33001/eth/v1/node/identity | jq -r .data.enr
        ```
5.  **Keep these two values.** You will need them for Server B's configuration.

### **Step 5: Deploy the Peer Node (Server B)**

1.  **Copy your project files** to Server B.
2.  **Update `server_b_params.yaml`:**
    *   Replace `YOUR_SERVER_B_PUBLIC_IP` with the public IP of Server B.
    *   Replace `ENODE_OF_SERVER_A` with the full enode URL you retrieved from Server A.
    *   Replace `ENR_OF_SERVER_A` with the ENR string you retrieved from Server A.
3.  **Start the peer node:**
    ```bash
    kurtosis run . --args-file server_b_params.yaml
    ```

### **Step 6: Verify the Connection**

After a few minutes, the nodes should discover each other and peer. You can verify this by checking the peer count on either node.

On Server B, run:
```bash
# Check Geth peer count (should be 1 or more)
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}' \
  http://127.0.0.1:32003

# Check Lighthouse peer count (should be 1 or more)
curl http://127.0.0.1:33001/eth/v1/node/peer_count | jq
```

If the peer counts are greater than 0, your decentralized network is running successfully!
