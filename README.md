# subtool
subnetting tool to help students
import tkinter as tk
from tkinter import ttk, messagebox
import ipaddress

# Function to validate and calculate subnets
def calculate_and_update_grid():
    try:
        # Retrieve and validate input values
        network_address = network_entry.get().strip()
        prefix_length = prefix_entry.get().strip()
        ip_version = ip_version_var.get()  # IPv4 or IPv6

        if not network_address or not prefix_length:
            raise ValueError("Network Address and Prefix Length cannot be empty.")
        prefix_length = int(prefix_length)

        # Validate prefix length for the selected IP version
        if ip_version == "IPv4" and (prefix_length < 1 or prefix_length > 32):
            raise ValueError("IPv4 prefix length must be between 1 and 32.")
        if ip_version == "IPv6" and (prefix_length < 1 or prefix_length > 128):
            raise ValueError("IPv6 prefix length must be between 1 and 128.")

        # Determine the IP version and create the base network
        if ip_version == "IPv4":
            network = ipaddress.ip_network(f"{network_address}/{prefix_length}", strict=False)
        elif ip_version == "IPv6":
            network = ipaddress.IPv6Network(f"{network_address}/{prefix_length}", strict=False)
        else:
            raise ValueError("Invalid IP version selected.")

        clients = []
        for i in range(len(client_names)):
            client_name = client_names[i].get().strip()
            device_count = client_devices[i].get().strip()

            if not client_name or not device_count:
                raise ValueError("Client name and device count cannot be empty.")
            clients.append((client_name, int(device_count)))

        subnets = []

        # Allocate subnets based on device counts
        breakdown_info = []  # To store detailed breakdown information
        for client, device_count in clients:
            try:
                # Calculate the required prefix for the device count
                host_bits_needed = (device_count - 1).bit_length()
                required_prefix = (128 if ip_version == "IPv6" else 32) - host_bits_needed

                # Ensure the required prefix does not exceed the base network's prefix length
                if required_prefix < prefix_length:
                    raise ValueError(
                        f"Not enough address space for {client}. "
                        f"Required prefix ({required_prefix}) is smaller than base network prefix ({prefix_length})."
                    )

                # Generate subnets
                subnets_list = list(network.subnets(new_prefix=required_prefix))
                if subnets_list:
                    allocated_subnet = subnets_list[0]
                    subnets.append((client, allocated_subnet))
                    
                    # Add breakdown details
                    breakdown_info.append({
                        "client": client,
                        "group_size": device_count,
                        "subnet": allocated_subnet,
                        "cidr_mask": f"{allocated_subnet.prefixlen}"
                    })

                    # Update the network for the next available subnet
                    network = subnets_list[-1].supernet(new_prefix=required_prefix)
                else:
                    messagebox.showerror(
                        "Subnet Error", f"No available subnets for {client} with {device_count} devices."
                    )
            except ValueError as ve:
                messagebox.showerror("Input Error", f"Invalid input for {client}: {ve}")
            except Exception as e:
                messagebox.showerror("Unexpected Error", f"An error occurred while processing {client}: {e}")

        # Update Treeview with results
        update_treeview(subnets, ip_version)

        # Open explanation window with breakdown details
        open_explanation_window(breakdown_info, ip_version)

    except Exception as e:
        messagebox.showerror("General Error", f"An unexpected error occurred: {e}")

# Function to update the Treeview widget with subnet details
def update_treeview(subnets, ip_version):
    # Clear existing rows in the Treeview
    for item in tree.get_children():
        tree.delete(item)

    # Populate Treeview with the new subnet data
    for client, subnet in subnets:
        try:
            # Gather subnet details
            network_id = subnet.network_address
            broadcast_address = subnet.broadcast_address if ip_version == "IPv4" else "N/A (IPv6)"
            usable_ips = list(subnet.hosts())
            cidr = f"{subnet}"
            usable_ip_range = f"{usable_ips[0]} - {usable_ips[-1]}" if usable_ips else "None"
            usable_ip_count = len(usable_ips)

            # Add data as a new row
            tree.insert(
                "", "end", 
                values=(client, cidr, network_id, broadcast_address, usable_ip_range, usable_ip_count)
            )
        except Exception as e:
            messagebox.showerror("Subnet Error", f"An error occurred while displaying {client}: {e}")

