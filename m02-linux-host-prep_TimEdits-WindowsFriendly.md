<!-- Slide number: 1 -->
# Certified Kubernetes Administrator (CKA):Installing Clusters with kubeadm
Preparing Linux hosts and container runtime dependencies

![](Picture2.jpg)
Tim Warner

![](PicturePlaceholder7.jpg)
Principal Author  |  IT Ops  |  Pluralsight
TechTrainerTim.com

### Notes:
Welcome to Course 2 in the eleven-course CKA skill path: Installing Clusters with kubeadm. I am Tim Warner. In Course 1 we used kind, a tool that runs Kubernetes nodes as Docker containers and hides the bootstrap process, so this course shifts to real Linux virtual machines and the actual kubeadm workflow the CKA exam tests on you. This first module covers infrastructure preparation, which is every step that must happen before kubeadm init, the cluster bootstrap command, can succeed. Skip any one of these steps and the bootstrap fails with errors that are very hard to interpret on exam day, especially if Linux is not your daily driver.

<!-- Slide number: 2 -->
# Four framing questions
Why does kubeadm require specific kernel modules and sysctl parameters?

![](ContentPlaceholder6.jpg)
What is containerd's role as the CRI-compliant container runtime?

![](ContentPlaceholder6.jpg)
How do I install and pin kubeadm, kubelet, and kubectl at a specific version?

![](ContentPlaceholder6.jpg)
How do I verify all prerequisites across multiple nodes before bootstrapping?

![](ContentPlaceholder6.jpg)

### Notes:
[FRAMER]: Four framing questions shape this module, and each one maps to a CKA exam competency and to real production admin work.

