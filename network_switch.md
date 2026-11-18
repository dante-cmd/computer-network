## Network Switch

A network switch is a piece of hardware that connects devices—like computers, printers, and servers—together on a Local Area Network (LAN). It acts as a central controller, allowing all these devices to communicate and share data with one another.

---

### How It Works

Unlike older network "hubs" that blindly broadcast data to every connected device, a switch is intelligent:

* **Targeted Delivery:** When a device sends a data packet, the switch receives it and reads the destination address. It then forwards that packet *only* to the specific port where the receiving device is connected.
* **MAC Addresses:** It does this by keeping a record of the unique Media Access Control (MAC) addresses of every device plugged into it.
* **Layer 2:** Because it relies on MAC addresses rather than IP addresses, a standard switch operates at the Data Link layer (Layer 2) of the OSI model.

By directing traffic only to its intended destination, a switch drastically reduces network congestion, prevents data collisions, and improves security.

### Switch vs. Router

These two are often confused, but they serve different purposes:

* **A Switch** connects multiple devices to create a single network.
* **A Router** connects multiple networks together. For example, a router connects your home or office LAN (managed by your switch) to the wider Internet.