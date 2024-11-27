

### **Dockerfiles**

#### **Router (router/Dockerfile)**

```dockerfile
FROM debian:latest

RUN apt-get update && apt-get install -y \
    iproute2 iptables \
    && apt-get clean

CMD ["bash"]
```

---

#### **PC (pc/Dockerfile)**

```dockerfile
FROM debian:latest

RUN apt-get update && apt-get install -y \
    curl iproute2 iptables openssh-server \
    && apt-get clean

RUN mkdir /var/run/sshd && echo 'root:root' | chpasswd

CMD ["/usr/sbin/sshd", "-D"]
```

---

### **Docker Compose File**

#### **docker-compose.yml**

```yaml
version: "3.9"
services:
  router:
    build:
      context: ./router
    container_name: router
    cap_add:
      - NET_ADMIN
    networks:
      subnet1:
        ipv4_address: 192.168.1.1
      subnet2:
        ipv4_address: 192.168.2.1
    command: ["bash"]

  pc0:
    build:
      context: ./pc
    container_name: pc0
    cap_add:
      - NET_ADMIN
    networks:
      subnet1:
        ipv4_address: 192.168.1.2
    command: ["bash"]

  pc1:
    build:
      context: ./pc
    container_name: pc1
    cap_add:
      - NET_ADMIN
    networks:
      subnet2:
        ipv4_address: 192.168.2.2
    command: ["bash"]

  pc2:
    build:
      context: ./pc
    container_name: pc2
    cap_add:
      - NET_ADMIN
    networks:
      subnet2:
        ipv4_address: 192.168.2.3
    command: ["bash"]

networks:
  subnet1:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.1.0/24
  subnet2:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.2.0/24
```

---

### **Steps to Solve Tasks**

#### **Task 1: Initial Setup**

1. **Build and Start Containers**  
   Use the following commands:
   ```bash
   docker-compose up --build -d
   ```

2. **Test Connectivity**  
   From `router`, check connectivity:
   ```bash
   docker exec -it router bash
   ping 192.168.1.2  # PC0 on subnet 1
   ping 192.168.2.2  # PC1 on subnet 2
   ```

3. **Ensure Services are Running**  
   - Set up `pc0` as a web server:
     ```bash
     docker exec -it pc0 bash
     apt-get update && apt-get install -y apache2
     service apache2 start
     ```
   - Verify that PC1 and PC2 can access the web server:
     ```bash
     docker exec -it pc1 bash
     curl 192.168.1.2
     ```

#### **Task 2: Enable Packet Forwarding**

1. Enable forwarding on the router:
   ```bash
   docker exec -it router bash
   sysctl -w net.ipv4.ip_forward=1
   ```

2. **Deface the Webserver (PC1)**  
   SSH into PC0 from PC1 and modify the web server:
   ```bash
   docker exec -it pc1 bash
   ssh root@192.168.1.2  # Password is 'root'
   echo "Defaced!" > /var/www/html/index.html
   ```

#### **Task 3: Block SSH from PC1**

1. Configure the router's firewall to block SSH from PC1:
   ```bash
   docker exec -it router bash
   iptables -A FORWARD -p tcp --dport 22 -s 192.168.2.2 -j DROP
   ```

2. Verify:
   - SSH from PC1 to PC0 fails:
     ```bash
     docker exec -it pc1 bash
     ssh root@192.168.1.2
     ```
   - SSH from PC2 to PC0 succeeds:
     ```bash
     docker exec -it pc2 bash
     ssh root@192.168.1.2
     ```

#### **Task 4: Configure UDP Server and Firewall**

1. Set up PC1 as a UDP server:
   ```bash
   docker exec -it pc1 bash
   apt-get update && apt-get install -y netcat
   nc -lu 5000  # Listen on UDP port 5000
   ```

2. Test UDP Ping:
   ```bash
   echo "Ping" | nc -u -w1 192.168.2.2 5000  # Run from PC2 or PC0
   ```

3. Block UDP from PC2:
   ```bash
   docker exec -it pc1 bash
   iptables -A INPUT -p udp -s 192.168.2.3 -j DROP
   ```