[Why does kubeadm require specific kernel modules and sysctl parameters?]: kubeadm is the cluster bootstrap command, and it depends on Linux kernel features and sysctl values (sysctl is Linux's runtime kernel-parameter interface, conceptually similar to the Windows registry) that ship disabled on stock Ubuntu, so you turn them on by hand.

[What is containerd's role as the CRI-compliant container runtime?]: containerd is the container engine that runs underneath Docker, and CRI is the gRPC contract Kubernetes uses to talk to whichever runtime is on the host, so you install and configure containerd before bootstrapping.

[How do I install and pin kubeadm, kubelet, and kubectl at a specific version?]: kubelet is the node agent that runs pods, kubectl is the cluster CLI, and all three binaries must run the same minor version on every node, so you install from the official pkgs.k8s.io repository and pin the packages so apt cannot upgrade them.

[How do I verify all prerequisites across multiple nodes before bootstrapping?]: One misconfigured node breaks the entire join workflow, so you script a verification pass that runs on every host before you touch kubeadm init.

<!-- Slide number: 3 -->

![](Picture5.jpg)
Checking in with Globomantics
“Infrastructure provisioned three bare Ubuntu VMs for your production cluster. You have root access and a deadline: control plane up by end of day. Before you touch kubeadm, these hosts need specific kernel modules, network settings, and a container runtime. Skip any step and kubeadm init fails with cryptic errors.”
Ravi P., Infrastructure Lead, Globomantics Robotics Company

### Notes:
[Checking in with Globomantics]: Infrastructure has provisioned three bare Ubuntu virtual machines for the production cluster, you have root access (root is the Linux equivalent of the local Administrator account), and the control plane has to be online by end of day.

[Why this matters]: The kind clusters from Course 1 were a great learning tool, but production needs real virtual machines with proper isolation, services managed by systemd (Linux's service controller, analogous to the Windows Service Control Manager), and kernel-level networking turned on. Every prerequisite we configure in this module maps directly to a preflight check that kubeadm runs during initialization.

<!-- Slide number: 4 -->
# Why does kubeadm require specific kernel modules and sysctl parameters?

### Notes:
Our first learning objective answers the question on the slide. Kubernetes networking depends on two kernel modules (kernel modules are loadable drivers that extend the running Linux kernel, similar in spirit to kernel-mode drivers on Windows) and three sysctl values that are disabled on a stock Ubuntu image, and the next three slides walk through each of them.

<!-- Slide number: 5 -->
# Kernel modules and sysctl: the networking foundation
overlay:
Enables overlay filesystem support required by container runtimes (containerd) for image layers
br_netfilter:
Enables iptables to inspect bridged network traffic
net.ipv4.ip_forward=1:
Enables packet forwarding between network interfaces (required for pod-to-pod communication across nodes)
net.bridge.bridge-nf-call-iptables=1; net.bridge.bridge-nf-call-ip6tables=1:
Enables bridged traffic filtering
Persistence:
Persist modules in /etc/modules-load.d/ and sysctl in /etc/sysctl.d/
Warning:
Without these, kubeadm preflight checks fail or cluster networking breaks later

### Notes:
[FRAMER]: Two kernel modules and three sysctl parameters form the networking foundation of every kubeadm cluster, and getting any of them wrong means kubeadm either fails fast or your cluster breaks later in ways that are much harder to diagnose.

[overlay]: The overlay kernel module enables the OverlayFS filesystem, which is what containerd uses to stack read-only image layers under a writable container layer (the same layered-image idea used by Windows containers under the hood). Without overlay loaded, containerd cannot start containers.

[br_netfilter]: The br_netfilter module wires Linux bridge traffic, meaning packets crossing a software Layer 2 switch, into iptables and netfilter, which is the Linux packet-filtering subsystem that acts as the host firewall and is what kube-proxy and most CNI plugins use to enforce service rules across pods.

[net.ipv4.ip_forward = 1]: This sysctl turns the host into a Layer 3 packet forwarder between network interfaces, which is required because pods on different nodes depend on the host kernel to route traffic between them.

[net.bridge.bridge-nf-call-iptables = 1]: This sysctl makes bridged Layer 2 traffic visible to iptables, which the official kubeadm documentation lists as a hard requirement for cluster networking to function.

<!-- Slide number: 6 -->
Swap must be disabled
sudo swapoff -a
All nodes must run identical kubeadm, kubelet, kubectl versions
Kernel modules + sysctl must be configured (networking foundation)
kubeadm does NOT install containerd or CNI — you must install both
kubeadm prerequisites: what must be correct before init

![](ContentPlaceholder9.jpg)

### Notes:
[FRAMER]: Before kubeadm initializes a cluster, it assumes the host is already correctly configured, and these four conditions are what kubeadm preflight verifies on every node.

[Swap must be disabled]: Swap is the Linux equivalent of the Windows pagefile, and the kubelet refuses to run with swap enabled because the scheduler relies on accurate memory accounting, so you disable swap with swapoff and remove the swap entry from /etc/fstab, the filesystem mount table, so it does not return after a reboot.

[All nodes must run identical kubeadm, kubelet, kubectl versions]: The bootstrap workflow assumes every node runs the same minor version of all three binaries, and version skew between the control plane and worker nodes causes the join to fail.

[Kernel modules + sysctl must be configured (networking foundation)]: This is the work from the previous slide, and it has to be in place before kubeadm init runs because preflight refuses to continue without it.

[kubeadm does NOT install containerd or CNI — you must install both]: kubeadm is a bootstrapper, not an installer, so you install a CRI-compliant runtime such as containerd before init and a CNI plugin (CNI stands for Container Network Interface, the pluggable component that provides pod networking) such as Calico or Flannel after init.

<!-- Slide number: 7 -->
# Configure kernel modules and sysctl for Kubernetes networking
# Load required kernel modules (runtime)
sudo modprobe overlay        # Required for container image layers (containerd)
sudo modprobe br_netfilter   # Allows iptables to see bridged traffic

# Persist modules across reboots
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Configure sysctl networking parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1              # Enable packet forwarding between interfaces
net.bridge.bridge-nf-call-iptables = 1  # Ensure bridged traffic hits iptables
# net.bridge.bridge-nf-call-ip6tables = 1  # Optional (IPv6 environments)
EOF

# Apply sysctl settings immediately
sudo sysctl --system

# Quick verification
lsmod | grep -E 'overlay|br_netfilter'   # Modules loaded
sysctl net.ipv4.ip_forward               # Should return = 1

### Notes:
[FRAMER]: This is the verbatim shell sequence that loads the two required kernel modules, persists them across reboots, and applies the three sysctl values that kubeadm preflight expects.

[modprobe overlay and modprobe br_netfilter]: modprobe is the Linux command that loads a kernel module into the running kernel, similar to loading a driver on demand on Windows, so this sequence makes both modules active for the current session without rebooting.

[/etc/modules-load.d/k8s.conf]: This file is a configuration drop-in that systemd reads at every boot to auto-load named modules, so writing overlay and br_netfilter into it makes the runtime change you just made survive a restart.

[/etc/sysctl.d/k8s.conf and sysctl --system]: The /etc/sysctl.d directory is the documented home for persistent kernel parameters, and the sysctl --system command reads every file under that directory and applies the values immediately, so the change takes effect now and on every reboot.

[Verification]: The lsmod command lists currently loaded modules, and the sysctl query confirms ip_forward returns 1, which proves the host is ready before you move to containerd.

<!-- Slide number: 8 -->
# What is containerd's role as the CRI-compliant container runtime?

### Notes:
Our second learning objective is to install and configure containerd as the CRI-compliant container runtime. The Container Runtime Interface, abbreviated CRI, is a gRPC-based contract (gRPC is a high-performance remote procedure call framework built on HTTP/2) that defines how the kubelet talks to whichever runtime is on the host, and containerd replaced Docker as the default in Kubernetes version 1.24.

<!-- Slide number: 9 -->

![](Picture3.jpg)

<!-- Slide number: 10 -->
# containerd and the Container Runtime Interface (CRI)

![](Picture4.jpg)

### Notes:
This infographic shows the relationship between the kubelet, the Container Runtime Interface socket, and the container runtime itself. The kubelet is the node agent that runs every pod, CRI is the standard contract between kubelet and runtime, and containerd is the implementation of that contract on each node. We will walk through each of those layers on the next slide.

<!-- Slide number: 11 -->
# containerd and the Container Runtime Interface (CRI)
Install containerd, generate default config with containerd config default
containerd replaced Docker as the default runtime (dockershim removed in 1.24)
CRI is the abstraction (the contract). containerd is one implementation -- CRI-O is another.
Verify with systemctl status containerd and crictl info
Ensure containerd and kubelet use the same cgroup driver (systemd on modern Linux)

### Notes:
[FRAMER]: Think of the Container Runtime Interface like a standard plug, where the kubelet is the laptop and the runtime is whatever you plug in.

[CRI is the abstraction (the contract). containerd is one implementation -- CRI-O is another.]: CRI defines the gRPC contract for pulling images and managing the container lifecycle, and containerd and CRI-O are the two production-grade implementations you are most likely to meet on a real exam virtual machine.

[containerd replaced Docker as the default runtime (dockershim removed in 1.24)]: For years the kubelet used a built-in adapter called dockershim to translate CRI calls into Docker API calls, but Kubernetes version 1.24 removed that shim and standardized on containerd, which is the lightweight engine that always ran underneath Docker anyway.

[Install containerd, generate default config with containerd config default]: The Ubuntu package leaves containerd unconfigured, so you generate a baseline config.toml (a TOML file is a human-readable configuration format) with containerd config default and edit that baseline.

[Ensure containerd and kubelet use the same cgroup driver (systemd on modern Linux)]: A cgroup, short for control group, is the Linux kernel feature that limits CPU and memory per process, conceptually similar to a Windows Job Object, and both the runtime and the kubelet must agree on which subsystem manages those cgroups.

[Verify with systemctl status containerd and crictl info]: The systemctl check confirms the service is active, and crictl info confirms the CRI socket is reachable.

<!-- Slide number: 12 -->
# Install and configure containerd for kubeadm (CRI + cgroups)
# Install containerd (Ubuntu example)
sudo apt-get update
sudo apt-get install -y containerd

# Generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Align cgroup driver -- #1 CAUSE OF kubeadm init FAILURES
# SystemdCgroup = true MUST match kubelet's driver (systemd on modern Linux)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml

# Restart containerd to apply config
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify container runtime + CRI endpoint
systemctl status containerd     # Service should be active (running)
sudo crictl info               # Confirms CRI is working and shows runtime details

### Notes:
[FRAMER]: Kubernetes does not run containers directly. The kubelet talks to a container runtime through CRI, and on a kubeadm cluster that runtime is containerd.

[apt-get install -y containerd]: apt-get is the Debian and Ubuntu package manager (comparable to winget on Windows), and this command installs the containerd binary and its systemd service but does not write a working config file.

[containerd config default | sudo tee /etc/containerd/config.toml]: sudo runs a command with elevated privileges, similar to "Run as administrator", and this pipeline emits the baseline config and writes it to the canonical path so you have something to edit.

[SystemdCgroup = true]: This single edit is the most important line in the file, because the kubeadm documentation requires that containerd's cgroup driver match the kubelet's, and a mismatch here is the number one cause of kubeadm init failures.

[systemctl restart containerd and systemctl enable containerd]: systemctl is the systemd service control command, the rough equivalent of Set-Service on Windows, where restart applies your edited config and enable makes containerd start automatically on boot.

[Verify with crictl info]: crictl is the CRI command-line client (the tool that talks directly to the runtime socket), and its info subcommand returns the same surface kubeadm init probes during preflight.

<!-- Slide number: 13 -->
# Configure crictl before you need it (exam-day lifesaver)
# Tell crictl which runtime socket to talk to
# Without this, crictl guesses -- and often guesses wrong on exam VMs
sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock

# Verify the config was written
cat /etc/crictl.yaml

# Expected output:
# runtime-endpoint: unix:///run/containerd/containerd.sock

# Now crictl runs cleanly -- no warnings, no endpoint guessing
sudo crictl info       # JSON dump of runtime details
sudo crictl ps         # Running containers (post-bootstrap)
sudo crictl images     # Cached images on this node

### Notes:
[FRAMER]: crictl is the command-line client for the CRI socket, and it is how you debug containerd directly when kubectl, the cluster-level CLI, cannot help you.

[crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock]: This command writes the runtime endpoint into /etc/crictl.yaml, a small YAML config file (YAML is the indented configuration format Kubernetes uses everywhere), so crictl knows which Unix domain socket to talk to instead of guessing.

[Why kubectl cannot help]: kubectl is the high-level Kubernetes CLI, and it talks to the API server, which only sees what the kubelet reports up, so when the kubelet itself or the runtime is broken, kubectl returns errors that point you in the wrong direction.

[The exam-day trap]: On a fresh exam virtual machine, an unconfigured crictl prints deprecation warnings and can pick the wrong socket on a host that has more than one runtime present, which wastes minutes you cannot spare.

[crictl info, crictl ps, crictl images]: Once the endpoint is set, info dumps runtime details, ps lists running containers including the Kubernetes system pods, and images shows what is cached on the node, which is the same trio you would reach for with the docker CLI.

<!-- Slide number: 14 -->
# How do I install and pin kubeadm, kubelet, and kubectl at a specific version?

### Notes:
Our third learning objective is to install the Kubernetes packages at a specific version and pin them so the operating system does not upgrade them behind your back. The CKA exam specifies an exact version in the question stem, and unplanned upgrades break running clusters in production.

<!-- Slide number: 15 -->
# Installing and pinning the Kubernetes packages
Add the official Kubernetes apt repository with signed GPG key
Install kubeadm, kubelet, and kubectl at v1.35

![](ContentPlaceholder29.jpg)

![](ContentPlaceholder35.jpg)

![](ContentPlaceholder31.jpg)
apt-mark hold kubeadm kubelet kubectl on ALL THREE NODES (CP + both workers) immediately after install

![](ContentPlaceholder37.jpg)
kubelet starts but crashes until kubeadm init configures it

![](ContentPlaceholder33.jpg)
Identical versions on every node — verify with dpkg -l kubeadm kubelet kubectl

### Notes:
[FRAMER]: Three packages, one version, pinned on every node. kubeadm is the bootstrapper, kubelet is the node agent that runs the workload, and kubectl is the administrator CLI.

[Add the official Kubernetes apt repository with signed GPG key]: An apt repository is a package source, and the GPG key (GPG is the open-source signing tool that plays the same role as Authenticode on Windows) proves the packages are authentic, so you download the key, dearmor it, and add the pkgs.k8s.io repository to apt's sources.

[apt-mark hold kubeadm kubelet kubectl on ALL THREE NODES (CP + both workers) immediately after install]: apt-mark hold tells apt to refuse upgrades for these three packages, which is the documented protection against an unattended apt-get upgrade silently breaking your cluster.

[Identical versions on every node — verify with dpkg -l kubeadm kubelet kubectl]: dpkg is the lower-level Debian package tool, and dpkg -l prints the installed versions so you can confirm the columns line up across the control plane and both workers.

[Install kubeadm, kubelet, and kubectl at v1.35]: A standard apt-get install pulls the latest version from the v1.35 channel and installs all three binaries together, which is what you want because version skew across the three is unsupported.

[kubelet starts but crashes until kubeadm init configures it]: This behavior surprises every new operator, but it is correct because the kubelet has no config yet, so systemd restarts it in a tight loop until kubeadm init writes /var/lib/kubelet/config.yaml.

<!-- Slide number: 16 -->
# How do I verify all prerequisites across multiple nodes before bootstrapping?

### Notes:
Our fourth learning objective is to verify every prerequisite on every node before running kubeadm init. A cluster is only as strong as its weakest node, and one missing kernel module on one worker derails the entire bootstrap with a preflight error that does not always tell you what is wrong.

<!-- Slide number: 17 -->
Every node needs identical kernel modules, sysctl, containerd, and packages
Run a verification script that checks every prerequisite on each node
Check swap is disabled: swapon --show should return nothing
Check containerd: systemctl status containerd and crictl info
Check packages: dpkg -l kubeadm kubelet kubectl shows pinned versions

Verifying prerequisites across all cluster nodes

### Notes:
[FRAMER]: A cluster is only as strong as its weakest node, so the goal of this verification pass is to find the one misconfigured host before kubeadm tells you about it in the worst possible way.

[Every node needs identical kernel modules, sysctl, containerd, and packages]: The control plane and both workers must be configured identically, because any deviation causes preflight failures during kubeadm init or kubeadm join.

[Run a verification script that checks every prerequisite on each node]: A short bash script (the Linux equivalent of a PowerShell .ps1) that runs the next three checks and exits on first failure turns this from a manual pass into something you run on every node in seconds.

[Check swap is disabled: swapon --show should return nothing]: swapon --show lists active swap areas, and an empty output means swap is off, which is what the kubelet requires, while any non-empty line means you still have cleanup to do in /etc/fstab.

[Check containerd: systemctl status containerd and crictl info]: The systemctl check confirms the service is in the running state inside systemd, and crictl info confirms the CRI socket is healthy and the runtime answers queries from the kubelet's perspective.

[Check packages: dpkg -l kubeadm kubelet kubectl shows pinned versions]: The dpkg output should show the same v1.35 version on all three packages across every node, and the "hi" flag in the status column proves the hold is in effect.

<!-- Slide number: 18 -->
# Prepare three Linux VMs for kubeadm from bare Ubuntu hosts

### Notes:
Time to put everything we just covered into practice. We will provision three Ubuntu virtual machines with Vagrant (Vagrant is a HashiCorp tool that scripts VM creation, similar to using PowerShell to drive Hyper-V), configure every prerequisite, and verify all three nodes are ready for kubeadm init. The demo walks through the Vagrantfile, loads and persists the kernel modules, configures sysctl, installs and aligns containerd's cgroup driver, installs and pins the Kubernetes packages, and runs the verification pass on every node.

<!-- Slide number: 19 -->

![](Picture5.jpg)
Checking out with Globomantics
“All three nodes passed the prerequisite check. Kernel modules loaded, sysctl configured, containerd running with the right cgroup driver, and packages pinned. Now I am confident these hosts will not surprise us during kubeadm init.”
Ravi P., Infrastructure Lead, Globomantics Robotics Company

### Notes:
[Checking out with Globomantics]: All three nodes passed the prerequisite check. Kernel modules are loaded, sysctl is configured, containerd is running with the right cgroup driver, and the packages are pinned at v1.35.

[Why this matters in production]: This mirrors the real-world production workflow, which is to verify every prerequisite before bootstrap, and on the exam, skipping verification means debugging cryptic preflight failures under time pressure. The verification checklist we just ran covers every preflight check kubeadm performs.

<!-- Slide number: 20 -->

![](Picture10.jpg)
# From Globomantics to you
Kernel modules are non-negotiable
overlay and br_netfilter must be loaded AND persisted before kubeadm init

![](ContentPlaceholder12.jpg)

![](ContentPlaceholder14.jpg)
Systemd Cgroup alignment prevents silent failures
Mismatched cgroup drivers between containerd and kubelet cause pods to fail without clear errors
Version pinning protects cluster stability
apt-mark hold prevents the OS package manager from upgrading Kubernetes behind your back

![](ContentPlaceholder16.jpg)
Verify every node before bootstrapping
Run a prerequisite check script on all nodes -- one misconfigured node derails the entire cluster

![](ContentPlaceholder18.jpg)

### Notes:
[FRAMER]: Four takeaways from this module that apply equally well to the CKA exam and to production cluster work.

[Kernel modules are non-negotiable / overlay and br_netfilter must be loaded AND persisted before kubeadm init]: Loading the modules at runtime is only half the job, because /etc/modules-load.d is what makes them survive a reboot, and the exam tests this from memory.

[Systemd Cgroup alignment prevents silent failures / Mismatched cgroup drivers between containerd and kubelet cause pods to fail without clear errors]: This is the hardest bug in the module to diagnose because pods start and then misbehave while the logs do not point at the cgroup mismatch directly.

[Version pinning protects cluster stability / apt-mark hold prevents the OS package manager from upgrading Kubernetes behind your back]: An unattended apt-get upgrade is exactly the kind of incident that ruins a Friday afternoon, and the hold flag is the documented protection against it, comparable to pinning a package version with winget on Windows.

[Verify every node before bootstrapping / Run a prerequisite check script on all nodes -- one misconfigured node derails the entire cluster]: Verification is not optional, because the seconds you save skipping it are dwarfed by the time you lose debugging a failed kubeadm init or kubeadm join.

[Looking ahead]: With these four habits in place, your infrastructure layer is ready, and the next module covers the actual kubeadm init and join workflow on top of the foundation you just built.