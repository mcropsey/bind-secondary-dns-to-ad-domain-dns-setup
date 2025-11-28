Here’s the exact, ready-to-use configuration for your RHEL 10 BIND server that will pull **all** the zones shown in your screenshots from your main Windows DC at **192.168.1.10**.

### Zones to replicate (exactly as they appear in your DNS Manager)

**Forward lookup zones**  
- corp.local  
- _msdcs.corp.local  
- apps.ocp.corp.local  

**Reverse lookup zone**  
- 1.168.192.in-addr.arpa   (covers the 192.168.1.0/24 network)

### Complete step-by-step for RHEL 10

#### 1. On the Windows DC (192.168.1.10) – allow zone transfers

For **each** of the four zones above:
1. Open DNS Manager → right-click the zone → Properties → Zone Transfers tab  
2. Check “Allow zone transfers”  
3. Choose “Only to the following servers”  
4. Add the IP of your RHEL server (example: 192.168.1.50)

Do this for:
- corp.local
- _msdcs.corp.local
- apps.ocp.corp.local
- 1.168.192.in-addr.arpa

#### 2. On RHEL 10 – install and configure BIND

```bash
# Install BIND
sudo dnf install -y bind bind-utils

# Enable and start
sudo systemctl enable named
sudo systemctl start named
```

#### 3. Replace /etc/named.conf with this exact content

```bash
options {
    listen-on port 53 { any; };
    listen-on-v6 { none; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { any; };           # adjust if you want to restrict
    recursion yes;
    dnssec-validation no;               # Windows AD zones usually not signed
};

# Your primary Windows DC
masters windows_dc {
    192.168.1.10;
};

# Create slaves directory
# (do this once manually)
# sudo mkdir -p /var/named/slaves
# sudo chown named:named /var/named/slaves
# sudo chmod 770 /var/named/slaves

# === ALL ZONES FROM YOUR SCREENSHOT ===

zone "corp.local" IN {
    type slave;
    file "slaves/corp.local.zone";
    masters { windows_dc; };
};

zone "_msdcs.corp.local" IN {
    type slave;
    file "slaves/_msdcs.corp.local.zone";
    masters { windows_dc; };
};

zone "apps.ocp.corp.local" IN {
    type slave;
    file "slaves/apps.ocp.corp.local.zone";
    masters { windows_dc; };
};

zone "1.168.192.in-addr.arpa" IN {
    type slave;
    file "slaves/1.168.192.in-addr.arpa.zone";
    masters { windows_dc; };
};
```

#### 4. Create the slaves directory and fix permissions

```bash
sudo mkdir -p /var/named/slaves
sudo chown named:named /var/named/slaves
sudo chmod 770 /var/named/slaves
```

#### 5. SELinux and firewall

```bash
# Allow BIND to write transferred zones
sudo setsebool -P named_write_master_zones 1

# Open DNS port
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

#### 6. Validate and restart

```bash
sudo named-checkconf /etc/named.conf
sudo systemctl restart named
```

#### 7. Watch the transfer happen (tail the log)

```bash
sudo journalctl -u named -f
```

You should see within seconds:

```
zone corp.local/IN: Transfer completed: ...
zone _msdcs.corp.local/IN: Transfer completed: ...
zone apps.ocp.corp.local/IN: Transfer completed: ...
zone 1.168.192.in-addr.arpa/IN: Transfer completed: ...
```

#### 8. Test it

```bash
dig @127.0.0.1 dc1.corp.local A
dig @127.0.0.1 apps.ocp.corp.local SOA
dig @127.0.0.1 -x 192.168.1.10      # reverse lookup test
```

That’s it. Your RHEL 10 box is now a fully synchronized secondary DNS server for every single zone you showed in the screenshot, pulling directly from 192.168.1.10. Updates (new hosts, DHCP registrations, etc.) will automatically propagate via IXFR within minutes.