# Function to open the explanation window with binary calculations
def open_explanation_window(breakdown_info, ip_version):
    # Create a new window
    explanation_window = tk.Toplevel(app)
    explanation_window.title("Subnetting Breakdown Step-by-Step")
    explanation_window.geometry("850x700")

    # Add a label
    ttk.Label(explanation_window, text="Comprehensive Subnetting Breakdown with Arithmetic", font=("Arial", 16)).pack(pady=10)

    # Explanation Text Area
    explanation_text = tk.Text(explanation_window, wrap=tk.WORD, height=30)
    explanation_text.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

    # Populate with a comprehensive step-by-step explanation
    for info in breakdown_info:
        client = info["client"]
        group_size = info["group_size"]
        subnet = info["subnet"]
        cidr_mask = info["cidr_mask"]

        # General Subnetting Steps
        explanation_text.insert(
            tk.END,
            f"--- Subnet Calculation for {client} ---\n"
            f"1. **Group Size (Devices)**: {group_size}\n"
            f"2. **Required IP Addresses**: {group_size + 2} (Including Network ID and Broadcast Address for IPv4).\n"
        )

        # Host Bits and New Prefix Length
        if ip_version == "IPv4":
            host_bits_needed = (group_size - 1).bit_length()
            new_prefix = 32 - host_bits_needed
            total_ips = 2 ** host_bits_needed
            subnet_mask = ".".join(str((0xffffffff << (32 - new_prefix) >> i) & 0xff) for i in [24, 16, 8, 0])

            explanation_text.insert(
                tk.END,
                f"3. **Host Bits Needed**: {host_bits_needed} (2^{host_bits_needed} = {total_ips} addresses).\n"
                f"4. **New Subnet Prefix Length**: /{new_prefix}.\n"
                f"5. **Subnet Mask**: {subnet_mask}.\n"
                f"6. **Allocated Subnet**: {subnet}.\n"
            )

            # Binary and Arithmetic
            network_binary = ".".join(format(octet, "08b") for octet in subnet.network_address.packed)
            mask_binary = ".".join(format((0xffffffff << (32 - new_prefix) >> i) & 0xff, "08b") for i in [24, 16, 8, 0])
            explanation_text.insert(
                tk.END,
                f"   Binary Representation:\n"
                f"   - Network Address (Binary): {network_binary}\n"
                f"   - Subnet Mask (Binary): {mask_binary}\n"
                f"   - Total Usable IPs: {total_ips - 2} (Excluding Network and Broadcast Addresses).\n\n"
            )

        elif ip_version == "IPv6":
            host_bits_needed = (group_size - 1).bit_length()
            new_prefix = 128 - host_bits_needed
            total_ips = 2 ** host_bits_needed
            explanation_text.insert(
                tk.END,
                f"3. **Host Bits Needed**: {host_bits_needed} (2^{host_bits_needed} = {total_ips} addresses).\n"
                f"4. **New Subnet Prefix Length**: /{new_prefix}.\n"
                f"5. **Allocated Subnet**: {subnet}.\n"
            )

            # Binary Arithmetic for IPv6
            network_binary = "".join(format(octet, "08b") for octet in subnet.network_address.packed)
            explanation_text.insert(
                tk.END,
                f"   Binary Representation:\n"
                f"   - Network Address (Binary): {network_binary}\n"
                f"   - Total Usable IPs: {total_ips} (IPv6 has no broadcast address).\n\n"
            )

    # Add general large-scale subnetting example
    explanation_text.insert(tk.END, "\n--- Large-Scale Subnetting Example ---\n\n")
    explanation_text.insert(
        tk.END,
        "For large networks (e.g., Class A or /32 networks in IPv6):\n"
        "1. Start with the original network (e.g., 10.0.0.0/8 for IPv4 or 2001:db8::/32 for IPv6).\n"
        "2. Divide the network into smaller subnets:\n"
        "   - For IPv4: Use /16 or /24 as needed to group devices logically.\n"
        "   - For IPv6: Use /48 or /64 for grouping (standard practice).\n"
        "3. Assign subnets to groups incrementally:\n"
        "   - Example IPv4: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24.\n"
        "   - Example IPv6: 2001:db8:0:0::/64, 2001:db8:0:1::/64, 2001:db8:0:2::/64.\n"
        "4. Use subnet masks to define boundaries and calculate available IPs per block.\n\n"
    )

    explanation_text.config(state=tk.DISABLED)  # Make text read-only



