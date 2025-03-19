# IKEv2-VPN-Service-OpenBSD
VPN Service I have deployed on the cloud. A network and VPN. IKEv2 protocol.

maxxing.club is the DNS

How to make?:


 AWS make an  openbsd ec2 instance prior.

1. **Prepare Your Server:**  
   - **Enable Key Network Settings:**  
     Run these commands to allow VPN traffic:  
     ```bash  
     sysctl net.inet.ip.forwarding=1  
     sysctl net.inet.esp.enable=1  
     sysctl net.inet.ah.enable=1  
     sysctl net.inet.ipcomp.enable=1  
     ```  
     Save these settings in `/etc/sysctl.conf` so they work after reboot.  

2. **Create a VPN Network Interface:**  
   - Create a file `/etc/hostname.enc0` with:  
     ```  
     inet 10.0.1.1 255.255.255.0 10.0.1.255  
     up  
     ```  
   - Reload network: `sh /etc/netstart`.

3. **Set Up Firewall Rules (pf.conf):**  
   - Edit `/etc/pf.conf` to allow VPN traffic and block bad IPs (optional). Example rules:  
     ```  
     intra = "vio0"  # Replace with your internet interface (e.g., vio0)  
     vpn = "enc0"  
     set reassemble yes
     set block-policy return
     set loginterface egress
     set skip on { lo, enc }
     match in all scrub (no-df random-id max-mss 1440)
     table <ossec_fwtable> persist
     table <badguys> persist file "/etc/badguys"
     block in quick  on egress from <badguys> label "bad"
     block out quick on egress to <badguys> label "bad"
     block in quick  on egress from <ossec_fwtable> label "bad"
     block out quick on egress to <ossec_fwtable> label "bad"
     block in quick from urpf-failed label uRPF
     block return log
     pass out all modulate state
     pass in on egress proto { ah, esp }
     pass in on egress proto udp to (egress) port { isakmp, ipsec-nat-t }
     pass out on egress from 10.0.1.0/24 to any nat-to (egress)
     pass out on $intra from 10.0.1.0/24 to $intra:network nat-to ($intra)
     pass in quick inet proto icmp icmp-type { echoreq, unreach }
     pass in on egress proto tcp from any to (egress) port 22 keep state (max-src-conn 40, max-src-conn-rate 10/30, overload <badguys> flush global)
     pass in on $intra proto { udp tcp } from any to ($intra) port 53
     ```  
   - Reload firewall: `pfctl -f /etc/pf.conf`.

4. **Configure the VPN (OpenIKED):**  
   - Generate a **strong Pre-Shared Key (PSK)** (e.g., use a password manager).  
   - Create `/etc/iked.conf` with your PSK and settings:  
     ```  
     ikev2 "inet" passive ipcomp esp \
      from 0.0.0.0/0 to 10.0.1.0/24 \
      from 10.0.0.0/24 to 10.0.1.0/24 \
      local egress peer any \
      psk "{{iked_psk}}" \
      config protected-subnet 0.0.0.0/0 \
      config address 10.0.1.0/24 \
      config name-server 172.31.1.0.2 \ #THIS SHOULD BE YOUR DNS, IT COULD BE WHAT EVER YOU USE IN YOUR OUTBOUND INTERFACE
      tag "IKED" tap enc0
     ```  
   - Restart OpenIKED: `rcctl reload iked`.

5. **Start the VPN Service:**  
   - Enable and start OpenIKED:  
     ```bash  
     rcctl enable iked  
     rcctl start iked  
     ```

6. **Set Up Port Forwarding on Your Router:**  
   - Forward UDP ports **500, 4500, and 1701** to your VPN server’s IP.  
   - Ensure your server has a **public IP or DNS address** (e.g., via your ISP or dynamic DNS).

7. **Connect Clients (e.g., iPhone):**  
   - **iOS/macOS Steps:**  
     1. Go to **Settings > General > VPN > Add VPN Configuration**.  
     2. Choose **IKEv2** type.  
     3. Enter server address (IP/DNS), Remote ID (server’s hostname), and PSK.  
     4. Save and connect!

**Optional Security:**  
- Block suspicious IPs by adding them to `/etc/badguys` (update firewall rules to use this list).  

**Done!** You now have a secure IKEv2 VPN. 
