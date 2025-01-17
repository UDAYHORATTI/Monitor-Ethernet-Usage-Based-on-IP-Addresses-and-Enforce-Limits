# Monitor-Ethernet-Usage-Based-on-IP-Addresses-and-Enforce-Limits
This script monitors traffic on the local network based on IP addresses, calculates data usage, and blocks devices exceeding the limit.
import subprocess
import time
import threading

# Configuration
USAGE_LIMIT_MB = 500  # Limit in MB
CHECK_INTERVAL = 60  # Check interval in seconds
DEVICES = {
    "192.168.1.101": {"name": "Device1", "usage": 0, "blocked": False},
    "192.168.1.102": {"name": "Device2", "usage": 0, "blocked": False},
}

# Function to block an IP address
def block_ip(ip_address):
    if not DEVICES[ip_address]["blocked"]:
        try:
            subprocess.run(["iptables", "-A", "OUTPUT", "-s", ip_address, "-j", "DROP"], check=True)
            subprocess.run(["iptables", "-A", "INPUT", "-s", ip_address, "-j", "DROP"], check=True)
            DEVICES[ip_address]["blocked"] = True
            print(f"Blocked IP: {ip_address}")
        except subprocess.CalledProcessError as e:
            print(f"Error blocking IP {ip_address}: {e}")

# Function to unblock an IP address
def unblock_ip(ip_address):
    if DEVICES[ip_address]["blocked"]:
        try:
            subprocess.run(["iptables", "-D", "OUTPUT", "-s", ip_address, "-j", "DROP"], check=True)
            subprocess.run(["iptables", "-D", "INPUT", "-s", ip_address, "-j", "DROP"], check=True)
            DEVICES[ip_address]["blocked"] = False
            DEVICES[ip_address]["usage"] = 0
            print(f"Unblocked IP: {ip_address}")
        except subprocess.CalledProcessError as e:
            print(f"Error unblocking IP {ip_address}: {e}")

# Function to track network traffic usage for each device
def track_usage():
    while True:
        time.sleep(CHECK_INTERVAL)
        for ip, data in DEVICES.items():
            # Use `iptables` to check for incoming and outgoing traffic for the device
            cmd_in = f"iptables -L -v -n -x | grep {ip} | grep ACCEPT | grep -oP '\\d+(?= bytes)'"
            cmd_out = f"iptables -L -v -n -x | grep {ip} | grep ACCEPT | grep -oP '\\d+(?= bytes)'"

            # Calculate the total incoming and outgoing traffic for the device
            in_traffic = subprocess.getoutput(cmd_in)
            out_traffic = subprocess.getoutput(cmd_out)

            if in_traffic and out_traffic:
                in_bytes = int(in_traffic)
                out_bytes = int(out_traffic)
                total_bytes = in_bytes + out_bytes

                DEVICES[ip]["usage"] = total_bytes
                usage_mb = total_bytes / (1024 * 1024)  # Convert to MB
                print(f"IP: {ip}, Usage: {usage_mb:.2f} MB")

                if usage_mb > USAGE_LIMIT_MB and not data["blocked"]:
                    block_ip(ip)

# Main function
def main():
    track_usage()

# Start tracking and limit enforcement in a background thread
if __name__ == "__main__":
    monitor_thread = threading.Thread(target=main, daemon=True)
    monitor_thread.start()

    # Keep the script running
    while True:
        time.sleep(1)
