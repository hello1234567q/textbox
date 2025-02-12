#!/bin/bash


apt-get update -y > /dev/null 2>&1


echo "📥 Downloading Tiny10 B4 x64..."
wget --progress=bar:force -O win10.iso "https://archive.org/download/tiny-10-b-2/Tiny10%20B4%20x64.iso" 2>&1 | grep -o '[0-9]*%' | tail -n1

RDP_ADDRESS=""


echo "🔍 Checking for Cloudflare Tunnel..."
if [ -f "/usr/local/bin/cloudflared" ]; then
    echo "✅ Cloudflare Tunnel found!"
    echo "🔗 Please log in to Cloudflare to set up your tunnel."
    cloudflared tunnel login
    read -p "Enter your Cloudflare Tunnel ID: " CLOUDFLARE_TUNNEL_ID
    cloudflared tunnel route ip add 0.0.0.0/0 $CLOUDFLARE_TUNNEL_ID
    cloudflared tunnel run $CLOUDFLARE_TUNNEL_ID &

    
    sleep 5
    TUNNEL_URL=$(curl --silent --show-error http://127.0.0.1:4040/api/tunnels | sed -nE 's/.*public_url":"tcp:..([^"]*).*/\1/p')
    RDP_ADDRESS="Cloudflare Tunnel: $TUNNEL_URL"

else
    echo "❌ No Cloudflare Tunnel found. Switching to DuckDNS..."

    
    if [ ! -f "duckdns_token.txt" ]; then
        echo "⚠️ You need a DuckDNS token. Visit https://www.duckdns.org/ to create one."
        read -p "Enter your DuckDNS token: " DUCKDNS_TOKEN
        echo "$DUCKDNS_TOKEN" > duckdns_token.txt
    else
        DUCKDNS_TOKEN=$(cat duckdns_token.txt)
    fi

    
    read -p "Enter your desired DuckDNS subdomain (e.g., myrdp): " DUCKDNS_SUBDOMAIN
    echo "🔗 Registering subdomain $DUCKDNS_SUBDOMAIN.duckdns.org..."
  
    wget -qO- "https://www.duckdns.org/update?domains=$DUCKDNS_SUBDOMAIN&token=$DUCKDNS_TOKEN&ip=" > /dev/null
    RDP_ADDRESS="DuckDNS: $DUCKDNS_SUBDOMAIN.duckdns.org:3389"
fi


echo "🖥 Installing QEMU..."
apt-get install qemu-system-x86 -y > /dev/null 2>&1


echo "🚀 Starting Windows Tiny10..."
qemu-system-x86_64 -hda win10.iso -m 4G -smp 4 -net user,hostfwd=tcp::3389-:3389 -net nic -vga vmware -nographic &>/dev/null &


echo "===================================="
echo "🖥 Your Windows RDP is ready!"
echo "🔑 Username: Administrator"
echo "🔑 Password: No password required"
echo "===================================="
echo "📌 Use Remote Desktop (RDP) to connect:"
echo "🔗 $RDP_ADDRESS"
echo "===================================="