# GUI Application Setup
app = tk.Tk()
app.title("Dynamic Subnetting Organizer with IPv4 and IPv6")
app.geometry("800x700")

# Input Fields
frame = ttk.Frame(app, padding="10")
frame.pack(fill=tk.BOTH, expand=True)

ttk.Label(frame, text="Network Address (e.g., 192.168.0.0 or 2001:db8::):").grid(row=0, column=0, sticky=tk.W)
network_entry = ttk.Entry(frame)
network_entry.grid(row=0, column=1, sticky=tk.W)

ttk.Label(frame, text="Prefix Length (e.g., 24 for IPv4, 64 for IPv6):").grid(row=1, column=0, sticky=tk.W)
prefix_entry = ttk.Entry(frame)
prefix_entry.grid(row=1, column=1, sticky=tk.W)

# Add radio buttons to select IPv4 or IPv6
ip_version_var = tk.StringVar(value="IPv4")
ttk.Label(frame, text="IP Version:").grid(row=2, column=0, sticky=tk.W)
ttk.Radiobutton(frame, text="IPv4", variable=ip_version_var, value="IPv4").grid(row=2, column=1, sticky=tk.W)
ttk.Radiobutton(frame, text="IPv6", variable=ip_version_var, value="IPv6").grid(row=2, column=2, sticky=tk.W)

client_names = []
client_devices = []

# List of predefined clients and their device counts
default_clients = [
    ("Sales Department", 70),
    ("IT and Network Infrastructure", 17),
    ("Design and Development Team", 50),
    ("Wi-Fi Users (General)", 200),
    ("Data Center", 15),
    ("Administration and Management", 45),
    ("Guest Wi-Fi", 50),
    ("Security Department", 20)
]

# Adjust the GUI to accommodate these entries
for i, (name, devices) in enumerate(default_clients):
    ttk.Label(frame, text=f"Client {i + 1} Name:").grid(row=3 + i * 2, column=0, sticky=tk.W)
    name_entry = ttk.Entry(frame)
    name_entry.insert(0, name)  # Pre-fill the client name
    name_entry.grid(row=3 + i * 2, column=1, sticky=tk.W)
    client_names.append(name_entry)

    ttk.Label(frame, text=f"Client {i + 1} Devices:").grid(row=4 + i * 2, column=0, sticky=tk.W)
    device_entry = ttk.Entry(frame)
    device_entry.insert(0, devices)  # Pre-fill the device count
    device_entry.grid(row=4 + i * 2, column=1, sticky=tk.W)
    client_devices.append(device_entry)

# Button to calculate and update the grid
calculate_button = ttk.Button(frame, text="Calculate Subnets", command=calculate_and_update_grid)
calculate_button.grid(row=3 + len(default_clients) * 2, column=0, columnspan=2, pady=10)

# Output Grid
output_frame = ttk.Frame(app, padding="10")
output_frame.pack(fill=tk.BOTH, expand=True)

ttk.Label(output_frame, text="Subnet Grid:").pack(anchor=tk.W)

# Treeview setup
columns = ("Client Name", "Subnet", "Network ID", "Broadcast Address", "Usable IP Range", "Usable IP Count")
tree = ttk.Treeview(output_frame, columns=columns, show="headings")
tree.pack(fill=tk.BOTH, expand=True)

# Configure column headings
for col in columns:
    tree.heading(col, text=col)
    tree.column(col, anchor=tk.CENTER, width=100)

# Scrollbars for the Treeview
scroll_x = ttk.Scrollbar(output_frame, orient=tk.HORIZONTAL, command=tree.xview)
scroll_x.pack(side=tk.BOTTOM, fill=tk.X)
tree.configure(xscrollcommand=scroll_x.set)

scroll_y = ttk.Scrollbar(output_frame, orient=tk.VERTICAL, command=tree.yview)
scroll_y.pack(side=tk.RIGHT, fill=tk.Y)
tree.configure(yscrollcommand=scroll_y.set)

# Run the application
app.mainloop()