4. Verify:
   - UDP from PC0 succeeds:
     ```bash
     echo "Ping" | nc -u -w1 192.168.2.2 5000
     ```
   - UDP from PC2 fails:
     ```bash
     echo "Ping" | nc -u -w1 192.168.2.2 5000
     ```

---

Here's how to approach the task using Docker containers `PC0` and `PC2`:

---

### **Task Setup**

1. **Launch Containers:**
   Use the `docker-compose.yml` from the previous task and ensure `PC0` and `PC2` are running.
   ```bash
   docker-compose up -d
   ```

2. **Install OpenSSL:**
   Install OpenSSL on both `PC0` and `PC2`:
   ```bash
   docker exec -it pc0 bash
   apt-get update && apt-get install -y openssl
   ```

   Repeat the same for `PC2`.

---

### **Question 1: Encrypt File**

1. **Create a File on PC2:**
   On `PC2`, create a text file with at least 56 bytes:
   ```bash
   docker exec -it pc2 bash
   echo "This is a test file to demonstrate encryption with AES in CTR and OFB modes." > message.txt
   ```

2. **Encrypt the File:**
   Encrypt the file using AES with CTR mode:
   ```bash
   openssl enc -aes-256-ctr -in message.txt -out encrypted_ctr.bin -k secretkey
   ```

   Encrypt the file using AES with OFB mode:
   ```bash
   openssl enc -aes-256-ofb -in message.txt -out encrypted_ofb.bin -k secretkey
   ```

3. **Transfer Files to PC0:**
   Use Docker’s `cp` command or SSH to transfer files:
   ```bash
   docker cp pc2:/encrypted_ctr.bin .
   docker cp encrypted_ctr.bin pc0:/encrypted_ctr.bin

   docker cp pc2:/encrypted_ofb.bin .
   docker cp encrypted_ofb.bin pc0:/encrypted_ofb.bin
   ```

4. **Evaluate Cipher Modes:**
   - **CTR Mode:** Error in ciphertext impacts only the corresponding plaintext block.
   - **OFB Mode:** Error in ciphertext impacts the same bit in all subsequent plaintext blocks.

---

### **Question 2: Corrupted Ciphertext**

1. **Corrupt the Ciphered File:**
   Modify the 6th bit of the ciphered file on `PC0`:
   ```bash
   docker exec -it pc0 bash
   xxd encrypted_ctr.bin | sed '1 s/06/00/' | xxd -r > corrupted_ctr.bin
   xxd encrypted_ofb.bin | sed '1 s/06/00/' | xxd -r > corrupted_ofb.bin
   ```

2. **Verify Corrupted Files:**
   - Verify integrity with OpenSSL:
     ```bash
     openssl enc -aes-256-ctr -d -in corrupted_ctr.bin -out decrypted_ctr.txt -k secretkey
     openssl enc -aes-256-ofb -d -in corrupted_ofb.bin -out decrypted_ofb.txt -k secretkey
     ```
   - Observe the corrupted content in `decrypted_ctr.txt` and `decrypted_ofb.txt`.

---

### **Question 3: Decrypt and Comment**

1. **Decrypt Files:**
   Decrypt the corrupted files:
   ```bash
   openssl enc -aes-256-ctr -d -in corrupted_ctr.bin -out decrypted_ctr.txt -k secretkey
   openssl enc -aes-256-ofb -d -in corrupted_ofb.bin -out decrypted_ofb.txt -k secretkey
   ```

2. **Comment on Error Propagation:**
   - **CTR Mode:**
     - Error affects only the corrupted block.
     - Adjacent plaintext blocks remain intact.
   - **OFB Mode:**
     - Error in a single bit propagates to all subsequent plaintext blocks.
     - Adjacent plaintext blocks are significantly impacted.

---

### **Summary:**
- **CTR Mode:** Better resilience to corruption as errors are localized to the affected block.
- **OFB Mode:** Error propagates to subsequent blocks, making it less robust for error-prone communication channels.
